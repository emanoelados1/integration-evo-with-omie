# Caso de Estudo: Integração EVO (W12) → Omie ERP

## Escopo da Integração

### Divisão de Responsabilidades

| Sistema | Responsabilidade |
|---------|-----------------|
| **EVO (W12)** | Gestão do cliente/aluno, planos, matrículas, cobranças e recebimentos |
| **Omie ERP** | Backoffice: contas a pagar, faturamento, fluxo de caixa, DRE, fiscal |

### Dados a Sincronizar (EVO → Omie)

| # | Dado | Descrição |
|---|------|-----------|
| 1 | **Cadastro de Clientes** | Members do EVO → Clientes Omie (pré-requisito para vincular parcelas) |
| 2 | **Parcelas a Receber por Serviço** | Receivables com status em aberto no EVO → Contas a Receber no Omie |
| 3 | **Parcelas Recebidas por Serviço (pós-conciliação)** | Receivables com status pago/conciliado no EVO → Baixa de Contas a Receber no Omie |

> **Premissa**: O EVO é o sistema-mestre de clientes e recebimentos. O Omie recebe os dados financeiros para compor o backoffice (fluxo de caixa, DRE, conciliação contábil).

---

## 1. Validação das APIs

### ✅ EVO (ABC Fitness / W12) — API Disponível

| Item | Detalhe |
|------|---------|
| **Documentação** | Swagger UI disponível em `https://evo-integracao.w12app.com.br/swagger` |
| **Formato** | REST API (OpenAPI/Swagger 3.0) — JSON |
| **Versionamento** | v1 e v2 (alguns endpoints v3) |
| **Autenticação** | **Basic Authentication** — Username = DNS da academia, Password = Secret Key |
| **Swagger JSON** | `https://evo-integracao.w12app.com.br/swagger/v1/swagger.json` |
| **Webhooks** | Suporte a webhooks (CRUD via API) |

#### Endpoints Relevantes para este Escopo

| Módulo | Endpoint | Uso nesta Integração |
|--------|----------|---------------------|
| **Members** | `GET /api/v2/members` | Listar membros para cadastro no Omie |
| | `GET /api/v2/members/{idMember}` | Detalhe do membro (CPF, endereço, contato) |
| | `GET /api/v2/members/active-members` | Membros ativos (filtro incremental) |
| **Receivables** | `GET /api/v1/receivables` | **Principal**: parcelas a receber e recebidas, filtradas por período |
| | `GET /api/v1/receivables/debtors` | Inadimplentes (parcelas vencidas em aberto) |
| **Sales** | `GET /api/v2/sales` | Vendas (contexto do serviço vinculado à parcela) |
| | `GET /api/v2/sales/{idSale}` | Detalhe da venda — tipo de serviço, plano |
| **Membership** | `GET /api/v2/membership` | Planos/serviços disponíveis (categorização) |
| | `GET /api/v3/membermembership` | Contratos ativos/cancelados por membro |
| **Webhook** | `POST /api/v1/webhook` | **Registrar webhook** para receber eventos em tempo real |
| | `GET /api/v2/webhook` | Listar webhooks configurados |
| | `DELETE /api/v1/webhook?IdWebhook={id}` | Remover webhook específico |

#### Webhooks EVO — Modelo Event-Driven (Formato da Chamada)

Para registrar um webhook, envia-se um `POST /api/v1/webhook` com o seguinte body:

```json
{
  "idBranch": 1,
  "eventType": "<tipo_do_evento>",
  "urlCallback": "https://seu-middleware.com/webhooks/evo",
  "headers": [
    { "nome": "X-Webhook-Secret", "valor": "seu_secret_de_validacao" }
  ],
  "filters": [
    { "filterType": "<tipo_filtro>", "value": "<valor>" }
  ]
}
```

> **Nota**: O campo `eventType` é uma string livre. Os tipos de evento devem ser consultados na documentação interna do EVO ou via suporte ABC Fitness. Eventos típicos em plataformas fitness incluem mudanças em membros, vendas e recebíveis. Recomenda-se listar webhooks existentes (`GET /api/v2/webhook`) após o cadastro para confirmar a configuração.

---

### ✅ Omie ERP — API Disponível

| Item | Detalhe |
|------|---------|
| **Documentação** | Portal do Desenvolvedor: `https://developer.omie.com.br` |
| **Lista de APIs** | `https://developer.omie.com.br/service-list/` |
| **Formato** | **SOAP via HTTP POST** — aceita JSON no body |
| **Autenticação** | `app_key` (numérico) + `app_secret` (string) enviados no body de cada request |
| **Rate Limit** | Possui limites de consumo (ver documentação) |
| **Webhooks** | Suporte a webhooks para eventos em tempo real |

#### Estrutura de uma Chamada Omie

```json
POST https://app.omie.com.br/api/v1/geral/clientes/
Content-Type: application/json

{
  "call": "IncluirCliente",
  "app_key": 123456,
  "app_secret": "seu_app_secret",
  "param": [
    {
      "codigo_cliente_integracao": "EVO-12345",
      "razao_social": "João da Silva",
      "cnpj_cpf": "123.456.789-00",
      "email": "joao@email.com"
    }
  ]
}
```

#### Endpoints Relevantes para este Escopo

| Módulo | Endpoint / Call | Uso nesta Integração |
|--------|----------------|---------------------|
| **Clientes** | `POST /api/v1/geral/clientes/` | |
| | call: `UpsertCliente` | Criar ou atualizar cliente (idempotente) |
| | call: `ConsultarCliente` | Verificar se cliente já existe |
| **Contas a Receber** | `POST /api/v1/financas/contareceber/` | |
| | call: `IncluirContaReceber` | Criar parcela a receber |
| | call: `UpsertContaReceber` | Criar ou atualizar parcela (idempotente) |
| | call: `AlterarContaReceber` | Atualizar parcela existente |
| | call: `LancarRecebimento` | **Registrar baixa/pagamento** de uma parcela |
| | call: `ConsultarContaReceber` | Verificar status de uma parcela |
| **Categorias** | `POST /api/v1/geral/categorias/` | Categorias financeiras para DRE |
| **Contas Correntes** | `POST /api/v1/geral/contacorrente/` | Conta corrente para conciliação |

---

## 2. Divisão de Papéis EVO × Omie

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        DIVISÃO DE RESPONSABILIDADES                             │
├────────────────────────────────┬────────────────────────────────────────────────┤
│         EVO (W12)              │              OMIE ERP                          │
│     Sistema Operacional        │           Backoffice                           │
│                                │                                                │
│  ✔ Cadastro de clientes/alunos │  ✔ Contas a pagar                             │
│  ✔ Gestão de planos/contratos  │  ✔ Faturamento / NFS-e                        │
│  ✔ Cobranças e recebimentos    │  ✔ Fluxo de caixa                             │
│  ✔ Controle de inadimplência   │  ✔ DRE (Demonstração do Resultado)            │
│  ✔ Check-in / acessos          │  ✔ Conciliação bancária                       │
│                                │  ✔ Relatórios contábeis/fiscais               │
│   ───── DADOS FLUEM ─────►    │                                                │
│   Clientes + Parcelas          │   Recebe dados para compor visão financeira   │
└────────────────────────────────┴────────────────────────────────────────────────┘
```

---

## 3. Arquitetura e Fluxo Completo

### 3.1 Visão Geral da Arquitetura (Event-Driven via Webhooks)

```
┌──────────────────┐                                              ┌──────────────────┐
│    EVO (W12)     │                                              │    Omie ERP      │
│                  │                                              │                  │
│  Gestão Clientes │     ┌──────────────────────────────────┐     │  Backoffice      │
│  Planos/Contratos│     │    MIDDLEWARE DE INTEGRAÇÃO       │────►│  Contas a Pagar  │
│  Cobranças       │     │      (API Server / Webhook)       │     │  Faturamento     │
│  Recebimentos    │     │                                  │     │  Fluxo de Caixa  │
│  Conciliação     │     │  ┌────────────────────────────┐   │     │  DRE             │
│                  │     │  │  WEBHOOK RECEIVER (HTTP)   │   │     │  Fiscal          │
│  ┌────────────┐  │     │  │                            │   │     │                  │
│  │ Webhooks   │──╋────►│  │  POST /webhooks/evo        │   │     │                  │
│  │ (push)     │  │     │  │  • Valida assinatura       │   │     │                  │
│  └────────────┘  │     │  │  • Roteia por eventType    │   │     │                  │
│                  │     │  └────────────┬───────────────┘   │     │                  │
└──────────────────┘     │               │                   │     └──────────────────┘
                         │               ▼                   │
                         │  ┌────────────────────────────┐   │
                         │  │  PROCESSADOR DE EVENTOS    │   │
                         │  │  (Fila assíncrona)         │   │
                         │  │                            │   │
                         │  │  • Sync Cliente → Omie     │   │
                         │  │  • Sync Parcela → Omie     │   │
                         │  │  • Sync Baixa → Omie       │   │
                         │  └────────────┬───────────────┘   │
                         │               │                   │
                         │  ┌────────────┴───────────────┐   │
                         │  │   BANCO DE ESTADO          │   │
                         │  │                            │   │
                         │  │ • Mapa EVO↔Omie IDs       │   │
                         │  │ • Status parcelas          │   │
                         │  │ • Logs de auditoria        │   │
                         │  │ • Eventos recebidos        │   │
                         │  └────────────────────────────┘   │
                         │                                   │
                         │  ┌────────────────────────────┐   │
                         │  │ CRON DE CONCILIAÇÃO        │   │
                         │  │ (safety net — ex: 1x/dia)  │   │
                         │  │ Detecta divergências e     │   │
                         │  │ eventos perdidos           │   │
                         │  └────────────────────────────┘   │
                         └──────────────────────────────────┘
```

> **Abordagem Event-Driven**: O EVO dispara webhooks automaticamente quando dados são criados ou alterados. O middleware **recebe** os eventos em tempo real via HTTP POST, sem necessidade de polling periódico. Um cron de conciliação roda como safety net (ex: 1x/dia) para detectar eventos perdidos.

### 3.2 Fluxo Detalhado: EVO → Webhook → Middleware → Omie

```
═══════════════════════════════════════════════════════════════════════════════════
                    FLUXO 1: CADASTRO DE CLIENTES
         (Pré-requisito para vincular parcelas — via Webhook)
═══════════════════════════════════════════════════════════════════════════════════

  EVO                          MIDDLEWARE                           OMIE
  ───                          ──────────                           ────

  ┌─────────────────┐
  │ Webhook Event:  │
  │ Membro criado/  │─── (1) EVO dispara ─────────►┌──────────────────────┐
  │ alterado        │   webhook automaticamente    │  Webhook Receiver    │
  │                 │   com dados do membro        │  POST /webhooks/evo  │
  └─────────────────┘                              │                      │
                                                   │  (2) Valida header   │
                                                   │  X-Webhook-Secret    │
                                                   │                      │
  ┌─────────────────┐                              │  (3) Busca detalhe   │
  │ GET /api/v2/    │                              │  completo do membro  │
  │ members/        │◄── (3a) Busca dados ────────│  (CPF, endereço...)  │
  │ {idMember}      │─── completos ──────────────►│                      │
  └─────────────────┘                              │  (4) Para cada membro│
                                                   │  • Valida CPF/CNPJ   │
                                                   │  • Monta payload Omie│
                                                   │  • Define código     │
                                                   │    integração:       │
                                                   │    "EVO-{idMember}"  │
                                                   └──────────┬───────────┘
                                                              │
                                                              ▼
                                          ┌───────────────────┴──────────────────┐
                                          │                                      │
                                    NÃO EXISTE                             JÁ EXISTE
                                          │                                      │
                                          ▼                                      ▼
                              ┌─────────────────────┐              ┌─────────────────────┐
                              │ POST Omie            │             │ POST Omie            │
                              │ call: UpsertCliente  │             │ call: UpsertCliente  │
                              │                      │──────────►  │ (atualiza dados)     │
                              │ codigo_cliente_      │             └──────────┬────────────┘
                              │ integracao:          │                        │
                              │ "EVO-{idMember}"     │                        │
                              └──────────┬───────────┘                        │
                                         │                                    │
                                         ▼                                    ▼
                              ┌──────────────────────────────────────────────────┐
                              │  (5) Salva no BD de Estado:                      │
                              │  • idMember (EVO) ↔ codigo_cliente (Omie)        │
                              │  • Data/hora da sincronização                    │
                              │  • Status: OK / ERRO                             │
                              │  • ID do evento webhook recebido                 │
                              └──────────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════════════════
               FLUXO 2: PARCELAS A RECEBER POR SERVIÇO
       (Receivables em aberto do EVO → Contas a Receber Omie — via Webhook)
═══════════════════════════════════════════════════════════════════════════════════

  EVO                          MIDDLEWARE                           OMIE
  ───                          ──────────                           ────

  ┌─────────────────┐
  │ Webhook Event:  │
  │ Receivable      │─── (1) EVO dispara ─────────►┌──────────────────────┐
  │ criado/alterado │   webhook com dados da       │  Webhook Receiver    │
  │                 │   parcela a receber           │  POST /webhooks/evo  │
  └─────────────────┘                              │                      │
                                                   │  (2) Filtra apenas   │
                                                   │  parcelas com status │
                                                   │  "Em Aberto"         │
                                                   │                      │
                                                   │  (3) Para cada       │
                                                   │  parcela:            │
  ┌─────────────────┐                              │                      │
  │ GET /api/v2/    │                              │  • Busca dados da    │
  │ sales/{idSale}  │◄── (3a) Busca detalhe ──────│    venda (serviço)   │
  │                 │─── do serviço vinculado ────►│                      │
  └─────────────────┘                              │  • Identifica tipo:  │
                                                   │    Mensalidade,      │
                                                   │    Personal, Avulso  │
                                                   │                      │
                                                   │  • Mapeia categoria  │
                                                   │    financeira (DRE)  │
                                                   └──────────┬───────────┘
                                                              │
                                                              ▼
                                                   ┌──────────────────────┐
                                                   │  (4) Verifica pré-   │
                                                   │  requisito: cliente  │
                                                   │  já existe no Omie?  │
                                                   └──────────┬───────────┘
                                                              │
                                          ┌───────────────────┴──────────────┐
                                          │                                  │
                                     SIM (OK)                        NÃO (PENDENTE)
                                          │                                  │
                                          ▼                                  ▼
                              ┌─────────────────────┐          ┌─────────────────────┐
                              │  (5) Monta payload  │          │  Enfileira para     │
                              │  Contas a Receber:   │         │  retry após sync    │
                              │                      │         │  de clientes        │
                              │  POST Omie           │         └─────────────────────┘
                              │  /financas/          │
                              │  contareceber/       │
                              │                      │
                              │  call:               │
                              │  UpsertContaReceber  │
                              │                      │──────────────────────────────────►
                              │  • codigo_lancamento │         ┌─────────────────────┐
                              │    _integracao:      │         │  OMIE               │
                              │    "EVO-RCV-{id}"    │         │                     │
                              │                      │         │  Contas a Receber   │
                              │  • codigo_cliente    │         │  alimenta:          │
                              │    _integracao:      │         │  • Fluxo de Caixa   │
                              │    "EVO-{idMember}"  │         │  • DRE (por categ.) │
                              │                      │         │  • Previsão Receita │
                              │  • data_vencimento   │         └─────────────────────┘
                              │  • valor_documento   │
                              │  • codigo_categoria  │
                              │    (tipo serviço)    │
                              │  • id_conta_corrente │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────────────────────────────────┐
                              │  (6) Salva no BD de Estado:                      │
                              │  • idReceivable (EVO) ↔ codigo_lancamento (Omie) │
                              │  • Tipo de serviço / Categoria                   │
                              │  • Valor, Vencimento, Status                     │
                              └──────────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════════════════
         FLUXO 3: PARCELAS RECEBIDAS PÓS-CONCILIAÇÃO
  (Pagamentos confirmados no EVO → Baixa de Contas a Receber Omie — via Webhook)
═══════════════════════════════════════════════════════════════════════════════════

  EVO                          MIDDLEWARE                           OMIE
  ───                          ──────────                           ────

  ┌─────────────────┐
  │ Webhook Event:  │
  │ Receivable      │─── (1) EVO dispara ─────────►┌──────────────────────┐
  │ pago/recebido   │   webhook automaticamente    │  Webhook Receiver    │
  │                 │   quando pagamento é          │  POST /webhooks/evo  │
  └─────────────────┘   confirmado/conciliado      │                      │
                                                   │  (2) Filtra apenas   │
                                                   │  parcelas com status │
                                                   │  "Pago" / "Recebido" │
                                                   │                      │
                                                   │  (3) Para cada       │
                                                   │  parcela recebida:   │
                                                   │                      │
                                                   │  • Verifica se a     │
                                                   │    parcela original  │
                                                   │    já existe no Omie │
                                                   │                      │
                                                   │  • Valida valor pago │
                                                   │    vs valor original │
                                                   │                      │
                                                   │  • Identifica forma  │
                                                   │    de pagamento      │
                                                   └──────────┬───────────┘
                                                              │
                                                              ▼
                                                   ┌──────────────────────┐
                                                   │  (4) Busca no BD de  │
                                                   │  Estado o ID da      │
                                                   │  conta a receber     │
                                                   │  correspondente      │
                                                   │  no Omie             │
                                                   └──────────┬───────────┘
                                                              │
                                          ┌───────────────────┴──────────────┐
                                          │                                  │
                                 ENCONTROU                          NÃO ENCONTROU
                                 (parcela existe)                   (parcela não existe)
                                          │                                  │
                                          ▼                                  ▼
                              ┌─────────────────────┐          ┌─────────────────────┐
                              │  (5) Registra baixa │          │  Cria conta a       │
                              │  no Omie:            │         │  receber + baixa    │
                              │                      │         │  simultânea         │
                              │  POST Omie           │         │  (parcela retroativa│
                              │  /financas/          │         │  já paga)           │
                              │  contareceber/       │         └──────────┬──────────┘
                              │                      │                    │
                              │  call:               │                    │
                              │  LancarRecebimento   │                    │
                              │                      │──────────────────────────────────►
                              │  • codigo_lancamento │         ┌─────────────────────┐
                              │    _integracao:      │         │  OMIE               │
                              │    "EVO-RCV-{id}"    │         │                     │
                              │                      │         │  Baixa alimenta:    │
                              │  • data:             │         │  • Fluxo de Caixa   │
                              │    data_pagamento    │         │    (entrada real)   │
                              │                      │         │  • Conc. Bancária   │
                              │  • valor:            │         │  • DRE realizado    │
                              │    valor_pago        │         │  • Saldo em Caixa   │
                              │                      │         └─────────────────────┘
                              │  • conta_corrente    │
                              │  • observacao:       │
                              │    forma_pagamento   │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────────────────────────────────┐
                              │  (6) Atualiza BD de Estado:                      │
                              │  • Status parcela: RECEBIDA                      │
                              │  • Data do pagamento                             │
                              │  • Valor recebido (pode diferir do original)     │
                              │  • ID da baixa no Omie                           │
                              └──────────────────────────────────────────────────┘
```

### 3.3 Fluxo de Processamento (Event-Driven via Webhooks)

```
    ┌──────────────────────────────────────────────────────────────────────────┐
    │              PROCESSAMENTO EVENT-DRIVEN (Tempo Real)                     │
    │                                                                          │
    │   Webhook recebido do EVO                                                │
    │          │                                                               │
    │          ▼                                                               │
    │   ┌──────────────┐                                                       │
    │   │ Identifica   │                                                       │
    │   │ tipo evento  │                                                       │
    │   └──────┬───────┘                                                       │
    │          │                                                               │
    │   ┌──────┼──────────────────┬──────────────────────────┐                  │
    │   │      │                  │                          │                  │
    │   ▼      ▼                  ▼                          ▼                  │
    │ ┌──────────┐  ┌─────────────────┐  ┌──────────────────────────┐           │
    │ │ MEMBRO   │  │ RECEIVABLE      │  │ RECEIVABLE               │           │
    │ │ criado/  │  │ criado/alterado  │  │ pago/conciliado          │           │
    │ │ alterado │  │ (em aberto)     │  │                          │           │
    │ │          │  │                 │  │                          │           │
    │ │ → Upsert │  │ → Verifica se   │  │ → Busca parcela no BD    │           │
    │ │   Cliente│  │   cliente existe │  │ → LancarRecebimento      │           │
    │ │   Omie   │  │   no Omie       │  │   no Omie                │           │
    │ │          │  │ → UpsertConta   │  │                          │           │
    │ │          │  │   Receber Omie  │  │                          │           │
    │ └──────────┘  └────────┬────────┘  └──────────────────────────┘           │
    │                        │                                                  │
    │                   NÃO EXISTE                                              │
    │                   cliente?                                                │
    │                        │                                                  │
    │                        ▼                                                  │
    │               ┌─────────────────┐                                         │
    │               │ Enfileira para  │   O middleware garante a ordem:          │
    │               │ retry: cria     │   Se o cliente ainda não existe          │
    │               │ cliente primeiro│   no Omie, primeiro sincroniza           │
    │               │ depois reprocessa│  o cliente, depois a parcela.           │
    │               │ a parcela       │                                         │
    │               └─────────────────┘                                         │
    └──────────────────────────────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────────────────────────────┐
    │              CRON DE CONCILIAÇÃO (Safety Net — 1x/dia)                    │
    │                                                                          │
    │   Roda periodicamente para:                                              │
    │   • Detectar eventos de webhook que possam ter sido perdidos             │
    │   • Comparar totais EVO × Omie para identificar divergências            │
    │   • Reprocessar itens com status ERRO ou PENDENTE                        │
    │   • Gerar relatório de integridade da sincronização                      │
    └──────────────────────────────────────────────────────────────────────────┘
```

> **Nota**: A ordem de dependência (cliente antes da parcela) continua obrigatória, mas agora é gerenciada **por evento**: quando chega um webhook de receivable e o cliente ainda não existe no Omie, o middleware automaticamente sincroniza o cliente primeiro e depois processa a parcela.

---

## 4. Mapeamento Detalhado de Campos

### 4.1 Clientes (EVO Members → Omie Clientes)

| Campo EVO (Origem) | Campo Omie (Destino) | Observação |
|---------------------|----------------------|------------|
| `idMember` | `codigo_cliente_integracao` | Prefixar: `"EVO-{idMember}"` |
| `firstName` + `lastName` | `razao_social` | Pessoa física: nome completo |
| `firstName` | `nome_fantasia` | Nome de tratamento |
| `document` (CPF) | `cnpj_cpf` | Validar formato antes de enviar |
| `email` | `email` | — |
| `phone` / `cellPhone` | `telefone1_numero` | Priorizar celular |
| `address.street` | `endereco` | — |
| `address.number` | `endereco_numero` | — |
| `address.neighborhood` | `bairro` | — |
| `address.city` | `cidade` | — |
| `address.state` | `estado` | UF com 2 letras |
| `address.zipCode` | `cep` | Apenas números |
| — | `pessoa_fisica` | Fixo: `"S"` (alunos são PF) |
| — | `contribuinte` | Fixo: `"2"` (não contribuinte) |
| — | `tags` | Opcional: `"EVO"` para identificação |

### 4.2 Parcelas a Receber (EVO Receivables → Omie Contas a Receber)

| Campo EVO (Origem) | Campo Omie (Destino) | Observação |
|---------------------|----------------------|------------|
| `idReceivable` | `codigo_lancamento_integracao` | Prefixar: `"EVO-RCV-{idReceivable}"` |
| `idMember` | `codigo_cliente_integracao` | Prefixar: `"EVO-{idMember}"` — vínculo ao cliente |
| `dueDate` | `data_vencimento` | Formato: `"dd/mm/aaaa"` |
| `issueDate` | `data_previsao` | Data de emissão da parcela |
| `ammount` / `value` | `valor_documento` | Valor da parcela |
| `description` / serviço | `observacao` | Descrição do serviço (plano, personal, etc.) |
| Tipo de serviço (via Sale) | `codigo_categoria` | Mapear para categoria Omie (ver tabela abaixo) |
| — | `id_conta_corrente` | Conta corrente Omie configurada |
| `receivableNumber` | `numero_documento` | Número sequencial da parcela |
| Parcela X de Y | `numero_parcela` / `quantidade_parcelas` | Se aplicável |

#### Mapeamento de Categorias (Tipo de Serviço → Categoria Omie para DRE)

| Tipo de Serviço no EVO | Código Categoria Omie | Descrição (exemplo) |
|------------------------|----------------------|---------------------|
| Mensalidade / Plano | `1.01.01` | Receita - Mensalidades |
| Personal Trainer | `1.01.02` | Receita - Personal |
| Avaliação Física | `1.01.03` | Receita - Avaliações |
| Produto / Loja | `1.01.04` | Receita - Vendas Loja |
| Matrícula / Taxa | `1.01.05` | Receita - Taxas |
| Outros Serviços | `1.01.99` | Receita - Outros |

> **Nota**: Os códigos de categoria devem ser previamente cadastrados no Omie e ajustados conforme o plano de contas da academia.

### 4.3 Parcelas Recebidas — Baixa (EVO Receivables Pagos → Omie LancarRecebimento)

| Campo EVO (Origem) | Campo Omie (Destino) | Observação |
|---------------------|----------------------|------------|
| `idReceivable` | `codigo_lancamento_integracao` | `"EVO-RCV-{idReceivable}"` — localiza a parcela no Omie |
| `paymentDate` | `data` | Data efetiva do pagamento |
| `ammountPaid` | `valor` | Valor efetivamente recebido |
| `paymentMethod` | `observacao` | Cartão, PIX, boleto, dinheiro |
| — | `codigo_conta_corrente` | Conta corrente de recebimento no Omie |
| `discount` | `desconto` | Desconto concedido (se houver) |
| `interest` | `juros` | Juros cobrados (se houver) |
| `fine` | `multa` | Multa cobrada (se houver) |

---

## 5. Cenários e Regras de Negócio

### 5.1 Parcela a Receber — Cenários

| Cenário | Tratamento |
|---------|-----------|
| **Nova parcela em aberto** | Criar `Conta a Receber` no Omie via `UpsertContaReceber` |
| **Parcela já enviada e não alterada** | Ignorar (verificar via BD de Estado) |
| **Parcela cancelada no EVO** | Excluir ou estornar no Omie via `CancelarContaReceber` |
| **Parcela com valor alterado** | Atualizar no Omie via `UpsertContaReceber` |
| **Parcela de cliente inexistente** | Enfileirar; sincronizar cliente primeiro, depois reprocessar |
| **Parcela vencida** | Enviar normalmente — no Omie aparecerá como atrasada |

### 5.2 Parcela Recebida (Pós-Conciliação) — Cenários

| Cenário | Tratamento |
|---------|-----------|
| **Pagamento integral** | `LancarRecebimento` com valor total |
| **Pagamento parcial** | `LancarRecebimento` com valor parcial — parcela fica com saldo |
| **Pagamento com desconto** | Informar `desconto` no lançamento |
| **Pagamento com juros/multa** | Informar `juros` e `multa` no lançamento |
| **Estorno de pagamento** | `EstornarRecebimento` para reverter baixa |
| **Recebimento sem parcela no Omie** | Criar conta a receber + baixa em sequência |
| **Duplicidade de baixa** | Verificar no BD se já foi baixada antes de enviar |

### 5.3 Conciliação — Garantindo Integridade

```
  EVO (fonte de verdade)                    OMIE (espelho financeiro)
  ──────────────────────                    ─────────────────────────

  Parcela #101                              Conta a Receber "EVO-RCV-101"
  Status: Em Aberto       ═══ sync ═══►     Status: A vencer
  Valor: R$ 150,00                          Valor: R$ 150,00
  Venc: 10/04/2026                          Venc: 10/04/2026

       │ (aluno paga)
       ▼

  Parcela #101                              Conta a Receber "EVO-RCV-101"
  Status: Pago             ═══ sync ═══►    Status: Liquidado
  Data Pgto: 12/04/2026                    Baixa: 12/04/2026
  Valor Pago: R$ 150,00                    Valor: R$ 150,00

                                            ─────────────────────────
                                            → Entra no Fluxo de Caixa
                                            → Compõe DRE realizado
                                            → Concilia com extrato
```

---

## 6. Modelo do Banco de Estado (Middleware)

### Tabelas Principais

```sql
-- Mapeamento de clientes sincronizados
CREATE TABLE sync_clientes (
    id                SERIAL PRIMARY KEY,
    evo_id_member     BIGINT UNIQUE NOT NULL,
    omie_codigo_cli   VARCHAR(50),        -- "EVO-{idMember}"
    omie_id_cliente   BIGINT,             -- código Omie retornado
    cpf_cnpj          VARCHAR(20),
    nome              VARCHAR(200),
    ultima_sync       TIMESTAMP NOT NULL,
    status            VARCHAR(20) NOT NULL -- OK, ERRO, PENDENTE
);

-- Mapeamento de parcelas sincronizadas
CREATE TABLE sync_parcelas (
    id                      SERIAL PRIMARY KEY,
    evo_id_receivable       BIGINT UNIQUE NOT NULL,
    evo_id_member           BIGINT NOT NULL,
    evo_id_sale             BIGINT,
    omie_codigo_lanc        VARCHAR(50),     -- "EVO-RCV-{idReceivable}"
    omie_id_conta_receber   BIGINT,          -- código Omie retornado
    tipo_servico            VARCHAR(100),     -- Mensalidade, Personal, etc.
    valor_original          DECIMAL(12,2),
    data_vencimento         DATE,
    status_evo              VARCHAR(30),      -- Aberto, Pago, Cancelado
    status_omie             VARCHAR(30),      -- A_Receber, Liquidado, Cancelado
    data_pagamento          DATE,
    valor_pago              DECIMAL(12,2),
    baixa_enviada           BOOLEAN DEFAULT FALSE,
    ultima_sync             TIMESTAMP NOT NULL,
    erro_detalhe            TEXT
);

-- Log de eventos webhook recebidos (deduplicação + auditoria)
CREATE TABLE webhook_events (
    id              SERIAL PRIMARY KEY,
    received_at     TIMESTAMP DEFAULT NOW(),
    event_type      VARCHAR(100) NOT NULL,   -- Tipo do evento recebido do EVO
    event_id        VARCHAR(100) UNIQUE,     -- ID único do evento (deduplicação)
    payload         JSONB NOT NULL,          -- Payload completo do webhook
    status          VARCHAR(20) NOT NULL,    -- RECEBIDO, PROCESSANDO, PROCESSADO, ERRO
    processed_at    TIMESTAMP,
    erro_detalhe    TEXT
);

-- Log de todas as operações
CREATE TABLE sync_log (
    id              SERIAL PRIMARY KEY,
    created_at      TIMESTAMP DEFAULT NOW(),
    fluxo           VARCHAR(50),   -- CLIENTES, PARCELAS_ABERTO, PARCELAS_BAIXA
    direcao         VARCHAR(10),   -- EVO_OMIE
    entidade_id     VARCHAR(50),   -- ID da entidade processada
    acao            VARCHAR(30),   -- CRIAR, ATUALIZAR, BAIXAR, CANCELAR, IGNORAR
    status          VARCHAR(20),   -- OK, ERRO, RETRY
    webhook_event_id INTEGER REFERENCES webhook_events(id), -- Evento que originou a operação
    request_payload JSONB,
    response_payload JSONB,
    erro_mensagem   TEXT
);
```

---

## 7. Stack Tecnológica

| Camada | Tecnologia | Justificativa |
|--------|-----------|---------------|
| **Linguagem** | TypeScript (Node.js) | Tipagem estática, excelente suporte a JSON, interfaces para contratos de API |
| **Runtime** | Node.js 20 LTS | Suporte nativo a fetch, performance para I/O assíncrono |
| **Web Framework** | Fastify | Alta performance, schema validation nativa (JSON Schema/TypeBox), plugins de rate limit |
| **ORM** | Prisma | Type-safe queries, migrations automáticas, excelente DX com TypeScript |
| **Queue** | BullMQ (Redis) | Processamento assíncrono com retry, backoff, dead-letter queue, dashboard (Bull Board) |
| **Database** | PostgreSQL | Armazenar mapeamentos, estado, logs e eventos de webhook |
| **HTTP Client** | Axios | Interceptors para retry/logging, suporte a Basic Auth (EVO) e custom headers |
| **Validação** | Zod | Validação de payloads de webhook e respostas de API com inferência de tipos |
| **Scheduler** | node-cron | Cron leve para job de conciliação diária (safety net) |
| **Testes** | Vitest + Supertest | Testes unitários e de integração com suporte nativo a TypeScript |
| **Build** | tsup / tsx | Build rápido para produção, tsx para dev com hot reload |
| **Deploy** | Docker + AWS/GCP/Azure | Escalabilidade — precisa de endpoint público para webhooks |
| **Monitoramento** | Sentry + Pino (logger) | Alertas de erro + logs estruturados JSON + monitorar saúde dos webhooks |

---

## 9. Endpoints do Middleware

Endpoints HTTP que o middleware expõe para receber webhooks, operar manualmente e monitorar a integração.

### 8.1 Webhook Receiver (EVO → Middleware)

| Método | Endpoint | Descrição | Autenticação |
|--------|----------|-----------|---------------|
| `POST` | `/webhooks/evo` | **Receptor principal** de todos os eventos do EVO. Valida `X-Webhook-Secret`, persiste o evento na tabela `webhook_events` e enfileira para processamento assíncrono. Retorna `200 OK` imediatamente. | Header `X-Webhook-Secret` |

**Request Body** (enviado pelo EVO):
```json
{
  "eventType": "member.created",
  "eventId": "evt-abc-123",
  "idBranch": 1,
  "data": { ... }
}
```

**Response**: `200 OK` (sempre — processamento é assíncrono)

---

### 8.2 Sincronização Manual (Operação / Backfill)

| Método | Endpoint | Descrição | Autenticação |
|--------|----------|-----------|---------------|
| `POST` | `/api/sync/clientes` | Dispara sincronização manual de clientes do EVO → Omie. Aceita filtros opcionais. Útil para carga inicial ou backfill. | API Key (`X-API-Key`) |
| `POST` | `/api/sync/clientes/:evoIdMember` | Sincroniza um cliente específico pelo ID do EVO | API Key |
| `POST` | `/api/sync/parcelas` | Dispara sincronização manual de parcelas a receber (receivables em aberto) do EVO → Omie | API Key |
| `POST` | `/api/sync/parcelas/:evoIdReceivable` | Sincroniza uma parcela específica pelo ID do EVO | API Key |
| `POST` | `/api/sync/baixas` | Dispara sincronização manual de parcelas recebidas (baixas) do EVO → Omie | API Key |
| `POST` | `/api/sync/baixas/:evoIdReceivable` | Registra baixa de uma parcela específica | API Key |

**Query Parameters** (para endpoints em lote):

| Parâmetro | Tipo | Descrição |
|-----------|------|----------|
| `dateFrom` | `string` (ISO 8601) | Data inicial do filtro |
| `dateTo` | `string` (ISO 8601) | Data final do filtro |
| `force` | `boolean` | Reprocessar mesmo itens já sincronizados com sucesso |

**Exemplo**:
```
POST /api/sync/clientes?dateFrom=2026-04-01&dateTo=2026-04-19
X-API-Key: sua_api_key
```

---

### 8.3 Conciliação

| Método | Endpoint | Descrição | Autenticação |
|--------|----------|-----------|---------------|
| `POST` | `/api/conciliacao/executar` | Executa o job de conciliação sob demanda (mesmo processo do cron diário). Compara totais EVO × Omie, detecta divergências e reprocessa itens com erro. | API Key |
| `GET` | `/api/conciliacao/relatorio` | Retorna o último relatório de conciliação com divergências encontradas | API Key |
| `POST` | `/api/conciliacao/reprocessar` | Reprocessa todos os itens com status `ERRO` ou `PENDENTE` na fila | API Key |

**Response** (`/api/conciliacao/relatorio`):
```json
{
  "executadoEm": "2026-04-19T03:00:00Z",
  "clientes": { "evo": 350, "omie": 348, "divergencias": 2 },
  "parcelasAberto": { "evo": 120, "omie": 118, "divergencias": 2 },
  "parcelasRecebidas": { "evo": 85, "omie": 85, "divergencias": 0 },
  "itensComErro": 3,
  "itensPendentes": 1
}
```

---

### 8.4 Consulta de Estado (Dashboard / Debug)

| Método | Endpoint | Descrição | Autenticação |
|--------|----------|-----------|---------------|
| `GET` | `/api/clientes` | Lista clientes sincronizados com status (mapeamento EVO ↔ Omie) | API Key |
| `GET` | `/api/clientes/:evoIdMember` | Detalhe de um cliente sincronizado (IDs EVO e Omie, status, última sync) | API Key |
| `GET` | `/api/parcelas` | Lista parcelas sincronizadas com filtros por status, período e tipo de serviço | API Key |
| `GET` | `/api/parcelas/:evoIdReceivable` | Detalhe de uma parcela sincronizada (status EVO/Omie, valor, datas) | API Key |
| `GET` | `/api/eventos` | Lista eventos de webhook recebidos com filtros por tipo, status e período | API Key |
| `GET` | `/api/eventos/:eventId` | Detalhe de um evento webhook (payload, status de processamento, erros) | API Key |
| `GET` | `/api/logs` | Consulta logs de sincronização com filtros por fluxo, ação e status | API Key |

**Query Parameters** (comuns a endpoints de listagem):

| Parâmetro | Tipo | Descrição |
|-----------|------|----------|
| `page` | `number` | Página (default: 1) |
| `limit` | `number` | Itens por página (default: 50, max: 200) |
| `status` | `string` | Filtrar por status: `OK`, `ERRO`, `PENDENTE` |
| `dateFrom` | `string` (ISO 8601) | Data inicial |
| `dateTo` | `string` (ISO 8601) | Data final |

---

### 8.5 Gestão de Webhooks no EVO

| Método | Endpoint | Descrição | Autenticação |
|--------|----------|-----------|---------------|
| `POST` | `/api/webhooks/registrar` | Registra os webhooks necessários no EVO via `POST /api/v1/webhook`. Configura `urlCallback`, `X-Webhook-Secret` e os `eventTypes` relevantes. | API Key |
| `GET` | `/api/webhooks/status` | Consulta os webhooks ativos no EVO via `GET /api/v2/webhook` e compara com os esperados. Retorna diagnóstico de saúde. | API Key |
| `DELETE` | `/api/webhooks/:idWebhook` | Remove um webhook específico do EVO via `DELETE /api/v1/webhook` | API Key |

**Response** (`/api/webhooks/status`):
```json
{
  "webhooksEsperados": 3,
  "webhooksAtivos": 3,
  "status": "OK",
  "detalhe": [
    { "eventType": "member.created", "ativo": true, "idWebhook": 101 },
    { "eventType": "receivable.created", "ativo": true, "idWebhook": 102 },
    { "eventType": "receivable.updated", "ativo": true, "idWebhook": 103 }
  ]
}
```

---

### 8.6 Health Check e Monitoramento

| Método | Endpoint | Descrição | Autenticação |
|--------|----------|-----------|---------------|
| `GET` | `/health` | Health check básico — retorna `200 OK` se o serviço está ativo | Público |
| `GET` | `/health/ready` | Readiness check — verifica conectividade com PostgreSQL, Redis, APIs EVO e Omie | Público |
| `GET` | `/api/metrics` | Métricas operacionais: eventos processados, erros, tempo médio de processamento, tamanho da fila | API Key |

**Response** (`/health/ready`):
```json
{
  "status": "ready",
  "checks": {
    "database": "ok",
    "redis": "ok",
    "evo_api": "ok",
    "omie_api": "ok"
  }
}
```

---

### 8.7 Resumo Visual dos Endpoints

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ENDPOINTS DO MIDDLEWARE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  WEBHOOK RECEIVER (EVO → Middleware)                                    │
│  ─────────────────────────────────                                      │
│  POST /webhooks/evo                  ← Recebe eventos do EVO            │
│                                                                         │
│  SINCRONIZAÇÃO MANUAL                                                   │
│  ────────────────────                                                   │
│  POST /api/sync/clientes             ← Sync clientes (lote)            │
│  POST /api/sync/clientes/:id         ← Sync cliente específico          │
│  POST /api/sync/parcelas             ← Sync parcelas (lote)            │
│  POST /api/sync/parcelas/:id         ← Sync parcela específica          │
│  POST /api/sync/baixas               ← Sync baixas (lote)              │
│  POST /api/sync/baixas/:id           ← Sync baixa específica            │
│                                                                         │
│  CONCILIAÇÃO                                                            │
│  ───────────                                                            │
│  POST /api/conciliacao/executar      ← Roda conciliação sob demanda     │
│  GET  /api/conciliacao/relatorio     ← Último relatório                 │
│  POST /api/conciliacao/reprocessar   ← Reprocessa erros/pendentes       │
│                                                                         │
│  CONSULTA DE ESTADO                                                     │
│  ──────────────────                                                     │
│  GET  /api/clientes                  ← Lista clientes sincronizados     │
│  GET  /api/clientes/:id              ← Detalhe cliente                  │
│  GET  /api/parcelas                  ← Lista parcelas sincronizadas     │
│  GET  /api/parcelas/:id              ← Detalhe parcela                  │
│  GET  /api/eventos                   ← Lista eventos webhook            │
│  GET  /api/eventos/:id               ← Detalhe evento                   │
│  GET  /api/logs                      ← Logs de sincronização            │
│                                                                         │
│  GESTÃO DE WEBHOOKS                                                     │
│  ──────────────────                                                     │
│  POST   /api/webhooks/registrar      ← Registra webhooks no EVO        │
│  GET    /api/webhooks/status         ← Diagnóstico de webhooks ativos   │
│  DELETE /api/webhooks/:id            ← Remove webhook do EVO            │
│                                                                         │
│  HEALTH / MONITORAMENTO                                                 │
│  ──────────────────────                                                 │
│  GET  /health                        ← Health check (público)           │
│  GET  /health/ready                  ← Readiness (DB, Redis, APIs)      │
│  GET  /api/metrics                   ← Métricas operacionais            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Requisitos para Começar

### Credenciais Necessárias

| Sistema | O que precisa |
|---------|--------------|
| **EVO** | DNS da academia (ex: `minhaacademia.w12app.com.br`) + Secret Key (obtida no painel administrativo do EVO) |
| **Omie** | `app_key` (numérico) + `app_secret` (string) — obtidos em [Meus Aplicativos](https://developer.omie.com.br/my-apps/) |

### Configurações Prévias no Omie

| Configuração | Motivo |
|--------------|--------|
| **Categorias financeiras** criadas | Para classificar receitas por tipo de serviço (DRE) |
| **Conta corrente** cadastrada | Para vincular recebimentos |
| **Aplicativo de integração** criado | Para obter `app_key` + `app_secret` |

### Pré-requisitos Técnicos

1. Acesso administrativo ao EVO para obter a Secret Key
2. Conta Omie com aplicativo criado para integração
3. Categorias financeiras configuradas no Omie (mapeamento de serviços)
4. **Servidor com endpoint público (HTTPS)** para receber webhooks do EVO
5. Banco PostgreSQL para armazenar estado da sincronização
6. **Registrar webhooks no EVO** via `POST /api/v1/webhook` para os eventos relevantes (membros, receivables)
7. Definir um `X-Webhook-Secret` para validar a autenticidade dos eventos recebidos

---

## 10. Boas Práticas

| Prática | Detalhes |
|---------|---------|
| **Webhooks como Fonte Primária** | Receber eventos do EVO em tempo real — sem polling periódico. Cron apenas como safety net |
| **Validação de Webhook** | Verificar header `X-Webhook-Secret` em todo evento recebido para garantir autenticidade |
| **Resposta Rápida (HTTP 200)** | Responder HTTP 200 ao EVO imediatamente ao receber o webhook; processar de forma assíncrona na fila |
| **Idempotência** | Usar `codigo_cliente_integracao` e `codigo_lancamento_integracao` — `Upsert` evita duplicatas. Guardar ID do evento para detectar duplicatas |
| **Ordem de Dependência** | Se receber evento de parcela sem cliente no Omie, sincronizar cliente primeiro (buscar via API) e depois processar a parcela |
| **Paginação** | Omie: máximo 500 registros/página. EVO: respeitar paginação da API (usado no cron de conciliação) |
| **Rate Limiting** | Respeitar limites de ambas as APIs, implementar throttling entre chamadas ao Omie |
| **Retry + Backoff** | Exponential backoff em caso de erro 429/5xx. Fila com dead-letter para eventos que falharam múltiplas vezes |
| **Validação de CPF** | Validar CPF/CNPJ antes de enviar — Omie rejeita documentos inválidos |
| **Logging** | Registrar toda operação em `sync_log` para auditoria e troubleshooting. Guardar payload do webhook recebido |
| **Segurança** | Nunca expor `app_secret`, Secret Key ou webhook secret em código. Usar variáveis de ambiente. Endpoint HTTPS obrigatório |
| **Conciliação Periódica** | Job diário comparando totais EVO × Omie para detectar divergências e eventos perdidos |

---

## 11. Riscos e Mitigações

| Risco | Mitigação |
|-------|----------|
| Cliente não existe no Omie ao criar parcela | Ao receber webhook de parcela, verificar cliente; se não existe, sincronizar cliente primeiro via API e depois processar parcela |
| Parcela baixada duplicadamente | Verificar flag `baixa_enviada` no BD antes de enviar + deduplicar por ID do evento webhook |
| CPF/CNPJ inválido no EVO | Validação prévia + log de rejeição + alerta |
| Valor pago difere do valor original | Registrar diferença como desconto/juros/multa conforme regra |
| Downtime de uma das APIs | Fila com retry automático e exponential backoff |
| Limites de rate da API Omie | Throttling + fila com delay configurável |
| Dados divergentes entre EVO e Omie | Job de conciliação diária + relatório de divergências |
| Mudança de versão da API EVO | Abstração de endpoints, monitorar changelog |
| Parcela cancelada no EVO após envio | Detectar cancelamento via webhook e enviar `CancelarContaReceber` |
| **Webhook não entregue / evento perdido** | Cron de conciliação diário como safety net; monitorar saúde dos webhooks via `GET /api/v2/webhook` |
| **Endpoint do middleware fora do ar** | Garantir alta disponibilidade do servidor; EVO pode não reenviar eventos perdidos — conciliação diária cobre isso |
| **Webhook mal configurado / deletado** | Verificar webhooks registrados periodicamente via `GET /api/v2/webhook`; alerta se webhook esperado não existir |

---

## 12. Cronograma Estimado de Fases

| Fase | Descrição | Dependências |
|------|-----------|-------------|
| **1 — Discovery** | Mapear campos Receivables EVO, definir categorias Omie, validar endpoints, **identificar eventTypes disponíveis nos webhooks EVO** | — |
| **2 — Setup** | Configurar credenciais, ambiente, projeto, banco de estado, **endpoint HTTPS público para webhooks** | Fase 1 |
| **3 — Webhooks + Receiver** | **Registrar webhooks no EVO** (`POST /api/v1/webhook`), implementar endpoint receptor com validação de segurança | Fase 2 |
| **4 — Clientes** | Processar eventos de membros → Clientes Omie (com Upsert) | Fase 3 |
| **5 — Parcelas a Receber** | Processar eventos de receivables (aberto) → Contas a Receber Omie por serviço | Fase 4 |
| **6 — Baixas (Pós-Conciliação)** | Processar eventos de receivables (pago) → LancarRecebimento no Omie | Fase 5 |
| **7 — Conciliação** | Job diário de verificação de integridade EVO × Omie (safety net) | Fase 6 |
| **8 — Testes & QA** | Testes end-to-end, edge cases, volume, simulação de eventos webhook | Fase 7 |
| **9 — Produção** | Deploy, monitoramento, runbook operacional, verificação de webhooks ativos | Fase 8 |

---

## 13. Conclusão

A integração foca em **três fluxos essenciais** do EVO para o Omie:

1. **Clientes** — base cadastral para vincular as parcelas
2. **Parcelas a Receber** — alimenta o fluxo de caixa previsto e DRE no Omie
3. **Parcelas Recebidas** — baixa financeira que reflete a receita realizada

O EVO permanece como **sistema-mestre** de gestão de alunos e recebimentos. O Omie atua como **backoffice financeiro**, recebendo os dados para compor contas a pagar, faturamento, fluxo de caixa e DRE.

A arquitetura utiliza **webhooks do EVO como fonte primária de eventos**, tornando a integração **event-driven e em tempo real** — sem necessidade de cron jobs para polling. Um cron de conciliação diário atua como safety net para detectar eventuais eventos perdidos.

Ambas as APIs são maduras e bem documentadas. A integração é **tecnicamente viável** seguindo o padrão de middleware event-driven com banco de estado para controle de idempotência, rastreabilidade e conciliação.
