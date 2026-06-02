---
name: novo-cliente
description: Cria uma nova pasta de cliente na raiz de clientes existente do workspace, com CLAUDE.md/AGENTS.md iniciais, links.md, calls/, checkins/, docs/ e campanhas/. Use quando o usuario rodar /novo-cliente, disser que quer adicionar um cliente novo, ou pedir para cadastrar uma nova base de cliente.
---

Crie a pasta de um novo cliente usando a estrutura real do workspace atual. O padrao antigo com `squads/{squad}/clientes/{cliente}/` so deve ser usado como fallback quando ele existir; no setup atual do Fabio, os clientes ficam em `CLIENTES V4/` e `builders-hub/clientes/` aponta para essa pasta.

## Processo

### Passo 1 - Descobrir a raiz de clientes

Detecte a raiz de clientes nesta ordem:

1. Se existir `clientes/` no diretorio atual, use `clientes/`.
2. Senao, se existir `builders-hub/clientes/`, use `builders-hub/clientes/`.
3. Senao, se existir `CLIENTES V4/`, use `CLIENTES V4/`.
4. Senao, se existir `squads/`, liste os squads existentes em `squads/` ignorando `_template-*`, pergunte em qual squad o cliente entra, e use `squads/[squad]/clientes/`.
5. Se nenhuma dessas estruturas existir, pare e diga que nao encontrou raiz de clientes nem squads e que precisa criar/indicar a pasta base antes.

No workspace do Fabio, prefira comunicar assim quando detectar o junction:

> "Vou criar em `builders-hub/clientes`, que aponta para `CLIENTES V4`."

### Passo 2 - Nome do cliente

Se o usuario ja passou o nome junto do comando, use esse nome. Caso contrario, pergunte:

> "Qual o nome do cliente?"

Converta para lowercase-com-hifens, sem acentos. Exemplo: `Academia Estacao Saude` vira `academia-estacao-saude`. Guarde como `[cliente]` e mantenha o original como `[Nome do Cliente]`.

Antes de criar, verifique se `[raiz-clientes]/[cliente]` ja existe. Se existir, pare e diga o caminho encontrado; nao sobrescreva cliente existente.

### Passo 3 - Descobrir o template

Use o primeiro template existente nesta ordem:

1. `bases/_template/_template-cliente`
2. `builders-hub/bases/_template/_template-cliente`
3. `[raiz-clientes]/_template`

Se nenhum template existir, crie a estrutura minima manualmente:

```text
[cliente]/
|-- calls/
|-- checkins/
|-- docs/
`-- campanhas/
```

### Passo 4 - Criar a estrutura

Se encontrou template, copie o template para `[raiz-clientes]/[cliente]`.

Depois, garanta que estas pastas existam, mesmo que o template antigo nao tenha todas:

```text
calls/
checkins/
docs/
campanhas/
```

Se existir `.env.example`, copie para `.env`. Se nao existir, crie `.env` vazio. Credenciais ficam locais.

### Passo 5 - Coletar links uteis

Pergunte uma de cada vez. Enter sem digitar pula a pergunta.

1. > "NotebookLM desse cliente? Cola o link (formato `https://notebooklm.google.com/notebook/XXXXX`) ou Enter pra pular."
2. > "Pasta no Google Drive? Cola o link ou Enter pra pular."
3. > "Site do cliente? Cola a URL ou Enter pra pular."
4. Loop:
   > "Mais algum link util? Formato: 'descricao - URL' (ex: 'Looker do cliente - https://...'). Enter sem digitar encerra."

Aceite quantos links extras o usuario passar. Se NotebookLM tiver link, extraia o notebook ID, que e a parte depois de `/notebook/`.

### Passo 6 - Escrever `links.md`

Atualize `[raiz-clientes]/[cliente]/links.md` com:

```markdown
# Links uteis

Recursos recorrentes deste cliente. Atualize sempre que aparecer link novo.

## Bases de conhecimento
- **NotebookLM:** [URL ou -]
  - **Notebook ID:** [ID ou -]

## Drives e armazenamento
- **Google Drive:** [URL ou -]

## Web
- **Site:** [URL ou -]

## Outros
- [descricao - URL]
```

Itens nao informados ficam com `-`. A secao "Outros" recebe os itens do loop; se nao houver nenhum, use `- -`.

### Passo 7 - Escrever `CLAUDE.md` e `AGENTS.md`

Crie ou substitua `[raiz-clientes]/[cliente]/CLAUDE.md` e `[raiz-clientes]/[cliente]/AGENTS.md` com o mesmo conteudo:

```markdown
# [Nome do Cliente]

## Recursos
Veja `links.md` na raiz desta pasta pra todos os links recorrentes (NotebookLM, Drive, site, dashboards, etc).
[Se tiver NotebookLM, adicione uma linha: "Use `notebooklm` CLI com o notebook ID `[ID]` pra consultar a base."]

## Contexto
Rode `/contexto` apos adicionar dados (calls, docs, campanhas) pra gerar o contexto completo do cliente.

## Quando trabalhar com este cliente
- Comece lendo `links.md` pra saber dos recursos disponiveis.
- Se o usuario compartilhar um link util durante a conversa (Drive novo, dashboard, sheet, conta de anuncios), pergunte se ele quer adicionar a `links.md`.
```

### Passo 8 - Confirmar

Mostre a estrutura criada:

```text
[raiz-clientes]/[cliente]/
|-- CLAUDE.md
|-- AGENTS.md
|-- links.md         # links uteis (NotebookLM, Drive, site, outros)
|-- .env            # credenciais locais
|-- .env.example    # se veio do template
|-- calls/           # transcripts brutos
|-- checkins/        # pautas, ensaios e reviews de check-in
|-- docs/
`-- campanhas/
```

Diga:

> "Cliente criado em `[raiz-clientes]/[cliente]`. Os links que voce passou ja estao em `links.md`. Jogue os dados nas pastas (calls, docs, campanhas) e rode `/contexto` quando tiver pronto. O `.env` ta vazio - preenche conforme as skills pedirem."
