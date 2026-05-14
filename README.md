# Ekyte Skills — Backup e Guia de Uso

Pacote contendo as **5 skills do ecossistema Ekyte** + **4 skills correlatas** que sustentam o fluxo (cadastro de cliente, NotebookLM, contexto). Nenhum dado de cliente foi incluso — só os arquivos `SKILL.md`.

Este README explica:
1. O que cada skill faz
2. Como elas conversam entre si
3. Passo a passo pra ativar o conjunto num ambiente novo
4. Os arquivos de suporte que cada skill espera encontrar

---

## 1) Mapa das 9 skills

### Núcleo Ekyte (5 skills)

| Skill | O que faz | Quando usar |
|---|---|---|
| `ekyte-task` | Cria tarefas no Ekyte via MCP. Monta título `[NN][SIGLA][IA] Cliente \| Demanda`, valida workspace/projeto, mostra preview e dispara. | "sobe uma task no ekyte de [tipo] pro [cliente]" |
| `ekyte-briefing` | Subskill da `ekyte-task`. Monta o corpo do briefing (texto plano formatado) puxando template por sigla + contexto do NotebookLM. | Chamada automaticamente pela `ekyte-task`, ou avulsa pra "pensar uma demanda" |
| `ekyte-refresh` | Atualiza `clientes/_skill-ekyte/cache.md` — re-puxa workspaces, projetos do trimestre, tipos de tarefa. | Cliente novo, trimestre novo, ou cache > 30 dias |
| `ekyte-briefing-refresh` | Atualiza `drives.md` e `backups-crm.md` — os arquivos de lookup que a `ekyte-briefing` consulta. | Cliente novo entrou, Drive trocou, planilha de backup CRM criada |
| `ekyte-templates-refresh` | CRUD nos templates de briefing por sigla em `clientes/_skill-ekyte/briefing-templates/`. | Criar template novo (KV, MT, EM), ajustar pergunta de CA, refinar depois de N tasks reais |

### Skills correlatas (4 skills)

| Skill | O que faz | Por que está aqui |
|---|---|---|
| `novo-cliente` | Cria pasta `clientes/<nome>/` com estrutura padrão e CLAUDE.md inicial. Pergunta NotebookLM na hora. | É pré-requisito da `ekyte-task` — sem pasta de cliente, a `ekyte-briefing` não tem onde buscar contexto |
| `outra-notebooklm-cadastrar` | Cadastro **em massa** de links de NotebookLM em clientes que já existem (escreve o bloco `## NotebookLM` no CLAUDE.md de cada um). | Quando você tem N clientes prontos e só agora pegou os links — não precisa re-rodar `/novo-cliente` |
| `cs-notebooklm-consulta-cliente` | Decompõe uma tarefa em até 5 perguntas, dispara contra o NotebookLM do cliente via `notebooklm-py`, agrega numa síntese. | É o motor de "contexto rico" que a `ekyte-briefing` chama internamente |
| `contexto` | Lê todos os arquivos da pasta de um cliente e gera CLAUDE.md / AGENTS.md completos. | Opcional — usa quando o cliente tem MUITO arquivo local (calls, docs) e você quer um CLAUDE.md denso |

---

## 2) Como elas conversam (diagrama de fluxo)

```
┌──────────────────────────────────────────────────────────────────┐
│  ENTRADA DO USUÁRIO                                              │
│  "sobe 5 criativos ads pro Euro com foco em remarketing"         │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │  /ekyte-task   │  ← orquestrador
                    └────────┬───────┘
                             │
        ┌────────────────────┼─────────────────────┐
        ▼                    ▼                     ▼
   lê cache.md         lê sla.md             monta título
   (IDs ekyte)         (prazos)              [NN][SIGLA][IA]
                                                   │
                             ┌─────────────────────┘
                             ▼
                  ┌────────────────────┐
                  │  /ekyte-briefing   │  ← subskill (passo 6)
                  └─────────┬──────────┘
                            │
        ┌───────────────────┼────────────────────────┐
        ▼                   ▼                        ▼
   lê template          lê drives.md       chama /cs-notebooklm-
   da sigla (CA.md,    e backups-crm.md    consulta-cliente
   LP.md, etc)                                     │
                                                   ▼
                                          notebooklm-py CLI
                                          (consulta NotebookLM
                                          do cliente, retorna
                                          síntese)
                                                   │
                            ┌──────────────────────┘
                            ▼
                   briefing montado (texto plano)
                            │
                            ▼
                  volta pra /ekyte-task
                            │
                            ▼
                     PREVIEW + confirma
                            │
                            ▼
                   MCP ekyte.criar_tarefa_tool
                            │
                            ▼
                       ✅ task criada
```

### Skills de manutenção (rodam por fora)

```
/novo-cliente            ──► cria clientes/<nome>/CLAUDE.md (com NotebookLM)
/outra-notebooklm-       ──► popula bloco ## NotebookLM em N clientes de uma vez
   cadastrar
/contexto                ──► enriquece CLAUDE.md lendo arquivos locais do cliente

/ekyte-refresh           ──► atualiza cache.md (workspaces, projetos, tipos)
/ekyte-briefing-refresh  ──► atualiza drives.md e backups-crm.md
/ekyte-templates-refresh ──► CRUD em briefing-templates/*.md
```

**Regra mental:** as skills com `-refresh` no nome só editam **arquivos de configuração**. Quem faz trabalho real (cria task, monta briefing) são `ekyte-task` e `ekyte-briefing`.

---

## 3) Passo a passo — ativar num ambiente novo

### Pré-requisitos do sistema

- **Claude Code** instalado (CLI ou extensão VSCode).
- **MCP `ekyte`** registrada em `~/.claude/mcp.json` (autenticada com a conta Ekyte).
- **`notebooklm-py`** instalado e logado (`pip install notebooklm-py` + `notebooklm login`).
- Pasta de trabalho com a árvore:
  ```
  meu-projeto/
  ├── .claude/skills/        ← skills vão aqui
  ├── clientes/
  │   ├── _template/         ← template de pasta de cliente (usado pelo /novo-cliente)
  │   └── _skill-ekyte/      ← arquivos de suporte das skills ekyte
  └── bases/                 ← opcional (projetos não-cliente)
  ```

### Passo 1 — Copiar as skills

```bash
cp -r skills/* /caminho/do/seu-projeto/.claude/skills/
```

As 9 pastas (`ekyte-task`, `ekyte-briefing`, `ekyte-refresh`, `ekyte-briefing-refresh`, `ekyte-templates-refresh`, `novo-cliente`, `outra-notebooklm-cadastrar`, `cs-notebooklm-consulta-cliente`, `contexto`) ficam disponíveis como `/skill-name` no Claude Code.

### Passo 2 — Criar estrutura `clientes/_skill-ekyte/`

As skills ekyte esperam encontrar estes arquivos. Crie a pasta vazia — as próprias skills populam quando você rodar elas pela primeira vez:

```
clientes/_skill-ekyte/
├── cache.md                  ← criado por /ekyte-refresh
├── sla.md                    ← criar manual (tabela "sigla → prazo em dias")
├── drives.md                 ← criado por /ekyte-briefing-refresh
├── backups-crm.md            ← criado por /ekyte-briefing-refresh
└── briefing-templates/       ← criado por /ekyte-templates-refresh
    ├── _header-universal.md
    ├── _base-criativo.md
    ├── _5w1h.md
    ├── CA.md   (Criativo Ads)
    ├── LP.md   (Landing Page)
    ├── RV.md   (Reels Vídeo)
    ├── PC.md   (Planejamento)
    ├── AN.md   (Análise)
    └── CRM.md  (CRM)
```

### Passo 3 — Cadastrar primeiro cliente

```
/novo-cliente
```

A skill pergunta:
1. Nome do cliente → vira `clientes/nome-do-cliente/`
2. Link do NotebookLM → escreve bloco `## NotebookLM` no CLAUDE.md

Já tem N clientes prontos? Pula esse passo e usa:
```
/notebooklm-cadastrar
```
Cola a lista no formato `nome-cliente: https://notebooklm.google.com/notebook/abc123`.

### Passo 4 — Popular cache do Ekyte

```
/ekyte-refresh
```

Confirma o trimestre vigente, e a skill puxa via MCP:
- Lista de workspaces (os 8 fixos da operação)
- Projetos do trimestre pra cada workspace
- Tipos de tarefa do workflow "Padrão Colli&Co (Oficial)"

Gera o `cache.md` que a `ekyte-task` consulta toda vez.

### Passo 5 — Popular templates de briefing

```
/ekyte-templates-refresh
```

Opção `4) Listar templates atuais` mostra o que existe. Comece criando os 6 essenciais (CA, LP, RV, PC, AN, CRM) + os 3 compartilhados (`_header-universal`, `_base-criativo`, `_5w1h`).

### Passo 6 — Popular drives e backups CRM

```
/ekyte-briefing-refresh
```

Cola os links de Drive de cada cliente. Faz uma vez, depois só atualiza quando entra cliente novo.

### Passo 7 — Subir a primeira task

```
/ekyte-task sobe 1 criativo ads pro [cliente] com foco em [tema]
```

Fluxo esperado:
1. Pre-fly check (se for lote ≥ 3, ela lista pendências de uma vez)
2. Mostra preview com título, workspace, projeto, tipo, SLA, briefing
3. Você responde "pode subir" → ela dispara MCP `criar_tarefa_tool`
4. Devolve URL da task criada

---

## 4) Arquivos de suporte — referência rápida

| Arquivo | Quem cria | Quem lê |
|---|---|---|
| `clientes/_template/` | manual (criar uma vez) | `/novo-cliente` |
| `clientes/<x>/CLAUDE.md` | `/novo-cliente` | `/ekyte-briefing`, `/cs-notebooklm-consulta-cliente` |
| `clientes/_skill-ekyte/cache.md` | `/ekyte-refresh` | `/ekyte-task` |
| `clientes/_skill-ekyte/sla.md` | manual | `/ekyte-task` |
| `clientes/_skill-ekyte/drives.md` | `/ekyte-briefing-refresh` | `/ekyte-briefing` |
| `clientes/_skill-ekyte/backups-crm.md` | `/ekyte-briefing-refresh` | `/ekyte-briefing` (templates de CRM) |
| `clientes/_skill-ekyte/briefing-templates/*.md` | `/ekyte-templates-refresh` | `/ekyte-briefing` |
| MCP `ekyte` | `~/.claude/mcp.json` | `/ekyte-task`, `/ekyte-refresh` |
| `notebooklm-py` CLI | sistema | `/cs-notebooklm-consulta-cliente` |

---

## 5) Notas importantes

- **Texto plano, não HTML.** A API do Ekyte aceita o campo `description_create_task` como string. O editor Quill que renderiza no front **não interpreta HTML** quando o texto vem por API REST. Por isso `ekyte-briefing` devolve texto plano formatado com emojis numerados e bullets `•`, não tags `<p>` ou `<h2>`.

- **Tipo "Personalizada" exige prefixo `[WEB]`** no título. O tipo `Personalizada` tem `task_type_id = 1` e a `ekyte-task` impõe essa regra automaticamente.

- **Pre-fly check propõe defaults, não lista perguntas.** Em lotes ≥ 3, a `ekyte-task` propõe defaults razoáveis inline e bloqueia só em itens sem default seguro (cliente sem NotebookLM, workspace fora dos 8 fixos, quantificação ambígua).

- **NotebookLM compartilhado.** Se dois clientes dividem o mesmo notebook (caso real: Euro + Eleva), toda consulta precisa especificar a marca alvo na pergunta — senão a síntese mistura as duas.

- **Sem dados de clientes.** Este pacote intencionalmente não inclui `clientes/<nome>/` nem `clientes/_skill-ekyte/`. Toda configuração específica é re-gerada pelas skills `*-refresh` quando você rodar elas no destino.

---

## 6) Ordem recomendada de invocação (cheat-sheet)

```
SETUP (uma vez)
  1. /novo-cliente              (ou /notebooklm-cadastrar pra lote)
  2. /ekyte-refresh
  3. /ekyte-templates-refresh
  4. /ekyte-briefing-refresh

USO DIÁRIO
  /ekyte-task <pedido livre ou aba da planilha>

MANUTENÇÃO (quando precisar)
  /ekyte-refresh              ← cliente/projeto novo, trimestre virou
  /ekyte-briefing-refresh     ← Drive ou backup CRM mudou
  /ekyte-templates-refresh    ← refinar template depois de N tasks reais
  /contexto                   ← enriquecer CLAUDE.md de um cliente
```
