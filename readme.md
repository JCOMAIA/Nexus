# Nexus AI — Apresentação

> **O funcionário que leu tudo da sua empresa.**
> Plataforma integrada de GraphRAG, agentes especializados de negócio,
> orquestração multi-modelo e automação — com rastreabilidade por operação,
> verificação matemática anti-alucinação e adaptação de interface sem retreino.

---

## Índice

- [Em 60 segundos](#em-60-segundos)
- [Arquitetura](#arquitetura)
- [Pilar 1 — Conhecimento](#pilar-1--conhecimento-rag--ingestão--memória)
- [Pilar 2 — Agentes](#pilar-2--agentes-de-negócio)
- [Pilar 3 — Tarefas](#pilar-3--tarefas-persistência--hitl)
- [Pilar 4 — Conectores](#pilar-4--conectores)
- [Vertical — Financeiro](#vertical--financeiro)
- [Vertical — Contabilidade brasileira](#vertical--contabilidade-brasileira-reforma-cbsibs)
- [Vertical — BI e Dashboards](#vertical--bi-e-dashboards)
- [Camada transversal — Harness](#camada-transversal--harness)
- [Camada transversal — Triggers](#camada-transversal--triggers-automação)
- [Camada transversal — Compartilhamento](#camada-transversal--compartilhamento)
- [Multi-provider](#multi-provider)
- [Stack tecnológica](#stack-tecnológica)
- [Princípios de design (o moat)](#princípios-de-design-o-moat)

---

## Em 60 segundos

A Nexus é a **camada de inteligência acima dos sistemas da empresa**. Ela:

- **Ingere** documentos, planilhas, contratos, bancos SQL, APIs, repositórios.
- **Indexa** com GraphRAG híbrido (Qdrant + Neo4j + Postgres + reranker).
- **Responde** perguntas em pt-BR **citando a fonte** — sem alucinar.
- **Pilota agentes especializados** de Workspace (CRM/Financeiro/Operações/Contabilidade), BI, BPM.
- **Verifica matematicamente** o que o LLM produz quando o cálculo importa
  (conciliação, agregações, desvios) — a LLM **nunca** vê o dump cru.
- **Roda em qualquer modelo** (Anthropic, OpenAI, OpenRouter, Ollama, custom).
- **Adapta a interface** entre modelo e ambiente via Harness (paper LIFE-HARNESS).
- **Bilha por operação** com créditos pré-pagos e margem configurável.

> Não substitui o ERP, planilha ou banco. Lê deles e produz inteligência.

---

## Arquitetura

```
┌───────────────────────── Frontend (React + Vite) ────────────────────────────┐
│  Dashboard · Query · Business (Overview · BI · BPM · CRM · Financeiro ·       │
│  Contabilidade · Ops · ML) · Tarefas · Ingestão · Conectores · Memória ·      │
│  Triggers · Dreamer · Observability · Modelos · Billing · API & Share         │
└─────────────────────────────────┬─────────────────────────────────────────────┘
                                  │ REST /api/v1  (+ JWT auth)
┌─────────────────────────────────┴─────────────────────────────────────────────┐
│                        Backend (FastAPI · async)                               │
│                                                                                │
│  Model Router  ── resolve task_type → provider+modelo (DB > .env > default)    │
│      │            tool-use loop provider-agnostic (OpenAI format lingua franca)│
│      │            checkpoint a cada iteração (AgentRun) · HITL · resume        │
│      ├── Anthropic · OpenAI · OpenRouter · Ollama · Custom OpenAI-compat       │
│                                                                                │
│  Harness       ── 4 camadas (contract · skill · action · trajectory)           │
│                   intervém no loop sem retreinar · auto-evolution offline       │
│                                                                                │
│  Ingestão      ── Loaders (PDF · CSV · Web · SQL · API · MCP · A2A · conector) │
│      └── Chunker → Entity Extract → Embedder → Graph + Vector                  │
│                                                                                │
│  Retrieval     ── Híbrido: Vector (Qdrant) + Grafo (Neo4j) + Fulltext → RRF    │
│      └── Rerank LLM + Síntese com citação + anti-alucinação                    │
│                                                                                │
│  Memória       ── MemCollab distillation · Dreamer · Memory Artifacts          │
│                                                                                │
│  Agentes       ── BI · BPM · Workspaces (5 domains) · Oráculo · Auto Agentes   │
│                                                                                │
│  Verticais BR  ── Variance Analyser · CBS/IBS Pricing · NF-e Credit Leak ·     │
│                   Contract Audit · SPED Check · Pareceres IFRS/CPC             │
│                                                                                │
│  Triggers      ── Pull → Transform → Destino (agendado/manual)                 │
│                                                                                │
│  Billing       ── Créditos pré-pagos · PagSeguro · margem por operação         │
└──────┬──────────┬──────────┬──────────┬───────────────────────────────────────┘
       │          │          │          │
   PostgreSQL  Neo4j     Qdrant    Redis/Celery
  (transacional, (grafo de   (vetores)  (jobs async,
   modelos,    conhecimento)            triggers)
   AgentRuns)
```

---

## Pilar 1 — Conhecimento (RAG + Ingestão + Memória)

### Ingestão

Loaders disponíveis (`nexus/ingest/loaders/`):

| Tipo | Cobertura |
|---|---|
| **PDF** | Texto + OCR (via processadores Docling/Tesseract) |
| **CSV / XLSX** | Detecção heurística de colunas, parsing pt-BR (R$, dd/mm/yyyy) |
| **Web** | URL fetch, extração de conteúdo, sitemap |
| **SQL** | Introspecção de schema + query custom |
| **API REST** | Bearer token, paginação (page/cursor/link), até N páginas |
| **MCP** | Servidor Model Context Protocol (stdio/SSE) |
| **A2A** | Agentes A2A JSON-RPC |
| **Conector unificado** | Endpoint único que substitui MCP/A2A/API com config no Conector |

Pipeline: **Chunker → Entity Extract (LLM) → Embedder → Graph (Neo4j) + Vector (Qdrant)**.
Cada documento gera resumo `.md` assíncrono para busca semântica rica.

### Retrieval

**Híbrido em paralelo:**
- **Vector** (Qdrant) — busca por similaridade
- **Grafo** (Neo4j) — fulltext em `chunk_text` index
- **Fusão** via Reciprocal Rank Fusion (RRF)
- **Rerank LLM** opcional (top-N reordenados)
- **Síntese** com citações inline + anti-alucinação (se constraint falha, devolve "não tenho informação suficiente")

### Memória

- **MemCollab distillation** — distila trajectories em fatos reutilizáveis
- **Dreamer** — consolidação semanal (compact + insights + heuristics + summary)
- **Memory Artifacts** — pacotes de conhecimento exportáveis
- **HeuristicDB** — bancos de heurísticas reutilizáveis (auto-gerados ou manuais), reusados pelos agentes
- **Loop de feedback:** uso dos agentes → trajetórias → Dreamer destila → heurísticas → próximos agentes melhores

---

## Pilar 2 — Agentes de negócio

Todos os agentes usam **o mesmo loop tool-use no router** (`run_tooluse`),
provider-agnostic, com persistência (AgentRun), HITL (request_user_input) e
opcionalmente Harness.

### Workspace (multi-domínio)

Personas com tools especializadas (`nexus/business/workspace/`):

| Domain | Persona | Tools especializadas |
|---|---|---|
| `crm` | Analista de CRM | `generate_ad_creative` + tools comuns |
| `finance` | Controller Financeiro | `query_finance_db` + tools comuns |
| `ops` | Gerente de Operações | tools comuns |
| `contabilidade` | Contador / Tributarista BR | `simulate_pricing`, `tax_schedule`, `credit_leak_summary`, `contract_audit_summary`, `sped_check_summary` |

**Tools comuns:** `search_rag`, `search_universal`, `query_sql`, `get_heuristics`,
`save_heuristic`, `render_text`, `render_table`, `render_chart`, mais
capabilities opcionais (search providers, integração WhatsApp/Telegram, etc).

### BI Agent

Chat especializado em análise de dados. Mesmo loop, com:
- `query_sql` autorizado por conector
- `search_rag` sobre fontes selecionadas (documentos, folders, scope temporal)
- Renderers de bloco: text, table, chart (bar/line/area/pie), tool_call
- **Subaba "Dashboards"** dentro do BI — ver vertical [Dashboards](#vertical--bi-e-dashboards)

### BPM Studio

Agente para modelagem e otimização de processos:
- Tools: `render_bpmn`, `render_text`, `render_table`, `render_roi`, `recommend_tools`
- Análise de gargalos via traces históricas
- Persistência: `BPMProcess` + `BPMMessage`

### Oráculo

Agente especial que **cruza insights de TODOS os módulos** (BI, BPM, memória,
contas, métricas) em uma única resposta. Roda no Opus por default para análise
mais profunda.

### Auto Agentes (Hermes / OpenClaw)

Conectores para agentes externos com seu próprio loop de raciocínio. Atuam
via MCP/A2A através de uma camada de adapter.

---

## Pilar 3 — Tarefas (persistência + HITL)

Toda corrida de agente vira uma linha em **`AgentRun`** (migration 048):

- **Checkpoint por iteração** — se o worker Celery morre, o estado fica salvo.
- **Status:** `running` · `paused` · `done` · `failed` · `cancelled`.
- **Resume** via `POST /agent_runs/{id}/resume` — retoma do último checkpoint
  preservando provider+modelo originais e acumulando custo/tokens/duração.
- **HITL:** tool padrão `request_user_input` pausa o run pedindo resposta;
  user responde via UI e o agente continua de onde parou.

### Aba Tarefas (frontend)

```
┌─ Sidebar ──────────┬─ Lista (filtrável) ────┬─ Detalhe ───────────────────┐
│ Dashboard          │ [Todos·Aguardando·...] │ Header: status, métricas    │
│ Query              │ ┌────────────────────┐ │                              │
│ Business           │ │ ▸ Run #1 paused 🟡 │ │ Painel HITL (se paused):    │
│ Tarefas (badge: 2) │ │   "Confirma R$..?" │ │   pause_state.prompt        │
│ Ingestão           │ │ ▸ Run #2 failed    │ │   [textarea] Enviar resposta│
│ ...                │ │ ▸ Run #3 done      │ │                              │
│                    │ └────────────────────┘ │ Métricas: model, tokens, $   │
│                    │                        │ Conversa: msgs + tool_calls  │
│                    │                        │ Harness: intervenções       │
└────────────────────┴────────────────────────┴──────────────────────────────┘
```

- **Sidebar badge âmbar** mostra count de runs paused (auto-refresh 10s).
- **Auto-refresh 3s** quando o run selecionado está `running`.
- **Painel Harness** lista o que cada camada fez naquela corrida.

---

## Pilar 4 — Conectores

Catálogo amplo em `nexus/connectors/catalog.py`:

**Bancos de dados**
- PostgreSQL · MySQL/MariaDB · SQL Server · SQLite · MongoDB · Elasticsearch · Redis

**Storage**
- AWS S3 (com R2/MinIO via endpoint custom) · GCS · Azure Blob · SFTP

**API**
- REST genérico (Bearer/API key) · GraphQL · Webhook (inbound) · Google Search (Serper/SerpAPI/CSE)

**SaaS**
- Slack · Notion · Google Drive · Jira · GitHub · Confluence

**Mensageria/Social**
- WhatsApp Cloud API · Telegram · Apache Kafka · RabbitMQ

**Protocolos**
- **MCP** (Model Context Protocol — stdio + SSE)
- **A2A** (Agent-to-Agent JSON-RPC)

Cada conector tem `kind`, `subtype`, schema de config, schema de credenciais
(sempre `password` no frontend), e `capabilities` (`ingest` · `query` ·
`stream` · `tool_call`). Tudo dinâmico — o frontend renderiza formulários a
partir do catálogo.

---

## Vertical — Financeiro

### Importação (`nexus/finance/importer.py`)

- Aceita **CSV** (sniff de delimiter) e **XLSX**.
- **Detecção heurística de colunas** com sinônimos pt-BR + en (fornecedor,
  cliente, vencimento, valor, doc, categoria, …).
- Parsing tolerante de valores BR (R$, vírgula decimal, ponto milhar) e datas
  em múltiplos formatos.
- Confiança ≥50% nos campos críticos → grava direto; abaixo disso pede
  confirmação do usuário.
- **LLM nunca vê o dump cru** — só headers se a heurística falhar.

### Aggregates densos (`finance/aggregates.py`)

Sumários compactos (~2-3 KB JSON mesmo com 100k+ linhas):
- Total, ticket médio, vencido count/sum
- Top fornecedores/clientes
- Top categorias
- Evolução mensal (12 meses)
- Status distribution

Esses agregados alimentam:
- Painéis de KPI no Dashboard
- Heurísticas auto-geradas (1 call LLM cheap → 5-12 heurísticas)
- Resumos `.md` ingeridos para RAG semântico

### Conciliação bancária (`finance/reconciliation.py`)

1. Extrai texto bruto de extratos (PDF/CSV).
2. **1 chamada LLM** estrutura tudo em JSON (transações + categorias).
3. **`verify_integrity()` em Python**: Σ créditos − Σ débitos vs saldo
   inicial/final. Se diferença > R$0,02 → `UNBALANCED`, humano revisa.

**Por isso o LLM nunca diz "o saldo bate" — quem diz é a aritmética.**

### Matcher determinístico

Casa `BankTransaction` ↔ `FinanceEntry` por valor (tolerância configurável) e
data (janela em dias) — sem LLM, score normalizado em [0..1].

### Subaba Desvios (Variance Analyser, migration 043)

- **Baseline histórico automático**: média móvel 3m, mesmo mês ano anterior,
  ou blend.
- **Cálculo determinístico** por categoria/contraparte/mês com limiar configurável.
- **1 LLM call batched** comenta apenas os desvios materiais, alimentada com
  agregados + top lançamentos cruzados (não com linhas individuais).
- **Trigger autônomo** `variance_scan` — agendável via Celery beat.
- Resultado: `VarianceReport` + `VarianceItem` persistidos, visível em
  Business → Financeiro → Desvios.

---

## Vertical — Contabilidade brasileira (Reforma CBS/IBS)

Agente especializado em pt-BR para a transição EC 132/2023 + LC 214/2025.

### Simulador de preços CBS/IBS (`finance/tax/`)

- **Cronograma estrutural fixo** 2026→2033 (lei).
- **Alíquotas de referência configuráveis** — defaults marcados como
  estimativa (CBS 8,8% / IBS 17,7% = ~26,5% total) porque ainda não estão
  fixadas em lei complementar.
- `simulate_pricing` projeta **preço sustentável ano a ano** preservando margem
  líquida sob carga híbrida.
- Subaba **Simulador** com gráfico + tabela detalhada.

### Auditor fatura×contrato (migration 045)

- Lê **Documents já ingeridos** (contratos em PDF/texto).
- **1 LLM call** extrai termos estruturados (tabela de preços, vigência,
  índice de reajuste, multas).
- **Auditoria determinística** confronta NF-e e Contas a Pagar do fornecedor
  contra os termos: preço acima, fora de vigência, item fora do contrato.
- Subaba **Contratos** com lista + detalhe + auditoria automática.

### Vazamento de créditos (migration 044)

- Parser **NF-e 4.0** (XML namespace-agnóstico) com suporte ao grupo
  **IBS/CBS** da NT 2025.
- Modelo `FiscalDocument` armazena tributos extraídos.
- Engine de creditável por regime (Real credita PIS/COFINS; ICMS por CST;
  IPI; IBS/CBS amplo).
- Reconciliação determinística NF-e ↔ Contas a Pagar — sinaliza notas sem
  lançamento conciliado como **crédito em risco**.
- Subaba **Créditos** com upload XMLs + análise + lista de flagged.

### Checagem SPED (migration 046)

- Parser pipe-delimitado universal (vale para EFD-Contribuições, EFD ICMS/IPI,
  ECD).
- Validações estruturais: 0000/9999 presentes, contagens do bloco 9 (9900)
  conferem com real, totais do 9990, etc.
- Auto-detecta tipo do SPED.
- Subaba **SPED**.

### Pareceres IFRS/CPC (migration 047)

- RAG sobre normas IFRS/CPC ingeridas.
- Gera pareceres fundamentados com lançamentos D/C sugeridos.
- Backend pronto; frontend em backlog.

---

## Vertical — BI e Dashboards

### Agente BI (chat de análise)

- Subaba **Análise** dentro de BI — chat com SQL + RAG + charts.
- Mesmo loop dos workspaces, com tools `query_sql`, `search_rag`, renderers.
- Sources configuráveis (folder/document IDs, connectors, scope temporal).
- Histórico com export para PDF.

### Dashboards (migration 049)

- **Páginas customizadas** com painéis arrastáveis.
- Subaba **Dashboards** dentro de BI.

**Tipos de painel:**

| Kind | Renderiza |
|---|---|
| `kpi` | Número grande com format (currency/percent/number), prefix/suffix, decimals |
| `table` | Tabela densa com max_rows configurável |
| `chart` | Bar/Line/Area/Pie via Recharts |
| `markdown` | Texto/markdown livre |

**Fontes de dados:**

| Source | Pulls |
|---|---|
| `sql` | Connector SQL + query custom |
| `kpi` | KPIs do sistema (saldo, margem, contagens) por path dotado |
| `finance` | Agregados financeiros (`ap_summary`, `ar_summary`, `variance`, `credits`) |
| `bi_block` | Snapshot de um bloco de BI session ("promova esse chart") |
| `static` | Markdown/texto estático |

**Recursos:**

- Editor com **grid 12-col** + **drag-and-drop nativo** + botões ↑/↓.
- Modal "Adicionar painel" com 4-kinds × 5-sources × display configs.
- **Refresh paralelo** de todos os painéis em uma chamada (`asyncio.gather`).
- Cache `last_value` + timestamp; visual de stale/erro inline.
- **Optimistic locking** via `version` (qualquer mudança incrementa, conflito
  retorna 409).
- **Compartilhamento** via ShareLink (`resource_type='dashboard'`) com link
  público read-only OU edit_token. View pública em `/s/{token}`.
- **Colaboradores** via SessionCollaborator (convite por email viewer/editor).

---

## Camada transversal — Harness

> **Detalhe completo em [HARNESS.md](HARNESS.md).**

Adaptação modular da **interface modelo↔ambiente**, inspirado em LIFE-HARNESS
(Univ. Pequim arXiv:2605.22166). Reduz falhas que vêm de mismatch de interface
(~90% das falhas em ambientes determinísticos, segundo o paper) **sem
retreinar nada**.

```
┌─ 4 camadas que intervêm em pontos diferentes do loop tool-use ─┐
│                                                                  │
│  L1 Contract  · Amenda system_prompt (regras estáveis)           │
│  L2 Skill     · Procedimentos via BM25 keywords                  │
│  L3 Action    · Block/canonicalize/regex antes de executar tool  │
│  L4 Trajectory· Loop/errors/budget detection após cada iteração  │
│                                                                  │
└─ Princípio: zero-diff quando harness off (toggle preserva tudo)─┘
```

- **Optimistic locking** por `version`.
- **Default por user+scope** (workspace/bi/bpm/any).
- **Audit log:** cada intervenção vira linha em `HarnessIntervention`,
  visível na aba Tarefas.
- **Auto-evolution:** `POST /harness/{id}/suggest` lê `AgentRun`s falhados,
  LLM propõe regras estruturadas, HITL aceita/rejeita.
- **Compartilhável** como produto (em desenvolvimento — exportar JSON funciona
  hoje).

---

## Camada transversal — Triggers (automação)

`nexus/triggers/`

Modelo: **pull → transform → destino**.

**Fontes (pull):**
- Connector (REST/GraphQL/MCP/A2A) com config de tool/target

**Transform opcional:**
- 1 LLM call sobre o conteúdo bruto (TaskType.TRIGGER_TRANSFORM)

**Destinos:**
- `document` (ingere no RAG)
- `memory_artifact`
- `heuristic_db` (parseia JSON ou linhas como heurísticas)
- `agent_message` (posta como mensagem em workspace/BI session — pilota agente)
- `finance_rebuild` (refaz heurísticas auto do financeiro)
- `variance_scan` (gera relatório de desvios automaticamente)

**Agendamento:** interval, cron, manual. Celery beat tick a cada 60s.

**Guard-rails duros:**
- `MAX_CHAIN_DEPTH=5` — encadeamento limitado
- `daily_budget_usd` — para o trigger se acumular custo > limite no dia
- `dedupe` — pula execução se conteúdo idêntico ao último sucesso
- `allow_loop` — feedback loop só com opt-in explícito

Toda corrida vira `TriggerRun` com cost/items/log/dedupe_key/chain_depth.

---

## Camada transversal — Compartilhamento

`nexus/sharing/` + `nexus/api/routes/integration.py`

### ShareLink (capability URL)

- Token público (24 bytes) — somente leitura.
- **edit_token** opcional separado — quem entra por ele recebe `can_write=true`.
- `resource_type` registrado: `document`, `doc_summary`, `memory_artifact`,
  `workspace`, `bi_session`, `bpm_process`, `chat_session`, `dashboard`.
- Expiração + revogação manual.
- View pública sem auth em `/s/{token}` (frontend `SharedView.tsx` despacha
  por resource_type).

### SessionCollaborator (convite por email)

- Convida emails como `viewer` ou `editor`.
- ACL respeitada por todas as rotas: `owner > editor > viewer > none`.
- `resolve_role()` e `require_role()` consultam owner direto e
  SessionCollaborator.

### ACL ([acl.py](nexus/sharing/acl.py))

Hierarquia com pesos. `_is_owner` e `load_resource` despacham por
`resource_type` para a tabela certa (BusinessWorkspace / BISession /
BPMProcess / ChatSession / Dashboard).

---

## Multi-provider

Toda chamada LLM passa pelo **Model Router** (`nexus/models/router.py`).

- **Configuração por TaskType** (e.g., `INGEST_ENTITY_EXTRACT`,
  `FINANCE_RECONCILE`, `VARIANCE`, `HARNESS_EVOLVE`).
- Resolução: **DB > .env global > DEFAULT_ROUTING**.
- Providers built-in: **Anthropic**, **OpenAI**, **OpenRouter**, **Ollama**.
- Custom providers OpenAI-compatible (vLLM, LMStudio, custom endpoints) via
  tabela `CustomProvider`.

### Tool-use provider-agnostic

A abstração `Message`/`LLMResponse` em `providers/base.py` modela
`tool_calls`/`tool_call_id` em formato **OpenAI/OpenRouter** (lingua franca).
O provider Anthropic traduz para/de blocos `tool_use`/`tool_result` nativos.

**`router.run_tooluse()`** é o loop unificado. Os 3 agentes business
(workspace/BI/BPM) viraram thin wrappers — funcionam com qualquer modelo que
suporte tool calling. Guard interno: se o modelo resolvido não suporta tools
nativos, faz fallback para um Anthropic default + warning.

### Observabilidade

`OperationLog` registra cada chamada LLM: provider, model, tokens, custo,
duração, success, system_prompt (gated por capture config), messages_json,
response_text. Para tool-use, cada iteração loga uma linha — `record_tooluse_usage`.

### Billing

`debit_operation(charged_usd = raw_cost * (1 + profit_margin))` debita
créditos do `User.credit_balance_usd` e cria `CreditTransaction` por
operação. Saldo baixo é flag (não bloqueia) — visível na sidebar.

---

## Stack tecnológica

| Camada | Tecnologia |
|---|---|
| Backend | FastAPI · SQLAlchemy (async) · Alembic · Pydantic v2 · Celery + Redis |
| Frontend | React · Vite · TypeScript · TailwindCSS · TanStack Query · Recharts · Lucide |
| Storage | PostgreSQL (transacional) · Neo4j (grafo) · Qdrant (vetor) · Redis (queue/cache) |
| Modelos | Anthropic SDK · OpenAI SDK · Ollama HTTP · HTTPX para custom |
| Auth | JWT (access + refresh) · Bcrypt hash |
| Async | asyncio puro + Celery para jobs/triggers |
| Observabilidade | structlog · OperationLog DB-side · TokenMeter UI-side |

### Migrations atuais

|  # | O que adiciona |
|---|---|
| 042 | ML módulo (drift, algorithms) |
| 043 | Variance Analyser |
| 044 | FiscalDocument (NF-e) |
| 045 | Contracts |
| 046 | SpedFile |
| 047 | Pareceres IFRS/CPC |
| 048 | AgentRun (persistência + HITL) |
| 049 | Dashboards + DashboardPanel |
| 050 | Harness + HarnessIntervention |
| 051 | HarnessSuggestion |

---

## Princípios de design (o moat)

Estes 5 princípios sustentam a diferenciação técnica da Nexus:

### 1. Deterministic core, LLM at the edges

**Todo número sai de código determinístico (Python/SQL).** O LLM só entra
para texto, narrativa, justificativa — nunca para somar, calcular margem,
ou validar consistência matemática.

Implementação real:
- `reconciliation.verify_integrity()` em Python puro decide se um extrato
  bate. LLM não vota.
- `aggregates.aggregate_for_kind()` produz KPIs densos antes de chamar LLM.
  Mesmo com 100k linhas, LLM vê 2-3 KB de JSON, não rows.
- `tax/pricing.simulate()` matematicamente projeta preço ano a ano —
  LLM **apenas explica** a tabela depois.
- `variance.compute_variance()` deterministic; só `run_variance_scan()` usa
  LLM para comentar os materiais (1 call batched).

### 2. Provider-agnostic com lingua franca aberta

OpenAI tool-calling como formato interno, Anthropic traduz na fronteira.
Funciona com qualquer modelo (Sonnet, GPT-4o, qualquer OpenRouter, vLLM
self-hosted). Multi-provider real, não promessa.

### 3. Adapta interface, não modelo

Harness (LIFE-HARNESS) intervém em 4 pontos do loop — contract, skill, action,
trajectory — corrigindo falhas de interface (~90% segundo o paper) sem
retreinar nada. **Model-agnostic + reusável + compartilhável.**

### 4. Loop de feedback de conhecimento

```
Uso → Trajetórias → Dreamer destila → Heurísticas → Próximos agentes melhores
         ↓
       AgentRuns falhadas → Harness Suggest → HITL aceita → Harness amadurece
```

Cada uso da plataforma melhora a próxima execução. Heurísticas e harness
viram **conhecimento operacional acumulado, exportável e vendável**.

### 5. Tudo auditável

- Cada chamada LLM → `OperationLog` (DB).
- Cada corrida de agente → `AgentRun` (DB) com checkpoint por iteração.
- Cada intervenção de harness → `HarnessIntervention` (DB).
- Cada operação de billing → `CreditTransaction` (DB).
- Cada trigger → `TriggerRun` (DB).

Você consegue responder a um cliente exatamente: *"o agente fez X porque
puxou Y de Z na iteração 3 às 14:23, custou $0,0042, e o harness aplicou
a regra W que bloqueou a tool inválida no caminho."*

---

## Documentos relacionados

- **[README.md](README.md)** — quick start, como rodar, configuração
- **[HARNESS.md](HARNESS.md)** — guia completo da camada de harness
- `guia-conector-hermes-openclaw.md` — integração com agentes externos

---

## Próximas direções

Roadmap visível hoje (gaps conhecidos):

- Frontend de **Pareceres IFRS/CPC** (backend pronto desde a migration 047)
- **Harness atrelado a recursos específicos** (workspace X usa harness A,
  workspace Y usa harness B) — hoje é por default global
- **ShareLink com `resource_type='harness'`** para distribuir harnesses como
  pacotes vendáveis
- **Ollama tool-calling** ativo (hoje silencia tools para modelos locais)
- **Billing no caminho via Trigger** — `_save_agent_message` ainda não debita
  créditos (só as rotas HTTP debitam)
- **Resume de BI Session com sources sensíveis** — reaproveita config
  persistida (limitação de produção)

Cada um desses é uma janela curta de trabalho que se encaixa nos padrões
arquiteturais já estabelecidos.
