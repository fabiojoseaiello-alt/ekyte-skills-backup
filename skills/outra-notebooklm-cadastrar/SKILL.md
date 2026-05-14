---
name: outra-notebooklm-cadastrar
description: Cadastra ou atualiza em massa os links de NotebookLM de clientes existentes, escrevendo o bloco "## NotebookLM" no CLAUDE.md de cada cliente em clientes/<cliente>/. Use sempre que o usuario rodar /notebooklm-cadastrar, mencionar que quer "cadastrar os notebooks dos clientes", "colar uma lista de links de NotebookLM", "atualizar o NotebookLM de um cliente", "registrar o link do NotebookLM da empresa X", ou pedir pra adicionar/trocar link de NotebookLM em cliente que ja existe sem precisar re-rodar /novo-cliente. Tambem dispara quando o usuario diz que tem varios links de NotebookLM pra organizar de uma vez.
area: outra
author: junioraiellogestaodetrafego
version: 1.0.0
---

# notebooklm-cadastrar — Cadastro em massa de NotebookLMs

Cadastra ou atualiza os links de NotebookLM dos clientes existentes nos respectivos `clientes/<cliente>/CLAUDE.md`. Resolve o caso "cliente ja existe e eu quero adicionar (ou trocar) o link do NotebookLM dele agora", sem precisar re-rodar `/novo-cliente`.

A skill `/novo-cliente` ja cadastra NotebookLM na criacao. Esta aqui e pro depois — quando voce conseguiu o link de varios clientes de uma vez e quer cadastrar tudo num passe so.

## Quando usar

- Usuario roda `/notebooklm-cadastrar`.
- Usuario quer cadastrar varios links de NotebookLM de uma vez ("vou colar uma lista").
- Usuario quer atualizar/trocar o link de NotebookLM de um cliente especifico que ja tem pasta criada.

Se for cliente novo (sem pasta ainda), oriente o usuario a rodar `/novo-cliente` em vez disso — essa skill nao cria pasta.

## Fluxo

### Passo 1 — Pedir a lista

Peca ao usuario pra colar a lista de clientes e links. Aceite o formato livre, contanto que seja uma linha por cliente e tenha um separador entre nome e link. Exemplos validos:

```
cliente-exemplo: https://notebooklm.google.com/notebook/abc123
empresa-x - https://notebooklm.google.com/notebook/def456?pli=1
loja-y = https://notebooklm.google.com/notebook/ghi789
```

Mensagem sugerida:

> "Cola aqui a lista de clientes e links de NotebookLM, uma linha por cliente. Formato:
> `nome-do-cliente: https://notebooklm.google.com/notebook/<id>`
> Aceita `:`, `-` ou `=` como separador."

### Passo 2 — Processar cada linha

Pra cada linha colada:

**2a. Normalizar o nome do cliente** pra bater com nome de pasta:
- lowercase
- remover acentos
- trocar espacos e `_` por hifens
- remover caracteres que nao sejam `a-z`, `0-9` ou `-`

Exemplos:
- "Academia Estação Saúde" → `academia-estacao-saude`
- "Loja_Y" → `loja-y`
- "Empresa X" → `empresa-x`

**2b. Extrair o notebook ID** da URL. O ID e tudo que vem depois de `/notebook/` ate a proxima `/`, `?` ou fim da string. Aceite URLs com ou sem query string.

Exemplos:
- `https://notebooklm.google.com/notebook/abc123` → `abc123`
- `https://notebooklm.google.com/notebook/abc123?pli=1` → `abc123`
- `https://notebooklm.google.com/notebook/abc123/audio` → `abc123`

Se nao conseguir extrair (URL malformada, sem `/notebook/`), registre como **erro** e siga pro proximo.

**2c. Verificar se a pasta do cliente existe** em `clientes/<nome-normalizado>/`.

- Se **nao existir**: registre como **pulado** com motivo "pasta nao existe". Nao crie pasta automaticamente. No final, sugira `/novo-cliente` pros clientes pulados.

**2c+. Consulta-canario (valida o ID antes de salvar).** Antes de gravar o bloco no CLAUDE.md, dispare 1 pergunta-canario contra o notebook pra confirmar que o ID corresponde mesmo ao cliente:

```bash
PYTHONIOENCODING=utf-8 notebooklm ask "Existe algum documento neste notebook que mencione a empresa <NOME-DO-CLIENTE>? Se nao houver, responda exatamente NAO ENCONTRADO. Nao invente." --notebook <ID>
```

- Se a resposta cita a empresa ou retorna fontes relevantes: ID **valido**, segue pra 2d.
- Se vier `NAO ENCONTRADO` ou a resposta cita outra empresa diferente: ID **suspeito**. Registre como **erro** com motivo "ID nao corresponde ao cliente — verificar com usuario". Nao salve. (Acontece quando o usuario cola por engano o ID de outro cliente — cobranca de 30s evita 5 min de retrabalho na hora da consulta.)
- 30s de canario por cliente. Pular esse passo so se o usuario explicitamente disser "nao valida, ja conferi".

**2d. Atualizar o CLAUDE.md** se a pasta existe. Leia `clientes/<nome>/CLAUDE.md`:

- **Se ja tem o bloco `## NotebookLM`** (procure pela linha exata `## NotebookLM`): substitua o bloco inteiro (do `## NotebookLM` ate a proxima linha que comece com `## ` ou ate fim do arquivo) pelo bloco novo. Registre como **atualizado**.

- **Se nao tem o bloco**: insira o bloco logo depois do primeiro titulo `# ` do arquivo. Se o arquivo nao tem titulo `# `, insira no inicio. Registre como **cadastrado**.

**Bloco a inserir:**

```markdown
## NotebookLM
- **Link:** <URL completa>
- **Notebook ID:** <ID extraido>

Use `notebooklm` CLI com o notebook ID acima para consultar a base de conhecimento desse cliente, gerar podcasts ou resumos.
```

### Passo 3 — Resumo final

Depois de processar tudo, mostre um resumo claro:

```
Resumo do cadastro:

✓ Cadastrados (3): cliente-a, cliente-b, cliente-c
↻ Atualizados (1): cliente-d (link trocado)
⊘ Pulados (1): cliente-e — pasta nao existe (rode /novo-cliente)
✗ Erros (1): cliente-f — URL invalida ("xxx")

Proximo passo: roda /notebooklm-consulta-cliente <cliente> "<tarefa>" pra usar.
```

Use os simbolos acima ou equivalentes. Mostre todas as 4 categorias mesmo que vazias (`(0)` se nada caiu nela), pra ficar claro o que aconteceu.

## Exemplo completo

**Input do usuario:**
```
/notebooklm-cadastrar
```

**Skill responde:**
> "Cola aqui a lista de clientes e links de NotebookLM, uma linha por cliente. Formato:
> `nome-do-cliente: https://notebooklm.google.com/notebook/<id>`"

**Usuario cola:**
```
cliente-exemplo: https://notebooklm.google.com/notebook/abc123
empresa-x - https://notebooklm.google.com/notebook/def456
nao-existe = https://notebooklm.google.com/notebook/xyz000
```

**Skill processa e responde:**

```
Processando 3 clientes...

✓ cliente-exemplo: cadastrado (bloco novo em clientes/cliente-exemplo/CLAUDE.md)
↻ empresa-x: atualizado (link trocado em clientes/empresa-x/CLAUDE.md)
⊘ nao-existe: pulado (pasta clientes/nao-existe/ nao encontrada)

Resumo:
✓ Cadastrados (1): cliente-exemplo
↻ Atualizados (1): empresa-x
⊘ Pulados (1): nao-existe — rode /novo-cliente se quiser criar
✗ Erros (0): nenhum

Proximo passo: /notebooklm-consulta-cliente <cliente> "<tarefa>" pra consultar.
```

## Edge cases

- **Lista vazia ou sem linhas validas:** avise "nao consegui parsear nenhuma linha" e mostre o formato esperado.
- **Linha duplicada (mesmo cliente 2 vezes):** processe a ultima ocorrencia (sobrescreve), e mencione no resumo.
- **CLAUDE.md nao existe na pasta do cliente:** crie com titulo `# <Nome do Cliente>` (use o nome normalizado e capitalizado) + o bloco NotebookLM.
- **Nome com acentos sem pasta correspondente:** se a normalizacao nao bater com nenhuma pasta existente, liste pulado — nao tente "adivinhar" pasta proxima.

## Por que isso e importante

A skill `/notebooklm-consulta-cliente` depende do bloco `## NotebookLM` estar presente no CLAUDE.md do cliente pra descobrir qual notebook consultar. Sem esse cadastro, ela aborta com "cliente nao tem NotebookLM cadastrado". Esta skill aqui resolve o gargalo de cadastrar varios clientes existentes de uma vez.
