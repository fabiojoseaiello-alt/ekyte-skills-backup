---
name: ekyte-briefing
description: Monta briefings ricos e estruturados pra tarefas do Ekyte com base em templates locais por sigla, contexto puxado do NotebookLM do cliente, e perguntas ativas estruturadas. Usado como subskill da `/ekyte-task` — recebe pacote {cliente, sigla, tipo, qtd, input, modo} e devolve HTML pronto pra entrar no campo `description_create_task` do Ekyte. Pode ser invocada avulsa quando o usuário só quer pensar/estruturar uma demanda sem subir task. Suporta modo 5W1H quando vem planilha de plano de ação.
user-invocable: true
---

# /ekyte-briefing — Geração de briefing rico pra tasks do Ekyte

Substitui o briefing "uma linha + `<<PREENCHER>>`" da v1 da `/ekyte-task` por um briefing estruturado, profundo e pré-preenchido com contexto do NotebookLM do cliente.

## Quando usar

**Modo subskill (principal)** — invocada pela `/ekyte-task` no passo 6 da criação de task. Recebe pacote estruturado, devolve briefing HTML pronto.

**Modo avulso** — Fabio chama direto:
- `/ekyte-briefing` + descrição livre
- "me brieffa um CA pro Euro com foco em remarketing produto X"
- "preciso pensar essa demanda CRM antes de subir — me ajuda a montar o briefing"

Em modo avulso, a skill devolve o briefing HTML **e** o Markdown limpo, e pergunta se Fabio quer subir como task (aí invoca `/ekyte-task`).

## Pré-requisitos

- MCP `ekyte` configurada (pra eventual lookup de tipo de tarefa).
- Cache `clientes/_skill-ekyte/cache.md` populado (workspaces/projetos/tipos).
- `clientes/_skill-ekyte/drives.md` populado (drives dos clientes).
- `clientes/_skill-ekyte/briefing-templates/*.md` presentes (templates por sigla + base + universal + 5W1H).
- Cliente alvo precisa ter `clientes/<cliente>/CLAUDE.md` com bloco `## NotebookLM` (ou skill segue sem síntese, mas avisa).
- `notebooklm-py` autenticado (pra `/cs-notebooklm-consulta-cliente` rodar). Se não estiver, skill pula a síntese e segue.

## Pacote de entrada (modo subskill)

A `/ekyte-task` passa:

```json
{
  "cliente": "Euro Colchões",
  "cliente_alias": "euro",
  "sigla": "CA",
  "task_type_name": "Criativo Ads",
  "task_type_id": "29740",
  "qtd": 9,
  "titulo": "[09][CA][IA] Euro Colchões | Remarketing produto X",
  "input_livre": "cria 9 CAs pro Euro com foco em remarketing produto X",
  "modo": "texto_livre",   // ou "planilha_demandas" ou "5w1h"
  "planilha_5w1h_url": null,  // preenchido se modo=5w1h
  "planilha_demanda_row": null  // preenchido se modo=planilha_demandas
}
```

## Fluxo

### 1) Carregar contexto inicial

Em paralelo (independentes):
- Ler `clientes/_skill-ekyte/drives.md` → mapa cliente → drive_link
- Ler `clientes/_skill-ekyte/backups-crm.md` (se sigla é CRM-relacionada)
- Ler `clientes/<cliente>/CLAUDE.md` → extrair Notebook ID do bloco `## NotebookLM`
- Carregar `clientes/_skill-ekyte/briefing-templates/_header-universal.md`
- Carregar template específico da sigla (ex: `CA.md`). Se sigla é criativa (CA/LP/RV), carregar também `_base-criativo.md`.

### 2) Decidir modo de operação

**Modo `texto_livre`** (default): segue fluxo padrão.

**Modo `5w1h`**: input contém link de planilha 5W1H + skill validou cabeçalho. Pula `_base-criativo.md`, carrega `_5w1h.md` e usa como layout principal. Skill abre a planilha (WebFetch) e extrai os 6W.

**Modo `planilha_demandas`**: input veio da planilha de demandas (já manipulada pela `/ekyte-task`). Skill recebe o `planilha_demanda_row` com colunas `descricao`, `Tags`, etc — usa `descricao` como base e enriquece via NotebookLM.

### 3) Cache de sessão NotebookLM

Verificar se já tem síntese cacheada pro cliente nesta conversa:

```
sessao_notebook_cache = {
  "Euro Colchões": { "sintese": "...", "ts": "2026-04-30 10:15", "perguntas": [...] }
}
```

- **Cache HIT** (cliente já consultado): reusar síntese.
- **Cache MISS** OU primeira task do cliente na sessão: seguir pro passo 3.5 antes de invocar NotebookLM.

### 3.5) Cache persistente de público (TTL 90d)

Antes de invocar `/cs-notebooklm-consulta-cliente`, checar `clientes/<cliente>/publicos-cache.md`. Formato e regras: ver [_publicos-cache-template.md](../../../clientes/_skill-ekyte/_publicos-cache-template.md).

1. **Identificar a linha/categoria** a partir do título/input:
   - Produto explícito no título (`Saint Tropez`, `Euro Baby`) → mapear pra linha conhecida.
   - Sinal contextual sem produto (`remarketing produto X`) → perguntar ao Fabio qual linha.
   - Cliente sem segmentação (Fiberwan, Outmat) → linha `Geral`.

2. **Abrir `publicos-cache.md`** do cliente. Buscar bloco `## <linha>` (match case-insensitive, ignora acentos).

3. **Decidir HIT/STALE/MISS:**
   - **HIT** (idade < 75d): pré-preencher campos 3-9 do `_base-criativo.md` (consciência, faixa, sexo, ganchos) + avatar/ofertas/restrições/tom-de-voz com os valores do cache. Marcar no preview: `[do cache: <linha> · <N>d]`. **Pula** o passo 4 (NotebookLM) — cache já cobre o público.
   - **STALE** (75d ≤ idade < 90d): usa o cache **mas** mostra aviso no preview da pergunta ativa: `⚠️ cache de público da linha "<X>" tem <N>d (expira em <M>d). Quer atualizar antes de seguir? (sim/não, default não)`. Resposta sim = força MISS path. Não/silêncio = HIT silencioso.
   - **MISS** (idade ≥ 90d OU bloco inexistente OU arquivo inexistente): segue pro passo 4 (NotebookLM completo). Após sucesso da consulta, **escrever o bloco no cache** (passo 4.5).

4. **Modo subskill chamado pelo modo inline rápido da `/ekyte-task`:** se o pacote de entrada vem com flag `modo_inline_rapido: true` (ver `/ekyte-task` passo 8.4), a estratégia muda:
   - HIT → usa cache, **não invoca NotebookLM em hipótese alguma**. Briefing inline montado direto da OBS da planilha + cache de público.
   - MISS/STALE-com-update → invoca `/cs-notebooklm-consulta-cliente` **só com a pergunta de público** (1 pergunta dirigida sobre avatar/faixa/sexo/consciência/ganchos da linha alvo), não as 5 do `_base-criativo.md`. ~1min em vez de ~5min. Após resposta, popular cache normalmente.

### 4) Invocar `/cs-notebooklm-consulta-cliente`

Compor as 5 perguntas dirigidas pelo tipo da task (cada template tem suas próprias na seção "Pré-preenchimento via NotebookLM").

Tarefa enviada:
```
"vou montar briefing de [tipo] (sigla [sigla]) pro [cliente]. preciso saber: [resumo das 5 perguntas]"
```

A skill `/cs-notebooklm-consulta-cliente` faz o trabalho pesado: decompõe, dispara, agrega, salva artefato. Retorna síntese estruturada.

**Se a `/cs-notebooklm-consulta-cliente` retornar erro** (sessão expirada, cliente sem NotebookLM, etc):
- Cliente sem NotebookLM cadastrado → seguir sem síntese, avisar Fabio.
- Sessão expirada → avisar Fabio (`"Sessão NotebookLM expirou. Roda 'notebooklm login' e tenta de novo. Continuo o briefing sem síntese?"`) e perguntar se segue ou aborta.

Cachear síntese em memória (sessão).

### 4.5) Popular cache persistente de público (após sucesso do passo 4)

Se o passo 3.5 deu MISS/STALE-com-update e o passo 4 (NotebookLM) retornou síntese válida, **escrever bloco no cache** antes de seguir pro 5:

1. Abrir/criar `clientes/<cliente>/publicos-cache.md` (formato em [_publicos-cache-template.md](../../../clientes/_skill-ekyte/_publicos-cache-template.md)).
2. Se já existe bloco `## <linha>`: **substituir inteiro** (preserva os demais blocos do arquivo).
3. Se não existe: **acrescentar ao final** do arquivo.
4. Atualizar header "Última escrita: YYYY-MM-DD".
5. Campos a gravar (NotebookLM silencioso em algum = `_não documentado_`, **nunca fabricar**):
   - `ult_consulta`, `expira` (ult_consulta + 90d), `fonte` (URL do notebook)
   - `Avatar` (texto narrativo 2-4 linhas)
   - `Faixa etária dominante` (bins)
   - `Sexo` (Masculino|Feminino|Ambos)
   - `Nível de consciência` (1-5 com labels)
   - `Ganchos com tração documentada` (lista)
   - `Ofertas que já rodaram`, `Restrições documentadas`, `Tom de voz`

Esse cache fica disponível pras próximas sessões (modo inline rápido da `/ekyte-task` consome direto sem precisar invocar essa skill).

### 5) Pré-preenchimento dos campos do template

Com a síntese em mãos, varrer o template da sigla e pré-preencher campos onde a síntese tem informação.

Marcar campos pré-preenchidos com `[sugerido pelo NotebookLM, confirma?]` no preview de pergunta. Se Fabio aceitar tudo, vira valor final; se editar, vai a edição.

### 6) Pergunta ativa em lote

Listar **todas** as perguntas do template em uma só rodada, numeradas. Pré-preenchidos aparecem com a sugestão visível.

Formato:

```
📋 BRIEFING: [09][CA][IA] Euro Colchões | Remarketing produto X

Pra montar o briefing, responde as perguntas abaixo. Pode responder tudo de uma vez,
em qualquer formato — se ficar dúvida, eu pergunto de novo só do que ficou solto.

NotebookLM consultado em 2026-04-30 10:15. Síntese disponível e usei pra
pré-preencher [N] campos. Você confirma ou ajusta abaixo.

──────────────────────────────────────────────

1) Tema/Produto: [sugerido: "Remarketing produto X — colchões mola ensacada R$ 2.5k+"]
2) Motivação: [sugerido: "Reativar carrinho abandonado e visitantes de PDP"]
3) Condição/Oferta (opcional):
4) Observações relevantes (opcional):
5) Objetivo (1-6, multi):
   1) Alcance  2) WhatsApp  3) Cadastro  4) Vídeo  5) Remarketing  6) Compra
   [sugerido: 5,6]
... (continua até a última)
```

Fabio responde. Skill interpreta — aceita formatos variados (números, texto livre, "ok pra todos os sugeridos").

**Se Fabio responder "ok" / "tudo ok pelos sugeridos"**: skill aceita os pré-preenchidos e pergunta só os campos sem sugestão.

### 7) Validar campos universais

Antes de montar o briefing final:

- **Drive do Cliente:** já lido em `drives.md`. Se cliente é DOM (sem Drive) ou não mapeado, perguntar e oferecer `/ekyte-briefing-refresh`.
- **NotebookLM:** já lido do CLAUDE.md.
- **KV** (sigla criativa): se Fabio não passou KV no input nem na pergunta ativa, perguntar agora ("KV específico pra essa campanha? cole o link ou diga 'usa KV padrão do Drive'"). Se "padrão do Drive", colocar `KV padrão (ver pasta Drive)`.
- **Referência:** se passou no input, usar. Senão, omitir.
- **Pra CRM:** Planilha Backup, Ferramenta, Acesso (perguntas A, B, C do `CRM.md`).

### 8) Montar Markdown completo

Ordem de composição:

```
BRIEFING — {{cliente_uppercase}} — {{tipo_uppercase}}
Tarefa: {{titulo}}

[bloco _header-universal.md preenchido]

[se sigla criativa: bloco _base-criativo.md preenchido]
[se modo 5w1h: bloco _5w1h.md no lugar do _base-criativo.md]

[bloco específico da sigla preenchido — CA.md / LP.md / etc]
```

### 9) Conversor Markdown → texto plano formatado (Ekyte)

⚠️ **DESCOBERTA 2026-04-30:** O Quill do Ekyte **não interpreta tags HTML** quando o conteúdo é enviado via API REST (campo `description_create_task`). Tags como `<div>`, `<h1>`, `<blockquote>`, `<strong>`, `<a>`, `<ul>` aparecem como **texto literal** no editor — não viram rich text. O Quill só interpreta HTML quando recebe via clipboard de fonte rich-text (ex: colado de um Google Doc) ou quando o usuário clica nos botões da toolbar.

**Conclusão: enviar via API = texto plano formatado.** O Ekyte renderiza isso em fonte monospace dentro de uma caixa estilo code block (fundo escurecido), preservando quebras de linha, bullets, emojis numerados, etc. Fica perfeitamente legível e organizado — só não tem cores nem `<h1>` rich.

Função `md_to_ekyte_plain(md_string) -> string`:

```
Regras (aplicadas linha-a-linha):

ESTRUTURA (preservar como texto):
1. Linha em branco                     → linha em branco (\n)
2. Linha começando com "## "           → "<emoji-numerado> TÍTULO EM CAIXA ALTA" (substitui ## por emoji ou só CAIXA ALTA)
3. Linha começando com "### "          → "TÍTULO Sub-seção:" (sem caixa alta)
4. Linha começando com "- " ou "* "    → "• texto"
5. Texto comum                         → mantém como está
6. **negrito**                         → REMOVER asteriscos, manter texto plano (texto monospace já dá ênfase visual)
7. *itálico*                           → REMOVER asteriscos, manter texto plano
8. [texto](url)                        → "texto: url"  OU  só "url" se texto for vazio/redundante
9. URLs soltas                         → manter como estão (Quill detecta e converte em link clicável automaticamente)

CONVENÇÕES VISUAIS (já que não temos cor/negrito):
- Cabeçalhos de seção principais: usar emojis numerados pré-fixos (1️⃣, 2️⃣, 3️⃣...) seguido de TÍTULO EM CAIXA ALTA
- Sub-seções: TÍTULO em caixa alta + ":" no fim, sem emoji
- Avisos / proibições: prefixar com ⚠️
- Listas/produtos: numeração explícita "1. ", "2. ", etc.
- Bullets: "•" (caractere unicode, mais limpo que "-")
- Separadores entre seções: linha em branco dupla ou linha de "─" se quiser destacar
```

**Exemplo de output bem formatado (referência: o briefing que renderizou bem na #2783891 quando colado):**

```
BRIEFING — EURO COLCHÕES — CRIATIVO ADS
Tarefa: [22][CA][IA] Euro Colchões | Fotos ambientalizadas linha de colchões adultos

🔗 ATIVOS DA CAMPANHA

Drive do Cliente: https://drive.google.com/drive/folders/...
NotebookLM: https://notebooklm.google.com/notebook/...
⚠️ NotebookLM compartilhado entre Euro Colchões e Eleva — esta task é Euro.

1️⃣ DIRECIONAMENTO DA CAMPANHA

Tema/Produto: Colchões — linha adulta completa do e-commerce
Motivação: Renovar banco de imagens dos PDPs
Observações Relevantes:
• Escopo é a linha adulta — excluir Euro Baby e Cama Euro Pet
• 2 variações de cena por produto (total 22 peças)

2️⃣ OBJETIVO

• Compra (uso primário: PDP)
Nota: não é mídia paga, é foto editorial pra PDP
```

**O que NÃO fazer:**
- ❌ Não enviar HTML (`<div>`, `<h1>`, `<strong>`, `<ul>`, etc) — vira texto literal feio
- ❌ Não enviar Markdown bruto com `**`, `##`, `[]()` — Quill não converte, fica feio
- ✅ Enviar **texto plano organizado visualmente**: emojis, caixa alta, bullets unicode, quebras de linha

### 10) Preview da `/ekyte-briefing` (modo subskill)

Mostrar pra Fabio o briefing **renderizado em Markdown** (mais legível no chat) **antes** de devolver pra `/ekyte-task`:

```
📋 BRIEFING MONTADO — [09][CA][IA] Euro Colchões | Remarketing produto X

[markdown completo aqui — Fabio lê, edita ou aprova]

──────────────────────────────────────────────
Confirma? (sim / editar campo X / regerar do zero)
```

Se Fabio aprovar, skill devolve **HTML Quill** pra `/ekyte-task` (que cuida do preview da task em si e da chamada MCP).

Se modo avulso (não foi chamada pela `/ekyte-task`), perguntar se quer subir como task ("quer que eu invoque a /ekyte-task pra subir? sim/não").

### 11) Devolver pra /ekyte-task

Output estruturado:

```json
{
  "briefing_html": "<div><strong>BRIEFING — EURO COLCHÕES — CRIATIVO ADS</strong></div>...",
  "briefing_markdown": "BRIEFING — EURO COLCHÕES...",
  "campos_pendentes": [],
  "notebook_consultado": true,
  "notebook_artefato": "clientes/euro/contexto-notebook/2026-04-30-1015-briefing-criativo-ca.md"
}
```

A `/ekyte-task` injeta `briefing_html` em `description_create_task` e segue o fluxo dela (preview de task, confirmação, MCP).

## Guardrails

1. **Cache de sessão NotebookLM é por cliente, não por sigla.** Sobreusar síntese de CA pra LP do mesmo cliente está OK — síntese trata do cliente, não do tipo. Mas pra cliente diferente, sempre re-consultar.

2. **Nunca fabricar dados.** Se NotebookLM disse 'NAO ENCONTRADO' pra avatar, **não** inventar avatar. Pré-preenchimento só usa o que veio literalmente da síntese. Se síntese silenciou, campo fica vazio e vai pra pergunta ativa.

3. **Perguntas ativas em lote única.** Não fragmentar em 12 perguntas separadas — faz UMA mensagem com todas, Fabio responde uma vez. Salvo se ele pedir uma por uma.

4. **Modo 5W1H só ativa com link explícito + validação.** Não detectar 5W1H "no chute" porque o input parece um plano. Sem link explícito, modo é texto_livre.

5. **Ekyte via API REST recebe TEXTO PLANO, não HTML.** Validado em 2026-04-30: tags HTML enviadas via `description_create_task` aparecem como texto literal no editor (não renderizam). Padrão definitivo: gerar texto plano formatado com emojis numerados (1️⃣2️⃣3️⃣), TÍTULOS EM CAIXA ALTA, bullets `•`, URLs soltas (Quill autodetecta), quebras de linha. Sem `<>`, sem `**`, sem `##`. Renderiza em fonte monospace (estilo code block) mas fica perfeitamente legível.

6. **Cliente sem NotebookLM cadastrado** (CLAUDE.md sem bloco `## NotebookLM`): skill segue sem síntese, avisa Fabio. Não tentar adivinhar contexto.

7. **Cliente sem Drive em `drives.md`** (DOM Suprimentos atualmente): perguntar no preview e oferecer `/ekyte-briefing-refresh` pra persistir.

8. **Não chamar MCP do Ekyte.** Esta skill **não** sobe nada — só monta briefing. A subida é da `/ekyte-task`.

9. **Cache de público não é compartilhado entre clientes**, mesmo quando NotebookLM é compartilhado (Euro+Eleva). Cada cliente tem seu `publicos-cache.md` — público é por marca, não por notebook.

10. **MISS de cliente sem NotebookLM cadastrado**: não cachear nada. Briefing segue com pergunta ativa pura, e o `publicos-cache.md` **não é criado** pra esse cliente até que ele tenha NotebookLM.

## Como invocar

- `/ekyte-briefing` — modo avulso, Fabio chama direto.
- Invocação automática pela `/ekyte-task` no passo 6 do fluxo dela.

## O que NÃO fazer

- Não chamar `criar_tarefa_tool` (responsabilidade da `/ekyte-task`).
- Não escrever briefing genérico de "uma linha" (motivo da skill existir).
- Não inventar avatar/oferta/restrição se NotebookLM não retornou.
- Não pular perguntas obrigatórias do template (Tema/Motivação são obrigatórios pra criativos).
- Não usar `<h1>`/`<h2>`/`<p>` (Ekyte/Quill usa `<div>`).
- Não rodar NotebookLM em paralelo (cara é browser automation lenta — sequencial).
