---
name: account-pesquisa-profunda-cliente
description: Pesquisa profunda de cliente para KB acionavel. PREMISSA OBRIGATORIA - dados do cliente ja na pasta (no minimo formulario de kickoff + transcricao de reuniao de vendas). Sem isso a skill nao roda. Usa esses dados pra preencher 4 prompts sequenciais pro Deep Research do Gemini (cliente+digital+regiao; produto+setor; consumidor; concorrencia). Opcional Perplexity. Nao usa Gem nem deep research no agente. Use antes de copy, conteudo, campanha ou LP.
area: account
author: guilhermelippert
version: 1.5.0
---

## REGRA CRITICA — paths absolutos sempre

**Toda operacao de filesystem (mkdir, rm, cp, mv, ls, find, Write, Edit) DEVE usar caminho absoluto comecando em `/Users/...` ou na raiz do repo.** Nunca path relativo.

**NUNCA use `cd` em comandos Bash.** O cwd persiste entre Bash calls e cria desastres tipo `clientes/X/calls/squads/squad-exemplo/clientes/X/docs/pesquisa-profunda/...` (estrutura aninhada duplicada porque um `cd` anterior mudou o cwd e um `mkdir -p` posterior com path relativo herdou esse cwd).

**Forma correta:**
- `mkdir -p "/Users/.../builders-hub/squads/{squad}/clientes/{cliente}/docs/pesquisa-profunda/prompts"` ✅
- `mkdir -p "squads/{squad}/clientes/{cliente}/docs/pesquisa-profunda/prompts"` ❌ (vai falhar se cwd mudou)
- `cd ".../calls" && wc -l *.md` ❌ (polui cwd da sessao)
- `wc -l "/Users/.../calls"/*.md` ✅

**Antes de qualquer `mkdir` ou `Write`:** se voce rodou Bash antes, valide que esta no cwd certo com `pwd` OU simplesmente use path absoluto (preferido). Quando em duvida, sempre absoluto.

**Apos criar a estrutura, faca `find <raiz-cliente> -type d` e confira que nao ha pastas duplicadas aninhadas.** Se aparecer `.../calls/squads/...` ou similar, remova e recrie com path absoluto.

---

# /account-pesquisa-profunda-cliente

Conduza o usuario a montar **Knowledge Base do cliente** para gerar resultado em copy, conteudo, campanha, LP/funil e decisoes.

**O agente normal nao faz deep research.** **Nao use Gem** nem orquestrador externo. O fluxo e:

1. **Verificar que a pasta do cliente ja tem dados internos** (gate obrigatorio).
2. **Ler e consolidar** esses dados num resumo `[DADOS INTERNOS CONSOLIDADOS]`.
3. Entregar **quatro prompts prontos** pro usuario colar no **Deep Research do Gemini** — uma rodada por vez, na ordem. Entre rodadas, o usuario anexa/cola o relatorio anterior junto com o proximo prompt.
4. **Opcional:** prompt pro **Perplexity** com foco social/reviews (no fim).

## Premissa (gate obrigatorio)

**A skill so roda se a pasta do cliente ja tiver dados internos.** Os prompts de Deep Research dependem desses dados pra serem especificos — sem isso, viram pesquisa generica e perdem valor.

**Minimo absoluto pra rodar:**

1. **Formulario de kickoff** preenchido (brief inicial do cliente).
2. **Transcricao da reuniao de vendas** (ou comercial/discovery — o que tiver).

**Ideal (se tiver, melhor ainda):** transcricoes de outras calls, proposta comercial, export de CRM, materiais que o cliente mandou (catalogo, deck, site interno), notas do gestor.

## Principios

- Nao invente fatos. Separe evidencias, hipoteses e lacunas.
- Material interno (kickoff, vendas, CRM) tem **prioridade** sobre inferencia da web.
- Nao crie pasta de cliente aqui. Cliente inexistente → `/novo-cliente`.
- **Gemini Deep Research:** quatro execucoes sequenciais conforme templates abaixo.
- **Perplexity:** complemento social com links (opcional).
- Ad libraries e scraping manual: `references/fontes-pesquisa-social.md`.

## Passo 1 — Identificar cliente e pasta

1. Pergunte squad e nome/slug do cliente; se ambiguo, liste `squads/{squad}/clientes/` (ignore `_template`) e pergunte.
2. Se nao existir pasta → mensagem fixa pra `/novo-cliente`.
3. Trabalhe em `squads/{squad}/clientes/<cliente>/docs/pesquisa-profunda/`.

Estrutura a criar ou completar:

```text
squads/{squad}/clientes/<cliente>/docs/pesquisa-profunda/
├── 00-briefing-pesquisa.md
├── 01-cliente-posicionamento-digital.md   # sintese pos DR-1
├── 02-mercado-setor-modelo-negocio.md     # sintese pos DR-2 (parte setor)
├── 03-produto-servico.md                  # sintese pos DR-2 (parte produto)
├── 04-concorrentes.md                     # sintese pos DR-4
├── 05-panorama-ads.md                     # checklist manual / pos DRs
├── 06-consumidor.md                       # sintese pos DR-3
├── 07-fontes-e-evidencias.md
├── prompts/
│   ├── dr-1-cliente-posicionamento-regiao.md
│   ├── dr-2-produto-setor.md
│   ├── dr-3-consumidor.md
│   └── dr-4-concorrencia.md
├── prompt-perplexity-social-opcional.md
└── bruto/
    ├── gemini/
    │   ├── dr-01-output.md
    │   ├── dr-02-output.md
    │   ├── dr-03-output.md
    │   └── dr-04-output.md
    └── perplexity/
```

Se `docs/` nao existir, crie.

## Passo 2 — Gate de dados internos (BLOQUEANTE)

**Antes de gerar qualquer prompt**, faca uma varredura na pasta do cliente atras de dados internos. Lugares pra olhar:

- `squads/{squad}/clientes/<cliente>/calls/` — transcricoes de kickoff, vendas, comerciais
- `squads/{squad}/clientes/<cliente>/docs/` — formulario de kickoff, brief, proposta
- `squads/{squad}/clientes/<cliente>/campanhas/` — material de campanha existente
- `squads/{squad}/clientes/<cliente>/links.md` — links uteis (NotebookLM, Drive, site)
- Qualquer outro arquivo na raiz do cliente

**Checklist do minimo:**

- [ ] Formulario de kickoff (brief inicial preenchido pelo cliente)
- [ ] Transcricao da reuniao de vendas (ou comercial / discovery)

**Se faltar qualquer um dos dois, PARE e mande a mensagem abaixo. Nao gere prompts. Nao prossiga.**

> Pra rodar a pesquisa profunda eu preciso, no minimo, de dois materiais ja na pasta do cliente:
>
> 1. **Formulario de kickoff** — o brief inicial preenchido com o cliente.
> 2. **Transcricao da reuniao de vendas** (ou comercial/discovery — o que voce tiver).
>
> Sem isso os prompts de Deep Research saem genericos e a KB nao serve. Faltando: **[X, Y]**.
>
> Como destravar:
> - Cola/salva o formulario de kickoff em `squads/{squad}/clientes/<cliente>/docs/`.
> - Cola/salva a transcricao em `squads/{squad}/clientes/<cliente>/calls/`.
> - Se tiver mais (outras calls, proposta, export CRM, deck, catalogo), joga junto — quanto mais, melhor.
>
> Quando estiver na pasta, roda `/account-pesquisa-profunda-cliente` de novo.

**Se tiver os dois (ou mais), prossiga.** Nao roda parcialmente "com o que tem" — o gate e duro de proposito.

## Passo 3 — Consolidar `[DADOS INTERNOS CONSOLIDADOS]`

Le tudo que achou na varredura e produz um bloco de **resumo fiel** com:

| Campo | Origem |
|-------|--------|
| Identidade (nome fantasia, razao, cidade/UF, abrangencia) | Kickoff + vendas |
| Digital (site, IG, TikTok, YouTube, LinkedIn, GMN, marketplaces) | Kickoff + links.md |
| Oferta (produtos/servicos, pacotes, B2B/B2C, ticket/faixa) | Kickoff + vendas |
| Mercado (concorrentes citados, restricoes — verba, compliance, canais) | Vendas + kickoff |
| Dores/objetivos do cliente (o que ele quer destravar) | Vendas |
| Historico (tentativas anteriores, o que funcionou/falhou) | Vendas |
| Citacoes literais relevantes do cliente | Transcricao (entre aspas) |
| Objetivo da KB (copy / campanha / LP / diagnostico) | Pergunta direta se nao tiver |

**Salve esse resumo em `00-briefing-pesquisa.md`** — vira a fonte unica que vai dentro de cada prompt como `<<<DADOS INTERNOS CONSOLIDADOS>>>`.

Se o cliente afirmou algo critico mas duvidoso na call, marque **VALIDAR**. Se uma area ficou vazia mesmo com kickoff+vendas, marque **LACUNA** — entra como pergunta pro DR.

## Passo 4 — Entregar os prompts AO USUARIO (no chat E em arquivo)

**REGRA:** os 4 prompts do Gemini + o prompt do Perplexity tem que aparecer **direto na conversa**, dentro de blocos `code` copiaveis, com o `<<<DADOS INTERNOS CONSOLIDADOS>>>` ja substituido pelo bloco real do Passo 3. Salvar nos arquivos NAO basta — o usuario precisa copiar e colar no Gemini sem abrir nada.

**Como entregar:**

1. Imprime os 5 prompts no chat, um abaixo do outro, cada um num bloco \`\`\`text fechado, na ordem DR-1 → DR-2 → DR-3 → DR-4 → Perplexity.
2. Antes de cada bloco, uma linha curta dizendo o que e e onde salvar o output bruto (ex: "**DR-1 — cole no Gemini Deep Research, salve a saida em `bruto/gemini/dr-01-output.md`**").
3. Em paralelo, salva o mesmo conteudo nos arquivos `prompts/dr-N-*.md` e `prompt-perplexity-social-opcional.md` (com path absoluto).
4. Fecha com a instrucao operacional:

> Rode **quatro Deep Researches no Gemini**, **um de cada vez**, na ordem acima. Em cada rodada apos a primeira, **anexe ou cole o relatorio da rodada anterior** no contexto do Deep Research junto com o novo prompt. Salve cada saida em `bruto/gemini/dr-0N-output.md`. Opcional no fim: rode o prompt do Perplexity.

**Nao resumir os prompts no chat.** O bloco copiavel e o entregavel — se voce truncar, o usuario tem que abrir arquivo, e o ponto da skill e zero atrito.

## Templates dos 4 prompts (entregar SEMPRE preenchidos com `[DADOS INTERNOS CONSOLIDADOS]`)

O agente deve **substituir** o placeholder `<<<DADOS INTERNOS CONSOLIDADOS>>>` pelo bloco real consolidado no Passo 3. Em DR-2+, deixe espaco pro usuario colar `<<<OUTPUT DR-ANTERIOR OU RESUMO>>>` quando rodar.

### DR-1 — Cliente, posicionamento digital e regiao

Salvar como `prompts/dr-1-cliente-posicionamento-regiao.md`. Objetivo: entender o cliente; mapear **tudo** que der pra achar e inferir do **posicionamento digital** (fato vs inferencia); **regiao de atuacao** (Brasil, estadual, local, multi-regiao) e **pormenores uteis** do territorio pra marketing (com fonte ou como inferencia explicita).

```text
Deep Research — Rodada 1 de 4: Cliente, presenca digital e regiao de atuacao.

DADOS INTERNOS (nao contradizer sem apontar inconsistencia):
<<<DADOS INTERNOS CONSOLIDADOS>>>

TAREFA:
1) Quem e o cliente e o que parece vender (cruzar digital + dados internos).
2) Posicionamento digital: site, redes, SEO aparente, tom, promessas, provas sociais, gaps.
3) Separar rigorosamente: FATO (fonte publica ou dado interno) / INFERENCIA / HIPOTESE / LACUNA.
4) Regiao de atuacao: escala (Brasil vs regional vs local); evidencias; nuances do territorio **uteis para marketing** (economia, consumo, concorrencia local quando houver fonte — evitar cliches sem fonte).
5) Perguntas obrigatorias para validar com o cliente.
6) Riscos: homonimia de marca, dados desatualizados.

REGRAS: Portugues BR; cite links; nao invente faturamento, precos internos nem performance de anuncios.
```

### DR-2 — Produto, oferta e como funciona o setor

Salvar como `prompts/dr-2-produto-setor.md`. Incluir contexto da DR-1 (usuario cola output ou resumo).

```text
Deep Research — Rodada 2 de 4: Produtos/servicos e dinamica do setor.

CONTEXTO DA RODADA 1 (cole aqui o relatorio completo ou resumo fiel):
<<<OUTPUT DR-1 OU RESUMO>>>

DADOS INTERNOS (repetir o essencial):
<<<DADOS INTERNOS CONSOLIDADOS>>>

TAREFA:
1) Detalhar produtos/servicos que o cliente oferece.
2) Como o SETOR funciona: modelo de negocio tipico, fontes de receita, recorrencia, sazonalidade.
3) O que **normalmente** da margem vs volume no setor (separar padrao de mercado vs especulacao sobre ESTE cliente).
4) Como empresas desse tipo **costumam vender** (canais, bundles, argumentos, servicos anexos).
5) Indicadores e linguagem do ramo.
6) Lacunas so o dono fecha.

REGRAS: PT-BR; separar "padrao do setor" vs "esta marca"; FATO/INFERENCIA/LACUNA; links.
```

### DR-3 — Consumidor (foco total na demanda)

```text
Deep Research — Rodada 3 de 4: Consumidor — panorama completo.

CONTEXTO DAS RODADAS 1 E 2 (cole resumos ou relatorios):
<<<OUTPUT DR-1 E DR-2 OU RESUMOS>>>

DADOS INTERNOS:
<<<DADOS INTERNOS CONSOLIDADOS>>>

TAREFA (prioridade maxima = quem COMPRA):
1) Segmentos, gatilhos de compra, contexto.
2) O que querem; como falam (aspas so com citacao real; senao SINTETIZADO).
3) Reclamacoes, medos, friccoes (com fontes quando possivel).
4) Criterios de escolha e alternativas.
5) Confianca vs rejeicao na categoria.
6) Mapa acionavel de dores/desejos para copy e criativo.
7) Lacunas de voz do consumidor online.

REGRAS: PT-BR; nao confundir persona inventada com evidencia.
```

### DR-4 — Concorrencia

```text
Deep Research — Rodada 4 de 4: Panorama competitivo geral.

CONTEXTO DAS RODADAS ANTERIORES (cole resumos ou relatorios 1–3):
<<<OUTPUT DR-1 A DR-3 OU RESUMOS>>>

DADOS INTERNOS (incluir concorrentes citados pelo cliente):
<<<DADOS INTERNOS CONSOLIDADOS>>>

TAREFA:
1) Concorrentes diretos, indiretos, substitutos relevantes (regiao do cliente; declarar limite se regiao incerta).
2) Promessas, posicionamento, precos publicos quando houver fonte.
3) Presenca digital comparativa (sem afirmar ROAS/CPA).
4) Gaps e oportunidades.
5) O que validar com o cliente sobre concorrencia.

REGRAS: PT-BR; links; FATO/INFERENCIA/LACUNA; homonimia.
```

## Opcional — Perplexity (busca social)

Salvar `prompt-perplexity-social-opcional.md` e dizer ao usuario que e **recomendado depois da DR-3 ou DR-4** pra cruzar reviews/comentarios com links.

```text
Pesquisa com FONTES e LINKS — foco em voz social do consumidor.

Marca/categoria: [NOME / CATEGORIA]
Regiao: [REGIAO ou Brasil]
Concorrentes: [LISTA ou LACUNA]

Quero: reviews, RA, comentarios, foruns em PT-BR; padroes de elogio/raiva; citacoes reais entre aspas ou FRASE SINTETIZADA; hesitacoes ("quase comprei"); concorrentes citados por usuarios; LACUNAS.

Separe FATO COM LINK vs INFERENCIA. Nao invente citacoes.
```

## Apos os quatro DRs — sintese nos markdowns

Quando o usuario voltar com outputs, ajude a distribuir:

| Output | Arquivo de sintese sugerido |
|--------|-----------------------------|
| DR-1 | `01-cliente-posicionamento-digital.md` (+ trechos de regiao se quiser no briefing) |
| DR-2 | `02-mercado-setor-modelo-negocio.md` e `03-produto-servico.md` |
| DR-3 | `06-consumidor.md` |
| DR-4 | `04-concorrentes.md` |
| Perplexity | `06-consumidor.md` (complemento) e `07-fontes-e-evidencias.md` |

Preencha `07-fontes-e-evidencias.md` com links, data, ferramenta, fato vs inferencia vs validar com cliente.

**Panorama de ads (05):** nao exige novo DR; checklist em `references/fontes-pesquisa-social.md` (Meta Ad Library, Google Transparency, TikTok Creative Center) — observacao vs inferencia; sem performance sem dado.

## Consumidor sintetico

> Isso nao e pesquisa real final; e hipotese para testar copy. Basear sempre em evidencias dos DRs.

## Fechamento

Entregue ao usuario, **na conversa**, nesta ordem:

1. **Os 5 prompts em blocos `code` copiaveis** (DR-1, DR-2, DR-3, DR-4, Perplexity), preenchidos com os dados internos reais — ver Passo 4. Isso e obrigatorio: o usuario precisa colar no Gemini sem abrir arquivo.
2. Link pro `00-briefing-pesquisa.md` atualizado com o `[DADOS INTERNOS CONSOLIDADOS]`.
3. Link pros 4 arquivos em `prompts/` e pro `prompt-perplexity-social-opcional.md` salvos.
4. Onde salvar cada output bruto (`bruto/gemini/dr-0N-output.md`, `bruto/perplexity/`).
5. Lacunas criticas pro cliente validar.

Nao entregue plano de campanha final aqui — isso vem depois da KB consolidada.
