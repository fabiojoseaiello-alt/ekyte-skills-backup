# Ekyte Skills — Backup e Guia de Uso

Pacote com o **ecossistema completo de skills do fluxo Ekyte** — da criação do cliente até a task briefada, ativada e taggeada. Nenhum dado de cliente foi incluso, só os arquivos de skill (`SKILL.md` + assets/scripts/references das próprias skills).

São **13 skills** organizadas em 3 camadas:

- **Esteira ponta a ponta** (10 skills) — do nascimento do cliente até a task no Ekyte.
- **Suporte de cache/templates** (1 skill) — abastece o que a esteira consome.
- **Pós-criação / gestão em lote** (2 skills) — tags e renomeação em massa.

---

## 1) A esteira completa (do cliente até a task briefada)

```
/novo-cliente
 → /account-handoff
 → /account-pesquisa-profunda-cliente
 → /contexto
 → /notebooklm-cadastrar          (se faltar NotebookLM)
 → /ekyte-refresh
 → /ekyte-briefing-refresh
 → /ekyte-task
      → /ekyte-briefing
           → /cs-notebooklm-consulta-cliente
      → preview obrigatório
      → criar task
      → ativar task
      → aplicar responsáveis por etapa (se pedidos)
      → aplicar tags finais
```

Essa esteira agora está **documentada dentro da própria `/ekyte-task`** (seção "Fluxo ponta a ponta do cliente até a task briefada"). Regra prática: se o cliente é novo ou pouco documentado, `/ekyte-task` **não** é o primeiro passo — sem a fundação, a task vira "pedido preenchido"; com a cadeia completa, nasce com contexto, público, objetivo, referências, riscos e guardrails.

---

## 2) Mapa das 13 skills

### Fundação do cliente (5 skills)

| Skill | O que faz | Quando usar |
|---|---|---|
| `novo-cliente` | Cria `clientes/<nome>/` com `CLAUDE.md`, `AGENTS.md`, `links.md`, `.env`, `calls/`, `checkins/`, `docs/`, `campanhas/`. Pode já cadastrar NotebookLM/Drive/site. | Cliente novo entrou na operação |
| `account-handoff` | Primeira leitura vendas → operação. Consome form de kickoff + transcript da reunião comercial (+ proposta opcional) e gera KB preliminar, riscos, promessas, perguntas de kickoff, Mission Control preliminar e deck HTML. | Logo após receber o cliente de vendas |
| `account-pesquisa-profunda-cliente` | Com dados internos mínimos já na pasta, consolida tudo e entrega 4 prompts sequenciais pro Gemini Deep Research (cliente, setor, consumidor, concorrência). | **Antes** de copy, conteúdo, campanha ou LP |
| `contexto` | Lê todos os arquivos da pasta do cliente e gera/atualiza `CLAUDE.md`, `AGENTS.md` e `mission-control/`. | Quando o cliente tem muito arquivo local (calls, docs) |
| `outra-notebooklm-cadastrar` | Cadastro **em massa** do bloco `## NotebookLM` no `CLAUDE.md` de clientes existentes. Invocada como `/notebooklm-cadastrar`. | Quando você tem N clientes prontos e só agora pegou os links |

### Núcleo Ekyte (2 skills da esteira + 1 subskill de contexto)

| Skill | O que faz | Quando usar |
|---|---|---|
| `ekyte-task` | Orquestrador. Cria a task no Ekyte via MCP, monta título `[NN][SIGLA][IA] Cliente \| Demanda`, valida workspace/projeto/tipo/SLA, mostra preview, cria, **ativa** ("Gerar tarefas"), troca responsável de etapa e aplica tags finais. | "sobe uma task no ekyte de [tipo] pro [cliente]" |
| `ekyte-briefing` | Subskill da `ekyte-task` (passo 6). Monta o briefing (texto plano formatado) com template por sigla + Drive + NotebookLM + cache de público + perguntas ativas. Não cria task. | Automática pela `ekyte-task`, ou avulsa pra "pensar uma demanda" |
| `cs-notebooklm-consulta-cliente` | Cérebro de contexto. Decompõe a demanda em até 5 perguntas, consulta o NotebookLM do cliente via `notebooklm-py`, agrega numa síntese e salva artefato datado. | Chamada internamente pela `ekyte-briefing` (obrigatória em projeto novo) |

### Suporte de cache e templates (3 skills `-refresh`)

| Skill | O que faz | Quando usar |
|---|---|---|
| `ekyte-refresh` | Atualiza `cache.md` (workspaces, projetos do trimestre, tipos) e `flows.md` (fases por tipo). | Cliente/projeto novo, trimestre novo, cache > 30 dias |
| `ekyte-briefing-refresh` | Atualiza `drives.md` e `backups-crm.md`. | Drive trocou, planilha de backup CRM criada |
| `ekyte-templates-refresh` | CRUD nos templates de briefing por sigla em `briefing-templates/`. | Criar template novo (KV, MT…), ajustar pergunta de CA, refinar depois de N tasks |

### Pós-criação / gestão em lote (2 skills)

| Skill | O que faz | Quando usar |
|---|---|---|
| `gestao-ekyte-tags` | Aplica etiquetas padronizadas em lote (modo ROTINA e modo TIPO) seguindo o Playbook de Tags da Colli & Co, com merge seguro. | Taggear rotina (Sprint Growth, Semana NN) ou entregáveis em massa |
| `gestao-ekyte-rename-tasks` | Renomeia títulos de tasks em lote via MCP oficial (find-replace por lista de IDs ou filtro). | Padronizar títulos de várias tasks de uma vez |

> A `/ekyte-task` já aplica as tags finais básicas (`SPRINT GROWTH` + `SEMANA NN` + `IA` vermelha) no passo 9.7. A `gestao-ekyte-tags` cobre os casos amplos de taggeamento em lote.

---

## 3) Como as skills conversam (runtime)

```
ENTRADA: "sobe 5 criativos ads pro Euro com foco em remarketing"
                          │
                          ▼
                   ┌──────────────┐
                   │ /ekyte-task  │  ← orquestrador
                   └──────┬───────┘
        lê cache.md / sla.md / flows.md, monta título
                          │
                          ▼
                ┌────────────────────┐
                │  /ekyte-briefing   │  ← subskill (passo 6)
                └─────────┬──────────┘
        lê template da sigla + drives.md + backups-crm.md
                          │
                          ▼
            ┌──────────────────────────────────┐
            │ /cs-notebooklm-consulta-cliente   │
            └──────────────┬───────────────────┘
              notebooklm-py CLI → síntese do cliente
                          │
                          ▼
              briefing (texto plano) volta pra /ekyte-task
                          │
              PREVIEW + confirma → criar_tarefa_tool
                          │
              ativar (Gerar tarefas) → responsáveis → tags finais
                          │
                          ▼
                     ✅ task briefada e ativa
```

**Regra mental:** as skills `-refresh` só editam arquivos de configuração. Quem faz trabalho real (cria task, monta briefing) são `ekyte-task` e `ekyte-briefing`. As skills `account-*` e `contexto` constroem a base de conhecimento do cliente que abastece o briefing.

---

## 4) Arquivos de suporte — referência rápida

| Arquivo | Quem cria | Quem lê |
|---|---|---|
| `clientes/<x>/CLAUDE.md` | `/novo-cliente`, `/account-handoff`, `/contexto` | `/ekyte-briefing`, `/cs-notebooklm-consulta-cliente` |
| `clientes/_skill-ekyte/cache.md` | `/ekyte-refresh` | `/ekyte-task` |
| `clientes/_skill-ekyte/flows.md` | `/ekyte-refresh` | `/ekyte-task` (responsável por etapa) |
| `clientes/_skill-ekyte/sla.md` | manual | `/ekyte-task` |
| `clientes/_skill-ekyte/drives.md` | `/ekyte-briefing-refresh` | `/ekyte-briefing` |
| `clientes/_skill-ekyte/backups-crm.md` | `/ekyte-briefing-refresh` | `/ekyte-briefing` |
| `clientes/_skill-ekyte/briefing-templates/*.md` | `/ekyte-templates-refresh` | `/ekyte-briefing` |
| `clientes/<x>/publicos-cache.md` | `/ekyte-briefing` | `/ekyte-briefing` |
| MCP `ekyte` + `ekyte-oficial` | `~/.claude/mcp.json` | `/ekyte-task`, `/ekyte-refresh`, gestão |
| `notebooklm-py` CLI | sistema | `/cs-notebooklm-consulta-cliente` |

---

## 5) Notas importantes

- **Texto plano, não HTML.** O Quill do Ekyte não interpreta HTML enviado via API REST (`description_create_task`). A `ekyte-briefing` devolve texto plano formatado (emojis numerados, bullets `•`, URLs soltas).
- **Dois MCPs do Ekyte.** `ekyte` (n8n) **cria**; `ekyte-oficial` (api.ekyte.com) **edita** pós-criação (renomear, fase, responsável, prazo) sem JWT manual.
- **Ativação obrigatória.** A task nasce "Não planejada" e precisa de "Gerar tarefas" (REST, body `[]`, project-level) pra virar "Ativa". HTTP 500 com body vazio = "nada pra ativar", não é erro real.
- **Nunca nascer atrasada.** SLA vai em "Concluir tarefa até"; a etapa atual fica com "Executar etapa até = hoje".
- **Projeto novo força NotebookLM.** Em `projeto_novo: true`, a consulta `/cs-notebooklm-consulta-cliente` é obrigatória; faltando NotebookLM/login, o fluxo pausa.
- **NotebookLM compartilhado.** Clientes que dividem notebook (Euro + Eleva) exigem nomear a marca alvo em toda consulta.
- **Sem dados de clientes.** Este pacote não inclui `clientes/<nome>/` nem `clientes/_skill-ekyte/`. Tudo é re-gerado pelas skills `*-refresh` no destino.

---

## 6) Cheat-sheet de invocação

```
SETUP DO CLIENTE (uma vez por cliente)
  1. /novo-cliente
  2. /account-handoff
  3. /account-pesquisa-profunda-cliente
  4. /contexto
  5. /notebooklm-cadastrar         (se faltar NotebookLM)

SETUP DO EKYTE (uma vez / quando muda)
  /ekyte-refresh
  /ekyte-templates-refresh
  /ekyte-briefing-refresh

USO DIÁRIO
  /ekyte-task <pedido livre ou aba da planilha>
     → /ekyte-briefing → /cs-notebooklm-consulta-cliente

PÓS-CRIAÇÃO / LOTE
  /gestao-ekyte-tags           ← taggear rotina/entregáveis em massa
  /gestao-ekyte-rename-tasks   ← padronizar títulos em massa
```
