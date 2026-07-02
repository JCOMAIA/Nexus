# Harness — Guia de uso

> **Adapta a interface entre agente e ambiente sem retreinar modelo.**
> Inspirado em [LIFE-HARNESS (arXiv:2605.22166)](https://arxiv.org/abs/2605.22166).

O Harness é uma camada opt-in que intervém em 4 pontos do loop tool-use dos
agentes (Workspace, BI, BPM, Contabilidade) — corrigindo falhas de interface
antes, durante e depois da execução, **sem mudar pesos de modelo ou
retreinar nada**.

---

## Índice

- [Conceito em 30 segundos](#conceito-em-30-segundos)
- [As 4 camadas](#as-4-camadas)
- [Princípio firme: zero-diff quando off](#princípio-firme-zero-diff-quando-off)
- [Acessando o harness](#acessando-o-harness)
- [Criando seu primeiro harness](#criando-seu-primeiro-harness)
- [Configurando cada camada](#configurando-cada-camada)
- [Fluxo de uso típico](#fluxo-de-uso-típico)
- [Auto-evolution (Sugestões)](#auto-evolution-sugestões)
- [Auditoria de intervenções](#auditoria-de-intervenções)
- [Tabela de decisão: qual camada usar?](#tabela-de-decisão-qual-camada-usar)
- [Anti-padrões](#anti-padrões)
- [Referência da API](#referência-da-api)
- [Modelo de dados](#modelo-de-dados)

---

## Conceito em 30 segundos

Pesquisa do paper LIFE-HARNESS mediu que **~90% das falhas** de agentes LLM em
ambientes determinísticos não vêm da falta de raciocínio do modelo — vêm de
**mismatches na interface**:

| Categoria | % das falhas |
|---|---|
| Action realization (formato errado, args faltando) | 33.6% |
| Environment contract (viola protocolo do ambiente) | 33.3% |
| Trajectory degeneration (loops, stagnation) | 23.2% |
| General reasoning (raciocínio errado de fato) | 9.9% |

Conclusão: você ganha mais ajustando a **interface** (regras, validações,
recovery) do que retreinando o modelo. **E essa interface é compartilhável
entre modelos** — funciona com Anthropic, OpenAI, OpenRouter, qualquer um.

---

## As 4 camadas

Cada camada vive em um campo JSON do harness e intervém num momento
específico do loop:

```
[antes do loop]              [task-conditioning]
  L1 Contract ────────────►  L2 Skill
  amenda o system_prompt     anexa procedimentos via BM25

[por tool call]              [após cada iteração]
  L3 Action Realization ──►  L4 Trajectory Regulation
  block / canonicalize       loop / errors / budget
```

### L1 — Contract amendments
Textos curtos anexados ao **system_prompt** antes da primeira chamada LLM.
Ordenados por `priority` (0-9). Use para **regras estáveis do ambiente** que
o modelo deveria conhecer mas não conhece.

### L2 — Procedural skills
Procedimentos reutilizáveis (`steps`) indexados por **trigger_keywords**.
Quando a mensagem do usuário contém palavras-chave relevantes (busca BM25-lite),
os top-3 skills entram no system_prompt. Use para **procedimentos
recorrentes** que o modelo deveria seguir mas frequentemente atalha.

### L3 — Action realization
Validação determinística **antes** de cada `tool_call` ser executada. 5
tipos de regras:

| `rule_kind` | O que faz |
|---|---|
| `blocklist_tool` | Bloqueia a tool inteira |
| `require_param` | Exige que um param esteja presente |
| `forbid_param_value` | Bloqueia valores específicos num param |
| `regex_param` | Valida o param contra um regex Python |
| `canonicalize_param` | Normaliza o valor (ex: trim, lowercase) |

Severities: `block` (não executa), `canonicalize` (muta args), `warn` (loga).

### L4 — Trajectory regulation
Monitora o histórico **depois** de cada iteração. 3 padrões:

| `pattern_kind` | Detecta |
|---|---|
| `repeated_tool_call` | Mesma tool N vezes seguidas |
| `consecutive_errors` | N tool_results consecutivos contendo erro |
| `budget_near_exhaustion` | Iteração ≥ N% do max_iterations |

Severities: `warn` (injeta mensagem corretiva pro próximo turn) ou
`halt` (aborta com `finish_reason="halted"`).

---

## Princípio firme: zero-diff quando off

Você pode desligar o harness a qualquer momento. **Quando desligado, o
comportamento do agente é idêntico ao de antes do harness existir.**

Três níveis de toggle, todos respeitados:

1. **Sem harness criado / sem default ativo** → loop atual sem mudança.
2. **`harness.enabled = false`** → ignorado completamente, nada injetado.
3. **Camadas vazias** (ex: `action_rules: []`) → não executa nada da camada.

Você pode criar um harness só com L1, ou só com L3 e L4, ou misturar como
quiser. **Cada camada é independente.**

---

## Acessando o harness

**Pela UI:** sidebar → **Harness** (ícone Shield). Lista de cards, botão "Novo
harness" no topo.

**Pela API:** `GET /api/v1/harness` (autenticada como qualquer rota).

---

## Criando seu primeiro harness

Pela UI:

1. **Sidebar → Harness → Novo harness.**
2. Preenche nome (ex: "Workspace BR formal"), descrição (opcional) e
   **scope** (`any` / `workspace` / `bi` / `bpm`).
3. Clica **Criar** → vai pro editor com as 5 abas.
4. Adiciona itens nas camadas (próxima seção tem exemplos).
5. **Salva** (botão no header).
6. Para que comece a aplicar **em todos os runs** do user para o scope, marca
   como **default** (estrela no card da lista).

Pela API:

```bash
curl -X POST /api/v1/harness \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name": "Workspace BR formal", "scope": "workspace"}'
```

---

## Configurando cada camada

Todos os exemplos abaixo são **copy-paste prontos** para o body do
`PATCH /api/v1/harness/{id}`. Lembre-se do `expected_version` (incrementa a
cada save).

### Camada 1 — Contract amendments

Use quando o modelo **não sabe uma regra estável** do seu ambiente.

```json
{
  "expected_version": 1,
  "contract_amendments": [
    {
      "text": " SPED:valores monetários sempre em centavos (multiplicar por 100 para reais).",
      "priority": 9,
      "source": "manual"
    },
    {
      "text": "Banco postgres produção: queries em `customers`, `orders`, `invoices` SEMPRE incluem WHERE deleted_at IS NULL.",
      "priority": 8,
      "source": "manual"
    },
    {
      "text": "Quando o usuário pedir 'relatório fiscal', cite a base normativa e marque o que é fato vs. estimativa.",
      "priority": 7,
      "source": "manual"
    }
  ]
}
```

**Efeito:** as 3 frases entram no system_prompt no início da corrida, ordenadas
por priority decrescente. O LLM passa a respeitar consistentemente sem você
precisar repetir em cada chat.

### Camada 2 — Procedural skills

Use quando há um **procedimento de N passos** que o modelo deveria seguir
mas frequentemente atalha.

```json
{
  "expected_version": 2,
  "procedural_skills": [
    {
      "trigger_keywords": ["preço", "precificar", "CBS", "IBS", "margem"],
      "steps": "Para precificação na transição CBS/IBS:\n1) Pergunte custo unitário e preço atual se não tiver.\n2) Rode simulate_pricing OBRIGATORIAMENTE antes de responder.\n3) Apresente a tabela ano a ano.\n4) Nunca dê números 'de cabeça' sobre essa transição.",
      "category": "tributário",
      "source": "manual"
    },
    {
      "trigger_keywords": ["crédito tributário", "vazamento", "perdido", "fornecedor"],
      "steps": "Sempre rode credit_leak_summary antes de afirmar valores de crédito em risco. Apresente top-5 fornecedores ordenados por leak_sum.",
      "category": "credit",
      "source": "manual"
    }
  ]
}
```

**Efeito:** Quando a pergunta do usuário tem qualquer palavra-chave de uma
skill, o procedimento é anexado ao system_prompt. Score BM25-lite combina
trigger_keywords (peso 2) com termos dos próprios steps (peso 1). Top-3
relevantes entram.

### Camada 3 — Action rules

Use quando args inválidos chegam até a tool, ou quando uma tool deve ser
**proibida** em certo contexto.

```json
{
  "expected_version": 3,
  "action_rules": [
    {
      "tool_name": "query_sql",
      "rule_kind": "regex_param",
      "config": { "param": "query", "pattern": "^(?i)\\s*(SELECT|WITH|EXPLAIN)\\b" },
      "severity": "block",
      "message": "Apenas SELECT/WITH/EXPLAIN. Para mutações, peça aprovação humana via request_user_input."
    },
    {
      "tool_name": "lookup_cnpj",
      "rule_kind": "regex_param",
      "config": { "param": "cnpj", "pattern": "^\\d{14}$" },
      "severity": "block",
      "message": "CNPJ deve ter 14 dígitos numéricos. Remova pontuação."
    },
    {
      "tool_name": "simulate_pricing",
      "rule_kind": "require_param",
      "config": { "param": "unit_cost" },
      "severity": "block",
      "message": "simulate_pricing requer unit_cost. Pergunte ao usuário se não tiver."
    },
    {
      "tool_name": null,
      "rule_kind": "canonicalize_param",
      "config": { "param": "regime", "value": "real" },
      "severity": "canonicalize",
      "message": "Regime normalizado para minúsculo."
    },
    {
      "tool_name": "send_email",
      "rule_kind": "forbid_param_value",
      "config": { "param": "to", "values": ["all@company.com", "everyone@company.com"] },
      "severity": "block",
      "message": "Envio em massa precisa de aprovação manual."
    }
  ]
}
```

Notas:

- **`tool_name: null`** = a regra aplica a **qualquer tool**. Útil para
  canonicalize universal (ex: normalizar `regime` em todas as tools que
  aceitam o param).
- **Pattern Python regex** — escape barras invertidas com `\\` no JSON.
- **`block` injeta tool_result corretivo** sem executar; o LLM lê o erro e
  adapta no próximo turn.
- **`canonicalize` muta args silenciosamente** — só normaliza se o param já
  existe (não cria param do nada).

### Camada 4 — Trajectory rules

Use quando o agente entra em loop, queima budget repetindo o mesmo erro,
ou continua quando deveria parar.

```json
{
  "expected_version": 4,
  "trajectory_rules": [
    {
      "pattern_kind": "repeated_tool_call",
      "config": { "tool": "search_rag", "max_repeats": 3 },
      "severity": "warn",
      "intervention": "Você chamou search_rag 3× sem novos resultados. Pare de buscar e responda com o que já tem OU peça especificamente o que falta via request_user_input."
    },
    {
      "pattern_kind": "consecutive_errors",
      "config": { "max": 4 },
      "severity": "halt",
      "intervention": "4 erros de tool consecutivos. Abortando para não queimar mais tokens."
    },
    {
      "pattern_kind": "budget_near_exhaustion",
      "config": { "threshold_pct": 0.75 }
    }
  ]
}
```

- **`warn`** injeta a `intervention` (ou fallback automático) como `Message(role="user", content="[harness:trajectory] …")` na próxima iteração — o LLM lê como feedback do ambiente.
- **`halt`** finaliza a corrida com `finish_reason="halted"` (vai pra `failed`
  na aba Tarefas, com retry disponível).

---

## Fluxo de uso típico

```
1. Cria harness vazio com scope correto. Marca como default (estrela).

2. Usa os agentes normalmente por uma semana.
   Alguns runs falham → aparecem em Tarefas → Falhados.

3. Vai em Harness → seu harness → aba Sugestões.
   Configura "últimas 30 runs" → clica Sugerir.
   LLM analisa as falhas e propõe regras concretas.

4. Revisa as sugestões uma por uma.
   Aceita as que fazem sentido (vira layer com source='evolved').
   Rejeita as duvidosas com decision_note.

5. Próximas corridas já aplicam as regras aceitas.
   Vê o painel "Harness" na aba Tarefas mostrando o que o harness fez.

6. Repete semanalmente → harness amadurece organicamente.
```

---

## Auto-evolution (Sugestões)

O endpoint `POST /harness/{id}/suggest` ativa o ciclo do paper: lê
`AgentRun`s problemáticos do user (status `failed` OU `finish_reason` em
`max_iterations|halted|error`), filtrados pelo `scope` do harness, e pede
ao LLM (Sonnet por padrão) para propor interventions estruturadas.

**Como funciona internamente:**

1. Coleta os N runs mais recentes do scope.
2. Adiciona as `HarnessIntervention` já registradas para que o LLM **não
   duplique** regras existentes.
3. Serializa cada run com truncamento agressivo (últimas 10 msgs, 400 chars
   cada).
4. Envia para o LLM com o schema esperado.
5. **Valida estruturalmente** cada sugestão (`_validate_suggestion`) antes
   de persistir — rejeita layer/rule_kind/pattern_kind inventados ou texto
   vazio.
6. Persiste as válidas como `HarnessSuggestion` em status `pending`.

**Aceitar uma sugestão:**

```bash
curl -X POST /api/v1/harness/suggestions/{sid}/accept \
  -d '{"expected_version": 7, "decision_note": "óbvio, 3 runs caíram nisso"}'
```

Isso anexa o `payload` da sugestão na camada correta do harness e
incrementa o `version` atomicamente. Se outra aba tiver editado entre o
seu GET e o POST, retorna 409 com a versão atual.

**Rejeitar** é apenas marcar `status='rejected'` (não afeta o harness):

```bash
curl -X POST /api/v1/harness/suggestions/{sid}/reject \
  -d '{"decision_note": "muito específico — vou ver mais um caso primeiro"}'
```

---

## Auditoria de intervenções

Cada vez que uma camada do harness **age**, uma linha é gravada em
`HarnessIntervention` com:

- `layer` (contract / skill / action / trajectory)
- `kind` (rule_kind ou pattern_kind ou 'amendment' / 'skill_match')
- `iteration` (-1 = pre-loop, 0..N = no loop)
- `action` (block / warn / canonicalize / inject / halt)
- `payload` (snapshot do contexto)
- `message` (humano-legível)
- `agent_run_id` (rastreabilidade)

**Onde ver:**

- **Aba Tarefas:** abre um AgentRun → painel **"Harness"** entre métricas
  e conversa, listando o que o harness fez naquele run.
- **API:** `GET /harness/run/{agent_run_id}/interventions` traz todas as
  intervenções de qualquer harness durante aquele run.
- **API:** `GET /harness/{id}/interventions?agent_run_id=…` filtra por
  harness específico.

---

## Tabela de decisão: qual camada usar?

| Situação | Camada |
|---|---|
| "O modelo *não sabe* uma regra estável do meu domínio" | **Contract** |
| "Há um procedimento de N passos que ele deveria seguir consistentemente" | **Skill** |
| "Args malformados chegam à tool (formato, faltando, fora do enum)" | **Action: regex/require_param/forbid_param_value** |
| "Args válidos mas precisam normalizar (case, trim, sinonímia)" | **Action: canonicalize_param** |
| "Esta tool não deve ser chamada nesse contexto" | **Action: blocklist_tool** |
| "Agente está em loop chamando a mesma tool" | **Trajectory: repeated_tool_call** |
| "Erros sucessivos sem progresso queimando tokens" | **Trajectory: consecutive_errors (halt)** |
| "Agente continua mesmo quando deveria parar" | **Trajectory: budget_near_exhaustion** |

---

## Anti-padrões

❌ **Não** use harness para esconder lógica complexa de negócio. Se você
tem um `if/else` profundo, escreva uma tool/capability — não 50 action_rules.

❌ **Não** sobrescreva o `persona` do agente via harness. O persona é o
**papel estável**; o harness adiciona regras pontuais. Se você precisa
mudar o papel inteiro, edite o persona.

❌ **Não** force o modelo a sempre responder X via harness. Harness não
amordaça raciocínio; se você quer respostas determinísticas, **chame a
tool determinística direto** (via UI ou via skill que orienta).

❌ **Não** use harness para controle de acesso. Permissões de connector,
de documento, de tool — isso é coisa do connector/auth/capability layer,
não de harness.

❌ **Não** crie regras de canonicalize que **inventam params**. A regra
`canonicalize_param` só normaliza se o param já existe (deliberadamente —
para não injetar campos que a tool não espera).

---

## Referência da API

Base: `/api/v1/harness`

### CRUD

| Método | Endpoint | Descrição |
|---|---|---|
| `GET` | `/harness?scope=…&enabled=…` | Lista com contagens por camada |
| `POST` | `/harness` | Cria harness vazio |
| `GET` | `/harness/{id}` | Detalhe completo |
| `PATCH` | `/harness/{id}` | Edita (body com `expected_version`) |
| `DELETE` | `/harness/{id}` | Apaga (cascade nas suggestions/interventions) |

### Toggle e default

| Método | Endpoint | Body |
|---|---|---|
| `POST` | `/harness/{id}/toggle` | `{"enabled": bool, "expected_version": int}` |
| `POST` | `/harness/{id}/set-default` | `{"is_default": bool, "expected_version": int}` |

`set-default` é atômico: ao marcar este como default, desmarca outros do
mesmo scope na mesma transação.

### Interventions (audit log)

| Método | Endpoint | Descrição |
|---|---|---|
| `GET` | `/harness/{id}/interventions?agent_run_id=…&limit=…` | Filtra por harness |
| `GET` | `/harness/run/{agent_run_id}/interventions` | Filtra por run |

### Evolution (sugestões)

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/harness/{id}/suggest` | Body `{since?, until?, max_runs=20}`. Roda LLM e cria suggestions pending. |
| `GET` | `/harness/{id}/suggestions?status=…&batch_id=…` | Lista sugestões |
| `POST` | `/harness/suggestions/{sid}/accept` | Body `{expected_version, decision_note?}`. Anexa payload no harness. |
| `POST` | `/harness/suggestions/{sid}/reject` | Body `{decision_note?}` |

---

## Modelo de dados

```
Harness                         HarnessIntervention (audit)
├── id, user_id                 ├── id, user_id
├── name, description           ├── harness_id (FK)
├── scope (any|workspace|...)   ├── agent_run_id (FK opcional)
├── is_default, enabled         ├── layer, kind, iteration
├── version (optimistic lock)   ├── action (block|warn|...)
├── contract_amendments []      ├── payload (snapshot)
├── procedural_skills []        ├── message
├── action_rules []             └── created_at
├── trajectory_rules []
├── use_count, last_used_at     HarnessSuggestion (HITL evolution)
└── created_at, updated_at      ├── id, user_id
                                ├── harness_id (FK)
                                ├── batch_id (agrupa lote)
                                ├── layer, payload
                                ├── rationale
                                ├── source_run_ids (auditoria)
                                ├── status (pending|accepted|rejected)
                                ├── decided_at, decision_note
                                └── cost_usd
```

Migrations: **050_harness** e **051_harness_suggestions**.

---

## Onde o harness se encaixa na arquitetura

```
┌─────────────────── nexus/business/workspace/agent.py ──────────┐
│ run_workspace_turn(harness_id=optional)                         │
│   ↓                                                              │
│   harness = await resolve_harness_for_run(...)                  │
│   ↓                                                              │
│   await router.run_tooluse(harness=harness, agent_run_id=...)   │
└─────────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────── nexus/models/router.py ──────────────────────┐
│ run_tooluse:                                                     │
│   • L1+L2 antes do loop (anexa ao system_prompt)                │
│   • L3 antes de cada on_tool_call (block/canonicalize)          │
│   • L4 após cada iteração (warn/halt)                           │
│   • Cada intervenção → log_interventions (HarnessIntervention)  │
└─────────────────────────────────────────────────────────────────┘
```

Lógica pura das 4 camadas em [`nexus/harness/runtime.py`](nexus/harness/runtime.py).
Resolução e persistência em [`nexus/harness/service.py`](nexus/harness/service.py).
Evolution offline em [`nexus/harness/evolve.py`](nexus/harness/evolve.py).

---

## FAQ

**P: Se eu desligar o harness no meio de uma corrida, o que acontece?**
R: A corrida atual termina com as regras vigentes no início. Próxima corrida
respeita o novo estado. O `enabled` é avaliado quando o `resolve_harness_for_run`
é chamado (uma vez por turn).

**P: O que acontece se duas pessoas editarem o mesmo harness ao mesmo tempo?**
R: A segunda recebe 409 Conflict com a versão atual. Recarrega e edita
novamente.

**P: Posso ter múltiplos harnesses ativos por user?**
R: Hoje, **um default por user+scope**. Para anexar harness a recursos
específicos (workspace X usa harness A, workspace Y usa harness B), seria
preciso adicionar FK `harness_id` nos modelos de recurso — gap conhecido,
fácil de adicionar quando aparecer necessidade.

**P: Harness funciona com qualquer provider de LLM?**
R: Sim, é completamente model-agnostic — vive no router, acima da abstração
de provider. Funciona com Anthropic, OpenAI, OpenRouter, custom OpenAI-compat.

**P: Posso compartilhar um harness com outros usuários?**
R: O modelo `ShareLink` ainda não tem `resource_type='harness'` registrado.
Por enquanto, exporte/importe o JSON dos 4 campos manualmente via `PATCH`.
Suporte ao ShareLink é uma adição pequena se virar prioridade.

**P: O harness afeta latência das corridas?**
R: L1 e L2 rodam **uma vez** antes do loop (custo desprezível). L3 roda em
cada tool call (operações puras, sub-millissegundo). L4 roda no fim de cada
iteração (idem). Custo de persistência: ~1 INSERT em `HarnessIntervention`
por intervenção real. Em prática, latência adicional é negligível comparada
à própria chamada LLM.
