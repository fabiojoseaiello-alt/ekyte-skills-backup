---
name: contexto
description: Le todos os arquivos em uma KB (cliente, squad ou projeto), gera CLAUDE.md e AGENTS.md, e quando for cliente cria/atualiza mission-control/ com OKRs, apostas vivas, combinados, personas e historico de check-ins. Detecta o nivel automaticamente. Use quando o usuario rodar /contexto, quiser que a IA "conheca" um cliente/squad/projeto, ou quiser criar/atualizar Mission Control de cliente.
---

Voce vai analisar uma Knowledge Base e gerar os arquivos `CLAUDE.md` e `AGENTS.md` que funcionem como "memoria" pra qualquer trabalho futuro. Quando a KB for de CLIENTE, tambem crie/atualize `mission-control/`, que e o estado vivo usado por skills de check-in.

## Estrutura esperada

- `squads/{squad}/` — pasta de squad (tem `README.md` com membros, `docs/`, e subpasta `clientes/`)
- `squads/{squad}/clientes/{cliente}/` — pasta de cliente (tem `calls/`, `checkins/`, `docs/`, `campanhas/`, `links.md`, e pode ter `mission-control/`)
- `bases/{projeto}/` — pasta de projeto/area (tem `docs/`, `dados/`, `referencias/`)

**Padrao obrigatorio:** todo cliente vive em `squads/{squad}/clientes/{cliente}/`. Cliente solto, fora de squad, nao existe.

## Processo

### Passo 1 — Identificar a base

Detecte o que rodar com base na pasta corrente do usuario:

- **Pasta corrente e cliente** (tem `calls/`, `docs/`, `campanhas/`; `checkins/` pode existir ou ser criado; E nao tem subpasta `clientes/`): use ela direto.
- **Pasta corrente e squad** (tem subpasta `clientes/` E `README.md`): use ela direto.
- **Pasta corrente e projeto** (`bases/{X}/`): use ela direto.
- **Caso contrario:** liste todas as KBs disponiveis e pergunte:
  - Squads: `squads/*/` (ignorando `_template-*`)
  - Clientes: `squads/*/clientes/*/`
  - Projetos: `bases/*/` (ignorando `_template`)

### Passo 2 — Detectar o tipo

- **CLIENTE** (operacao): tem `calls/`, `docs/`, `campanhas/`; `checkins/` e recomendado
- **SQUAD**: tem subpasta `clientes/` e `README.md` com membros
- **PROJETO/AREA** (generico): tem `docs/`, `dados/`, `referencias/`

### Passo 3 — Ler tudo

Leia TODOS os arquivos da pasta:

- **Cliente**: leia tudo em `calls/`, `checkins/`, `docs/`, `campanhas/`, `links.md`, `mission-control/` se existir, e qualquer outro arquivo.
- **Squad**: leia `README.md` e tudo em `docs/`. NAO leia o conteudo dos clientes filhos — so liste os nomes das pastas.
- **Projeto**: leia tudo recursivamente.

Leia cada arquivo por completo. Nao pule nada.

### Passo 4 — Analisar e gerar

**Se for CLIENTE (operacao):**

Extraia: nome da empresa, segmento, produto/servico, publico-alvo, diferenciais, canais, investimento, metricas, contatos, combinados, pendencias, objetivos, teses, historico, proximos passos e aprendizados de check-ins. Tambem inclua os links uteis encontrados em `links.md`.

Gere o `CLAUDE.md` e o `AGENTS.md` (mesmo conteudo) com:

```markdown
# [Nome da Empresa]

## Resumo
[2-3 frases: quem e, o que faz, momento atual]

## Recursos
Veja `links.md` na raiz desta pasta pra todos os links uteis.
[Liste aqui os principais inline: NotebookLM, Drive, site — pra ja entrarem no contexto cascateado.]

## Negocio
- **Segmento:** [X]
- **Produto/Servico:** [X]
- **Publico-alvo:** [X]
- **Diferenciais:** [X]

## Operacao
- **Canais ativos:** [X]
- **Investimento:** [X/mes]
- **Metricas atuais:** [CPC, CPL, ROAS, etc]
- **Problemas:** [X]
- **Oportunidades:** [X]

## Relacionamento
- **Contatos:** [nomes e funcoes]
- **Combinados:** [o que foi prometido/acordado]
- **Pendencias:** [entregas pendentes]

## Estrategia
- **Objetivos:** [X]
- **Teses atuais:** [X]
- **Historico:** [o que ja testaram]
- **Proximos passos:** [X]

## Notas Importantes
[Qualquer informacao critica que nao se encaixou acima]

## Quando trabalhar com este cliente
- Comece lendo `links.md` pra saber dos recursos disponiveis.
- Se o usuario compartilhar um link util durante a conversa, pergunte se quer adicionar a `links.md`.
```

Depois, crie ou atualize `mission-control/` na raiz do cliente com os 5 arquivos abaixo. Se o arquivo ja existir, preserve informacoes historicas e atualize com base no material novo. Nao apague aprendizados antigos sem evidencia clara de que ficaram obsoletos.

Garanta tambem que exista a pasta `checkins/` na raiz do cliente. Ela guarda pautas, ensaios, reviews e relatorios de check-in. Nao coloque transcripts brutos ali; transcripts ficam em `calls/`.

```text
mission-control/
|-- okr-quarter.md
|-- apostas-vivas.md
|-- combinados.md
|-- personas-call.md
`-- historico-checkins.md
```

**`okr-quarter.md`**
- Objetivo do quarter atual.
- KRs mensuraveis.
- Status atual e mes N de 3.
- Fonte usada (planejamento pos-kickoff, kickoff, check-in, docs).
- Se nao houver OKR explicito, escreva `[nao encontrado nos docs disponiveis]` e liste o que o account precisa preencher.

**`apostas-vivas.md`**
Use a tabela obrigatoria:

```markdown
| Aposta (o que cremos) | Por que apostamos | Como mata (sinal + prazo) | Plano B se morrer |
|---|---|---|---|
```

Registre 3 a 5 apostas estrategicas atuais. Cada aposta precisa ser testavel. Quando inferir criterio de morte ou plano B, marque `[INFERIDO - confirmar com account]`.

**`combinados.md`**
Separe pendentes, em andamento e feitos. Use o schema:

```markdown
- [ ] {dono} {acao} ate {prazo}
- [->] {dono} {acao} (em andamento)
- [x] {dono} {acao} (feito em {data})
```

Se nao houver dono ou prazo, marque `[A CONFIRMAR]`.

**`personas-call.md`**
Para cada stakeholder relevante, registre:
- Papel na conta.
- Arquetipo de call (ex: decisor agressivo, operacional cetico, estrategista, passivo).
- Voz e jeito de falar.
- Gatilhos.
- Padroes de provocacao.
- Como argumenta.
- Frases tipicas (citacao literal curta ou parafrase fiel).

Se nao houver check-ins salvos, pergunte ao account quais arquetipos parecem mais com os stakeholders e marque como `[declarado pelo account - refinar com proximas calls]`.

**`historico-checkins.md`**
Liste as calls em ordem cronologica:

```markdown
## YYYY-MM-DD - {Tipo da call}
**Modo:** TEM | SEM | ND
**Resumo (1 linha):** ...
**Transcript:** [link relativo](../calls/{arquivo}.md)
**Pontos criticos:**
- ...
```

Use `ND` para calls anteriores ao framework ROPRE V2 ou quando o modo nao estiver claro.

**Se for SQUAD:**

Extraia do `README.md`: nome do squad, membros (nome + funcao). Extraia de `docs/`: acordos do squad, processos, links uteis. Liste os clientes filhos (so os nomes das pastas).

Gere o `CLAUDE.md` e o `AGENTS.md` (mesmo conteudo) com:

```markdown
# Squad [Nome]

## Membros
- [Nome — Funcao]
- ...

## Clientes
- [nome-formatado-da-pasta]
- ...

## Acordos e processos
[Sintese do que esta em docs/. Se vazio, "Nada documentado ainda — adicione em docs/."]

## Notas Importantes
[Qualquer info critica que nao se encaixa acima]
```

**Se for PROJETO/AREA (generico):**

Extraia: nome, objetivo, pessoas, responsabilidades, dados, metricas, processos, workflows, problemas, oportunidades, decisoes, pendencias.

Gere o `CLAUDE.md` e o `AGENTS.md` (mesmo conteudo) com:

```markdown
# [Nome do Projeto/Area]

## Resumo
[2-3 frases: o que e, qual o objetivo, momento atual]

## Contexto
- **Objetivo:** [X]
- **Pessoas envolvidas:** [nomes e papeis]
- **Status atual:** [X]

## Dados
- **Metricas principais:** [o que foi encontrado nos dados]
- **Fontes:** [de onde vem os dados]

## Processos
- **Workflows identificados:** [o que a area faz]
- **Ferramentas usadas:** [se mencionado]

## Situacao Atual
- **Problemas:** [X]
- **Oportunidades:** [X]
- **Decisoes tomadas:** [X]
- **Pendencias:** [X]

## Notas Importantes
[Qualquer informacao critica que nao se encaixou acima]
```

### Passo 5 — Apresentar ao usuario

Mostre um resumo do que encontrou e os arquivos gerados. Pergunte:
- "Tem algo que eu errei ou que falta?"
- "Quer adicionar alguma informacao que nao estava nos arquivos?"

Ajuste conforme o feedback.

### Passo 6 — Confirmar

Salve e diga:
> "Pronto. Agora toda vez que voce trabalhar nessa pasta, a IA vai ler esse contexto automaticamente. Se os dados mudarem, rode `/contexto` de novo pra atualizar."

## Regras

- NAO invente informacoes. Se nao encontrou algo, deixe como "[nao disponivel]".
- Em `mission-control/`, diferencie evidencia direta de inferencia com `[INFERIDO]` ou `[A CONFIRMAR]`.
- Se a KB estiver vazia ou quase vazia, avise e sugira quais dados adicionar.
- Priorize fatos sobre interpretacoes.
- Mantenha os arquivos concisos — maximo 150 linhas.
- Em pasta de squad, NUNCA leia o conteudo de pastas de clientes filhos. Cada cliente tem seu proprio CLAUDE.md.
