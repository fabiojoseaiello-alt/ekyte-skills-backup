---
name: ekyte-task
description: Cria tarefas no Ekyte automaticamente via MCP, com briefing estruturado seguindo o padrão Colli&Co (Oficial). Use quando o Fabio pedir "criar/subir/abrir tarefa(s) no ekyte" — em texto livre ("quero 5 criativos ads pro Voupensar com foco em remarketing") OU apontando aba da planilha de demandas. Sempre mostra preview antes de disparar e bloqueia criação fora dos 8 workspaces conhecidos sem confirmação explícita.
user-invocable: true
---

# /ekyte-task — Criar tarefas no Ekyte com briefing estruturado

Sobe tarefas no Ekyte via MCP `ekyte`, com briefing oficial puxado do próprio ekyte e título no formato `[NN][SIGLA][IA] Cliente | Demanda`. Substitui o trabalho manual de 15 min por task por preview + 1 confirmação.

## Quando usar

Use quando o Fabio:
- Disser "cria/sobe/abre uma task no ekyte de [tipo] pro [cliente]"
- Pedir lote de tasks ("sobe 5 criativos pro Euro")
- Falar "tá na aba [nome] da planilha de demandas, sobe tudo"
- Mencionar `/ekyte-task` ou `ekyte-task`

**Não use** quando o pedido é só consultar dados do ekyte (listar projetos, listar workspaces) — a MCP `ekyte` cobre direto sem precisar dessa skill.

## Pré-requisitos

- MCP `ekyte` configurada em `~/.claude/mcp.json` (já está)
- `clientes/_skill-ekyte/cache.md` — IDs de workspaces, projetos e tipos
- `clientes/_skill-ekyte/sla.md` — tabela de prazos por tipo

## Fluxo

### 0) Pre-fly check (obrigatório em lotes ≥ 3 tasks)

**Antes** de carregar cache, ler input ou montar preview, fazer um varredura de pendências e reportar TUDO numa mensagem só. Em lotes a maior perda é round-trip — uma pergunta de cada vez transforma 5 min de trabalho em 30 min de chat.

**Princípio chave (lição 2026-05-05):** Fabio responde "pode seguir" pra listas de perguntas abertas — ele delega defaults. Em vez de fazer 5 perguntas, **propor defaults razoáveis inline** e pedir só "confirma ou ajusta?". Bloquear de verdade só pra coisas SEM default seguro.

Checar e propor:
1. **NotebookLM cadastrado** pra cada cliente do lote: `ls clientes/<x>/CLAUDE.md` + grep `## NotebookLM`. **SEM default** — se faltar, listar e perguntar: "Outmat e Associação não têm NotebookLM cadastrado — quer cadastrar agora ou seguir só com descrição?"
2. **Quantificação implícita**: se algum item menciona "X por produto", "todos os produtos", "fotos da categoria Y" ou similar — **SEM default seguro** — perguntar explícito: "Essas linhas de fotos viram 1 task com lote de N peças, ou quer que cada linha vire `[N×SKUs]` no título?"
3. **Workspaces fora dos 8 fixos**: **SEM default** — bloquear até confirmação explícita.
4. **Tipos com sigla ambígua** (CA, CJ, GP, MT, AUX, CRM, PMM, RI, SM, EV): **PROPOR default** baseado no contexto da demanda ("vou usar `34435` Configuração de CRM — match óbvio pra Chatwoot. Confirma ou ajusta?"). Não listar as 12 variantes pedindo escolha.
5. **SLA dos tipos sem entrada** na tabela: **PROPOR default** ("AN sem SLA tabelado — vou usar 3 dias úteis. Ok?"). Não perguntar aberto.
6. **Quantidade [NN] quando o lote indica claramente 1-task-1-entrega**: **PROPOR `[01]` direto** sem perguntar.

Formato preferido do pre-fly:
```
Vou seguir com defaults razoáveis:
- #X → tipo Y (justificativa de 1 linha)
- #Z → SLA W (justificativa)
Bloqueando só em: [item sem default seguro]
Confirma ou ajusta?
```

Sair desse passo só com **uma resposta consolidada** do Fabio (pode ser só "pode seguir").

### 1) Carregar contexto (sempre)

Logo no início, ler:
1. `clientes/_skill-ekyte/cache.md` — workspaces, projetos descobertos, tipos
2. `clientes/_skill-ekyte/sla.md` — SLA por sigla

Não chamar a MCP do ekyte sem antes consultar o cache.

### 2) Identificar o modo de input

**Modo A — texto livre no chat:**
Exemplos:
- "cria uma task de criativos ads pro Euro Colchões com foco em remarketing produto X, 6 peças"
- "sobe 3 LPs pro Fiberwan: blackfriday, prospect, retorno"
- "preciso de uma publicação de campanha pra Samech amanhã"

Extrair:
- **Cliente** (alias → workspace_id via cache)
- **Sigla/tipo** ("criativos ads" → CA → 29740)
- **Quantidade [NN]** ("3 LPs" → 03; default 01 se não especificado)
- **Demanda específica** (parte que vira o título depois do `|`)
- **Prazo** (default = SLA da tabela; sobrescrever se usuário disser "urgente", "amanhã", "X dias")

**Modo B — aba da planilha:**
Quando o Fabio disser "tá na aba `<nome>`" ou similar, ler:
- URL: `https://docs.google.com/spreadsheets/d/1SAWHbC3dog5_IrwCYY1_buF5yGtV_lMTiXL_v9iGrns/edit`
- Usar WebFetch com prompt pedindo: "lista todas as linhas da aba `<nome>` com colunas sequencial, titulo, data inicio, data entrega, tipo de tarefa (id), email executor, esforço, descricao, Qtd Peças, Tags, Fase"
- Cada linha vira 1 task. A planilha já tem o `task_type_id` numérico, datas em ISO, email do executor — **não precisa inferir IDs**.

### 3) Resolver workspace e projeto

**Workspace:**
- Procurar o cliente no cache (8 fixos + apelidos + workspaces internos)
- Se ambíguo (ex: usuário disse só "Eleva" mas cache tem variações): perguntar qual
- Se não estiver nos 8 fixos NEM nos internos conhecidos: PARAR e pedir confirmação explícita ("Esse cliente não está na sua lista habitual. Confirma criar mesmo?"). **Workspaces internos recorrentes** (V4 Company / Billions Timesheet, qualquer setup interno V4 que o Fabio usa toda semana) ficam no cache na seção "Workspaces internos" e NÃO disparam essa confirmação.

**Projeto (Q vigente):**
- Padrão de nome: `<Cliente> | Q2/2026` (Q2 ativo até virar trimestre)
- Se já tem `project_id` no cache para esse workspace + período: usar
- Se não tem: chamar MCP `ekyte.listar_projetos_tool` com `workspace_id`, `created_from` = início do trimestre, `created_to` = fim do trimestre. Encontrar projeto cujo nome contém `Q2/2026`. Salvar no cache.
- Se aparecer >1 match: perguntar qual

### 4) Resolver tipo de tarefa (sigla → task_type_id)

- Se o Fabio falou em palavras ("criativos ads"): mapear pra sigla (CA) e buscar no cache.
- Se a sigla tem 1 só ID: usar.
- Se a sigla tem múltiplos (CA, CJ, GP, MT, AUX, CRM, PMM, RI, SM): **desambiguação obrigatória** — listar opções e perguntar qual.
- Se não achar a sigla no cache: chamar `ekyte.listar_tipos_de_tarefas_tool({"name_type_task": "<termo>", "parameters1_Value": "3535"})` (workflow Padrão Colli&Co), filtrar resultado, perguntar.

**Caso especial WEB:** se o pedido é "demanda web", usar o título com `[WEB]` no slot da sigla. Tipo de tarefa = "Personalizada". Se ainda não temos `task_type_id` da Personalizada no cache, perguntar ao Fabio qual ID usar e salvar.

### 5) Montar o título

```
[NN][SIGLA][IA] Cliente | Demanda específica
```

Onde:
- `NN` = quantidade em 2 dígitos (`01`, `09`, `15`)
- `SIGLA` = sigla do tipo (CA, LP, PC, RV, GP…) ou `WEB` para personalizadas
- `[IA]` = sempre, marca que foi gerada via skill
- `Cliente` = nome curto do cliente (sem `[BILLIONS]`, sem brackets)
- `Demanda específica` = parte livre, descritiva

Exemplos:
- `[09][CA][IA] Euro Colchões | Remarketing produto X`
- `[01][LP][IA] Fiberwan | Página obrigado transceivers`
- `[03][WEB][IA] Eleva | Ajustes header e footer site institucional`

### 6) Montar o briefing (description) — **delegado pra `/ekyte-briefing`**

A partir de 2026-04-30, a montagem de briefing é responsabilidade da skill `/ekyte-briefing`. Esta skill (`/ekyte-task`) **não** monta mais o `description_create_task` por conta própria.

**Como invocar:** depois de resolver workspace, projeto, sigla e tipo (passos 3-4), montar o pacote de entrada e chamar `/ekyte-briefing`:

```json
{
  "cliente": "Euro Colchões",
  "cliente_alias": "euro",
  "sigla": "CA",
  "task_type_name": "Criativo Ads",
  "task_type_id": "29740",
  "qtd": 9,
  "titulo": "[09][CA][IA] Euro Colchões | Remarketing produto X",
  "input_livre": "<input original do Fabio>",
  "modo": "texto_livre",          // ou "planilha_demandas" ou "5w1h"
  "planilha_5w1h_url": null,
  "planilha_demanda_row": null    // preenchido se modo=planilha_demandas
}
```

A `/ekyte-briefing` faz: carrega template da sigla, lê Drive em `drives.md`, lê NotebookLM do `CLAUDE.md` do cliente, **invoca `/cs-notebooklm-consulta-cliente`** pra puxar contexto, faz perguntas ativas em lote ao Fabio, monta briefing em Markdown e converte pra HTML do Quill (Ekyte).

**Devolve:**
```json
{
  "briefing_html": "<div>...</div>...",
  "briefing_markdown": "BRIEFING — ...",
  "campos_pendentes": [],
  "notebook_consultado": true,
  "notebook_artefato": "clientes/euro/contexto-notebook/2026-04-30-1015-...md"
}
```

A `/ekyte-task` injeta `briefing_html` direto em `description_create_task` na chamada `criar_tarefa_tool`.

**Importante (achado da v2):** o Quill do Ekyte aceita HTML mas a v1 vinha mandando texto plano (sem `<div>`/`<br>`), por isso briefings antigos viraram parede de texto. A `/ekyte-briefing` corrige isso na conversão Markdown→HTML.

**Modo planilha de demandas (Modo B antigo):** quando o input vem da aba da planilha (`https://docs.google.com/.../edit`), a `/ekyte-task` extrai a linha (`descricao`, `task_type_id`, `email`, etc) e passa pra `/ekyte-briefing` com `modo: "planilha_demandas"` + `planilha_demanda_row` preenchido. Se a coluna `descricao` for substancial, a `/ekyte-briefing` usa como base e enriquece via NotebookLM. Se for vazia/genérica, monta do template normal.

**Modo 5W1H:** quando Fabio menciona link de planilha 5W1H ("plano de ação"), passar `modo: "5w1h"` + `planilha_5w1h_url`. A `/ekyte-briefing` valida o cabeçalho e usa layout 5W1H.

### 7) Calcular prazo

Consultar `sla.md`:
- Tipo na tabela → `current_due_date_create_task` = hoje + SLA
- Tipo NÃO na tabela → perguntar no preview, ou usar default genérico (3 dias úteis) se o usuário disser "padrão"
- Usuário disse "urgente" / "amanhã" / data específica → sobrescrever

`phase_start_date_create_task` = sempre hoje.

### 8) Preview obrigatório

Antes de chamar `criar_tarefa_tool`, mostrar pra Fabio:

```
📋 PREVIEW — Task #1 de N

Workspace: Euro Colchões (id: 124061)
Projeto: Euro Colchões | Q2/2026 (id: <X>)
Tipo: [CA] Criativo Ads (id: 29740)
Responsável: fabiojose.aiello@v4company.com
Início: 2026-04-28
Prazo: 2026-05-09 (SLA CA = 11 dias)

Título: [09][CA][IA] Euro Colchões | Remarketing produto X

Briefing:
<<resumo do briefing — primeiras 5 linhas>>
[ver completo abaixo]

Após criação: vou disparar `generate-tasks` no projeto pra
sair de "Não planejada" → "Ativa" automaticamente (passo 9.5).

---

Confirma criação + geração? (sim/não/editar)
```

Se for lote (modo B com planilha): mostrar tabela resumida (linha | título | tipo | prazo) e pedir aprovação **única** pro lote, OU "1 a 1" se o Fabio preferir.

**Nunca** dispara `criar_tarefa_tool` sem ver o "sim".

### 8.4) Modo Inline Rápido (3-5 tasks com OBS robusta) — lição 2026-05-05, revisado 2026-05-14

Quando lote é **3-5 tasks** + a coluna `OBS/Descrição` da planilha já tem texto substancial (objetivo claro, link de transcrição/referência) + cliente tem NotebookLM cadastrado: **pular invocação formal de `/ekyte-briefing`** e montar briefings inline em texto plano direto, usando os templates da sigla só como guia estrutural.

Regra prática:
- Lote ≤5 + OBS substancial + cliente conhecido → **inline rápido** (este modo). Consumir cache persistente de público (8.4.1).
- Lote ≤5 + OBS vazia/genérica OU cliente novo → **fluxo formal** com `/ekyte-briefing` (passo 6 padrão).
- Lote ≥6 → **modo script Python** (8.5).

Por que funciona em 3-5: o overhead de invocar `/ekyte-briefing` (consulta NotebookLM + perguntas ativas) só compensa quando o input é raso. Se a planilha já tem briefing parcial pronto, basta enriquecer estruturalmente (Objetivo / Contexto / Entregáveis / Sucesso) e disparar.

Validado em 2026-05-05 com lote de 5 tasks (Fiberwan x2 + Outmat x3): 5 criações em paralelo, ~30s.

#### 8.4.1) Público vem do cache persistente, sempre

Lição 2026-05-14: a v1 do modo inline pulava NotebookLM totalmente — campo público acabava saindo "a consultar" ou genérico. Conflito direto com a regra [public sempre vem do NotebookLM](../../../memory/feedback_publico_notebook.md).

**Resolução:** modo inline consulta `clientes/<cliente>/publicos-cache.md` antes de montar briefing. Layout e TTL: ver [_publicos-cache-template.md](../../../clientes/_skill-ekyte/_publicos-cache-template.md).

Fluxo:

1. **Identificar a linha** de cada task do lote (Colchões adultos / Euro Baby / Geral / etc).
2. **Para cada linha distinta no lote**, abrir o cache uma vez:
   - **HIT** (< 75d) → usa direto, não consulta NotebookLM. Briefing inline ganha bloco "Público" preenchido (avatar + faixa + sexo + consciência + ganchos) com marcador `[do cache: <linha> · <N>d]`.
   - **STALE** (75-90d) → pergunta no pre-fly: `⚠️ cache de público da linha "Euro Baby" tem 78d. Atualizar antes do lote? (sim/não)`. Sem resposta = HIT silencioso.
   - **MISS** (≥ 90d OU inexistente) → invocar `/cs-notebooklm-consulta-cliente` **uma única vez por linha do lote**, com 1 pergunta dirigida só de público (não as 5 padrão). ~1min/linha. Após resposta, escrever bloco no cache.
3. **Não consultar NotebookLM mais de uma vez por linha por lote.** Se 4 tasks são todas "Colchões adultos", consulta roda 1x (no MISS) e as 4 reusam.

Quando MISS rola, o cache populado fica disponível pras próximas sessões — paga o investimento em 2-3 lotes.

### 8.5) Modo Lote — script Python (≥6 tasks)

Quando o lote for ≥6 tasks, montar briefing inline na chat custa contexto e fica frágil. **Default a partir de 6:** gerar tudo via script Python parametrizado.

Estrutura recomendada:
1. Criar pasta de trabalho `<workspace_user>/briefings-ekyte-<YYYY-MM-DD>/`.
2. Escrever `gerar_briefings.py` que define funções reaproveitáveis por padrão de demanda (ex: `web_pdp(cliente, ws_id, proj_id)`, `fotos(filename, qtd_por_sku, total_skus, ...)`) e chama essas funções pra cada task. Saída: 1 `.docx` por task (use `python-docx`).
3. Escrever `extrair_textos.py` que converte os .docx em texto plano formatado (emojis numerados pras seções, bullets `•`, sem HTML — feedback consolidado em [Ekyte description = texto plano](feedback_ekyte_descricao_texto_plano.md)). **Pular o bloco "Metadados (Ekyte)"** — esses dados já vão nos campos próprios da MCP. Salva tudo num `_briefings_textos.json` chave→texto.
4. Disparar as MCP `criar_tarefa_tool` em paralelo (várias por mensagem) com `description_create_task` lendo do JSON.

Vantagens medidas (lote de 25 em 2026-05-04):
- Estrutura uniforme (toda task cobre Objetivo / Contexto / Escopo / Entregáveis / Perguntas / Referências).
- Reaproveitamento de padrão (8 fotos Euro = 1 função × 6 chamadas, não 6 briefings escritos à mão).
- Os .docx ficam de subproduto pro Fabio revisar antes do disparo se quiser.

### 9) Chamar a MCP

Após confirmação, chamar `ekyte.criar_tarefa_tool` com os 8 campos:

```json
{
  "workspace_id_create_task": "124061",
  "ctc_task_type_id_create_task": "29740",
  "user_email_create_task": "fabiojose.aiello@v4company.com",
  "title_create_task": "[09][CA][IA] Euro Colchões | Remarketing produto X",
  "description_create_task": "<HTML do briefing>",
  "current_due_date_create_task": "2026-05-09",
  "ctc_task_project_id_create_task": "<id do projeto>",
  "phase_start_date_create_task": "2026-04-28"
}
```

Reportar sucesso/erro de cada criação. Se erro: parar, mostrar a mensagem, não tentar a próxima sem aprovação.

### 9.5) Gerar tarefas (sair de "Não planejada" → "Ativa") — adicionado 2026-05-14

A MCP `ekyte` só cria a task; ela cai como **"Não planejada"** dentro do projeto e precisa do passo "Gerar tarefas" pra virar **"Ativa"** (executável, visível pro responsável, contando no kanban). Esse passo não está exposto na MCP — usar REST direto.

**Endpoint** (descoberto e validado 2026-05-14 via DevTools + teste curl real):
```
POST https://api.ekyte.com/api/v2/companies/{company_id}/projects/{project_id}/generate-tasks
Headers:
  Authorization: Bearer <EKYTE_TOKEN>
  Content-Type: application/json
Body: []
```

**Atenção ao body:** é **array vazio `[]`**, não objeto `{}`. Mandar `{}` retorna 422 com `"requires a JSON array"`. O backend é .NET e desserializa em `System.Int64[]`. Hipótese: passar `[123, 456]` ativaria só essas tasks específicas, mas `[]` ativa **todas** as "Não planejadas" do projeto (validado 200 OK + `plannedTasksCount: 0` no retorno).

**Comportamento:** ativa **todas** as tasks "Não planejadas" do projeto de uma vez — não é por-task, é por-projeto. Validado em 2 projetos reais (Zabeu 297264 e Outmat 291499 em 2026-05-14/15).

**Formato da resposta 200 OK (duas variantes possíveis — preparar pra ambas):**

- **Variante A — array de tasks** (quando o projeto tem poucas "Não planejadas" pendurinhas): `[{id, ctcTaskType, executor, phase, ...}, ...]`. `N ativadas = len(response)`.
- **Variante B — objeto `{project: {...}}` com métricas** (quando o projeto tem muitas tasks no total): a resposta é o objeto completo do projeto. Campos chave pra reportar:
  - `unplannedProjectTasksCount` → **deve ser 0** após sucesso (confirma que ativou tudo que tinha pra ativar)
  - `plannedActiveProjectTasksCount` → quantas tasks estão ativas no projeto agora
  - `overduePlannedProjectTasksCount` → bônus: quantas estão atrasadas (alerta opcional)

**Reportar pro Fabio independente da variante**: `✅ Projeto <nome>: 0 tasks não-planejadas. <X> ativas no total.`

**Implementação no fluxo:**

1. Ler `clientes/_skill-ekyte/.env` (formato em `.env.example`). Se arquivo não existe, avisar Fabio: `"⚠️ clientes/_skill-ekyte/.env não cadastrado. Tasks criadas vão ficar como 'Não planejada' até você gerar manualmente no Ekyte. Quer cadastrar o token agora? (sim/não)"`. Se "não", pular este passo e seguir pro 10.

2. **Agrupar tasks criadas por `ctc_task_project_id_create_task`**. Lote de 5 tasks em projetos diferentes = 5 chamadas; lote de 10 tasks todas no mesmo Q2 do Euro = 1 chamada.

3. **Para cada `project_id` único**, disparar:
   ```bash
   curl -X POST "$EKYTE_API_BASE/companies/$EKYTE_COMPANY_ID/projects/<project_id>/generate-tasks" \
     -H "Authorization: Bearer $EKYTE_TOKEN" \
     -H "Content-Type: application/json" \
     -H "Origin: https://app.ekyte.com" \
     -H "Referer: https://app.ekyte.com/" \
     -d '[]'
   ```
   **Body = `[]` (array vazio).** Mandar `{}` retorna 422.

4. **Tratamento de resposta:**
   - **200 OK** → reportar `✅ <N> tasks do projeto <nome> ativadas`.
   - **401 Unauthorized** → token expirou. Avisar Fabio: `"❌ Token EKYTE_TOKEN expirou. Renova via DevTools (F12 → Network → qualquer request pra api.ekyte.com → Headers → copia Authorization). As tasks foram criadas mas ficaram como 'Não planejada' — depois de renovar, pode rodar /ekyte-task-gerar <project_id> ou ativar manualmente."`. **Não tentar refresh automático** — token é JWT manual.
   - **Outros erros** → reportar status + body, não interromper outras chamadas de outros projetos, mas avisar no relatório final.

5. **Relatório consolidado** após todas as chamadas:
   ```
   ✅ 8 tasks criadas
   ✅ Geradas: Euro Colchões | Q2/2026 (5 tasks), Fiberwan | Q2/2026 (3 tasks)
   ```

   Se algum projeto falhou na geração:
   ```
   ✅ 8 tasks criadas
   ✅ Geradas: Euro Colchões | Q2/2026 (5 tasks)
   ⚠️ NÃO geradas: Fiberwan | Q2/2026 (3 tasks) — erro 401, token expirado
   ```

### 10) Atualizar cache

Sempre que descobrir um `project_id` novo (ou um `task_type_id` confirmado de tipo Personalizada), atualizar `clientes/_skill-ekyte/cache.md` antes de encerrar.

---

## Guardrails (não-negociáveis)

1. **Lock de cliente por sessão.** Se o Fabio começou trabalhando "Euro Colchões", não criar tasks em outro workspace sem confirmação explícita ("acabamos com Euro, agora estou indo pra Fiberwan").

2. **Desambiguação obrigatória** em:
   - Cliente com >1 match (rara, mas possível com nomes parecidos)
   - Sigla com >1 task_type_id (CA, CJ, GP, MT, AUX, CRM, PMM, RI, SM)
   - Projeto com >1 match para o período pedido

3. **Preview obrigatório** antes de cada criação. Sem exceção. Mesmo no modo B em lote, exibir resumo + esperar "ok".

4. **Workspace fora dos 8** → PARAR e pedir confirmação explícita. A conta é master Collie, blast radius alto.

5. **Erro = parar.** Se uma criação falhar (HTTP error, timeout, schema rejeitado), pausar o fluxo e reportar. Não seguir pra próxima task do lote sem aprovação.

6. **Sem auto-execução.** Mesmo que o Fabio diga "confia, manda tudo", insistir no preview da primeira task pelo menos.

7. **Gerar tarefas é parte obrigatória do fluxo.** Toda task criada deve sair de "Não planejada" no final da execução. Se token expirou ou `.env` não existe, **avisar explicitamente** que as tasks ficaram em rascunho — Fabio não pode descobrir isso depois.

8. **Token Ekyte é sensível.** Nunca logar/imprimir o valor de `EKYTE_TOKEN` nas mensagens pro Fabio (mesmo truncado). Só dizer "token presente/ausente/expirado".

---

## Como invocar

- `/ekyte-task` — usuário chama explicitamente
- Detectar automaticamente quando o Fabio:
  - Disser "cria/sobe/abre uma task/tarefa no ekyte"
  - Mencionar uma sigla `[CA]`, `[LP]`, etc. seguida de cliente
  - Apontar aba da planilha de demandas pra subida em lote

## O que NÃO fazer

- Não inventar `task_type_id`, `workspace_id` ou `project_id`. Se não está no cache, buscar ou perguntar.
- Não criar workspace, projeto ou tipo novos por conta própria. Skill v1 só **cria task**, nunca infraestrutura.
- Não pular o preview, mesmo que o Fabio peça pressa.
- Não escrever briefing do zero quando o tipo tem template oficial — sempre usar o template e preencher.
- Não usar tag nativa do ekyte na chamada da MCP — o schema não expõe. Marcar IA pelo título `[IA]`.
