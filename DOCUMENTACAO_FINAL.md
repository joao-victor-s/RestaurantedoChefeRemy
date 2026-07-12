# DSL RECIP-E — Restaurante do Chefe Remy

## Descrição Resumida da DSL

**RECIP-E** é uma Linguagem de Domínio Específico (DSL) imperativa, em
português, para descrever receitas culinárias como programas estruturados,
legíveis e validáveis. Ingredientes e utensílios são **recursos declarados**;
os passos do preparo são **instruções sequenciais** que operam sobre esses
recursos.

**Motivação.** Texto livre de receita (*"uma pitada"*, *"asse até dourar"*) é
ambíguo — impossível de validar por máquina e propenso a interpretação humana
divergente. RECIP-E remove essa ambiguidade sem sacrificar a legibilidade de
quem cozinha.

**Relevância.** É um caso interessante de DSL com **público duplo**: o
cozinheiro (lê/escreve) e o sistema (processa). Ao formalizar a estrutura,
viabiliza:

- **validação estática** (ingredientes usados foram declarados? soma das
  medidas excede o estoque?),
- **integração com sistemas externos** (cardápios, geração de lista de
  compras, cozinhas robóticas),
- **reuso por composição** — sub-receitas (um bechamel referenciado por
  várias receitas).

Este projeto implementa o **front-end** da linguagem (lexer → parser →
pretty-print → validação estática) em **Scheme**, executado em Jupyter via
kernel **Calysto Scheme**. Disciplina **MC346 — Paradigmas de Programação**,
UNICAMP.

## Slides

Apresentação final: [`RECIP-E_Apresentacao_Final.pptx`](./RECIP-E_Apresentacao_Final.pptx)

## Sintaxe da Linguagem

### Estrutura geral

Programa RECIP-E = até 4 blocos, nesta ordem (todos opcionais; `INGREDIENTES`
e `METODOS` são recomendados):

```
RECEITA
    NOME: "..."
    PORCOES: <n>
    NIVEL: <FACIL|MEDIO|AVANCADO>
    TEMPO_PREPARO: <n> MINUTOS

INGREDIENTES
    <qty> <unidade> <nome>
    ...

UTENSILIOS
    <qty> <nome>
    ...

METODOS
    <instrucao>
    ...
```

Blocos abrem pela palavra-chave; conteúdo **delimitado por indentação**
(4 espaços ou 1 tab). Sem `{}`, sem `;`.

### Comentários

```
// comentário de linha
/* comentário
   de bloco */
```

### RECEITA — metadados

`NOME` (string), `PORCOES` (número), `NIVEL` (`FACIL|MEDIO|AVANCADO`),
`TEMPO_PREPARO` (número + unidade de tempo).

### INGREDIENTES — insumos

```
INGREDIENTES
    2 unidade ovo
    200 ml leite
    10 g sal
    1 unidade "RecheioDeQueijo"   // sub-receita
```

Unidades aceitas: `g kg ml l unidade colher colher_cha xcara pitada a_gosto faca`.

Nome entre aspas ⇒ **sub-receita** (outra receita RECIP-E referenciada por
nome).

### UTENSILIOS — ambiente

```
UTENSILIOS
    1 frigideira
    1 garfo
    1 tigela
```

### METODOS — fluxo de execução

**Comando-verbo:**

```
ADICIONAR ovo, sal USANDO 5 g EM tigela
MISTURAR ovo, pimenta_do_reino USANDO 2 g EM tigela COM garfo
ASSAR massa EM forno A 180 GRAUS POR 40 MINUTOS
CORTAR cebola EM tabua NO_ESTILO BRUNOISE
FRITAR ovo EM frigideira A FOGO_MEDIO
```

| Parâmetro | Significado |
|---|---|
| `USANDO <qty> <unidade>` | Medida específica no passo (por argumento) |
| `EM <utensílio>` | Onde a operação acontece |
| `COM <utensílio>` | Ferramenta auxiliar |
| `A <n> GRAUS` / `A FOGO_*` | Temperatura / nível de chama |
| `POR <n> MINUTOS\|HORAS\|SEGUNDOS` | Duração |
| `ATE <condição>` | Loop até condição (`FIRMAR`, `DOURAR`…) |
| `NO_ESTILO <tipo>` | Técnica de corte (`CUBOS`, `FATIAS`, `BRUNOISE`…) |

**Atribuição — `ESTE é`:**

```
MISTURAR ovo, sal EM tigela
ESTE é MISTURA01
```

Nomeia o resultado da linha imediatamente anterior; o nome vira ingrediente
disponível para as instruções seguintes.

**Espera — `AGUARDAR`:**

```
AGUARDAR 30 MINUTOS
```

**Condicional — `VERIFICAR SE / SENAO`:**

```
VERIFICAR SE MISTURA01 ESTA FIRME
    CONTINUAR
SENAO
    ASSAR MISTURA01 EM frigideira POR 2 MINUTOS
```

**Paralelismo — `AO_MESMO_TEMPO`:**

```
AO_MESMO_TEMPO
    ADICIONAR oleo EM frigideira
    AGUARDAR 1 MINUTOS
```

Cada linha do bloco indentado é um ramo independente. A validação estática
**alerta** se dois ramos disputam o mesmo utensílio.

### Verbos nativos

`ADICIONAR`, `MISTURAR`, `BATER`, `TEMPERAR`, `ASSAR`, `FRITAR`, `COZINHAR`,
`REFOGAR`, `GRELHAR`, `CORTAR`, `RALAR`, `ESCORRER`, `COLOCAR`, `DECORAR`.

## Gramática da Linguagem

EBNF estendida. `INDENT` e `DEDENT` são tokens sintéticos emitidos pelo lexer
com base no nível de indentação (modelo Python).

```
programa         ::= bloco_receita? bloco_ingredientes? bloco_utensilios? bloco_metodos?

bloco_receita    ::= 'RECEITA' NEWLINE INDENT campo_meta+ DEDENT
campo_meta       ::= IDENT ':' (STRING | NUMBER | IDENT) NEWLINE

bloco_ingredientes ::= 'INGREDIENTES' NEWLINE INDENT decl_ingrediente+ DEDENT
decl_ingrediente   ::= NUMBER unidade (IDENT | STRING) NEWLINE
unidade            ::= 'g' | 'kg' | 'ml' | 'l' | 'unidade'
                     | 'colher' | 'colher_cha' | 'xcara'
                     | 'pitada' | 'a_gosto' | 'faca'

bloco_utensilios ::= 'UTENSILIOS' NEWLINE INDENT decl_utensilio+ DEDENT
decl_utensilio   ::= NUMBER IDENT NEWLINE

bloco_metodos    ::= 'METODOS' NEWLINE INDENT instrucao+ DEDENT
instrucao        ::= cmd_verbo | atribuicao | aguardar
                   | verificar | ao_mesmo_tempo | 'CONTINUAR'

cmd_verbo        ::= VERBO lista_arg ('EM' IDENT)? ('COM' IDENT)?
                            ('A' (NUMBER 'GRAUS' | nivel_fogo))?
                            ('POR' NUMBER unidade_tempo)?
                            ('ATE' IDENT)?
                            ('NO_ESTILO' IDENT)?
                            NEWLINE
lista_arg        ::= arg (',' arg)*
arg              ::= IDENT ('USANDO' NUMBER unidade)?

atribuicao       ::= 'ESTE' 'é' IDENT NEWLINE
aguardar         ::= 'AGUARDAR' NUMBER unidade_tempo NEWLINE

verificar        ::= 'VERIFICAR' 'SE' IDENT 'ESTA' IDENT NEWLINE
                       INDENT instrucao+ DEDENT
                     ('SENAO' NEWLINE INDENT instrucao+ DEDENT)?

ao_mesmo_tempo   ::= 'AO_MESMO_TEMPO' NEWLINE
                       INDENT instrucao+ DEDENT

VERBO            ::= 'ADICIONAR' | 'MISTURAR' | 'BATER' | 'TEMPERAR'
                   | 'ASSAR' | 'FRITAR' | 'COZINHAR' | 'REFOGAR'
                   | 'GRELHAR' | 'CORTAR' | 'RALAR' | 'ESCORRER'
                   | 'COLOCAR' | 'DECORAR'

unidade_tempo    ::= 'MINUTOS' | 'HORAS' | 'SEGUNDOS'
nivel_fogo       ::= 'FOGO_ALTO' | 'FOGO_MEDIO' | 'FOGO_BAIXO'

STRING           ::= '"' <qualquer caractere exceto '"'>* '"'
NUMBER           ::= [0-9]+ ('.' [0-9]+)?
IDENT            ::= [a-zA-Z_] [a-zA-Z0-9_]*
```

> **Diferenças vs. spec original (v1.0):** o grupo trocou `{ } ;` por
> **indentação** e introduziu `ESTE é`, `USANDO`, `AO_MESMO_TEMPO`,
> `VERIFICAR SE/SENAO`. Discussão completa na seção *Discussão*.

## Notebook

Todo o código (lexer, parser, pretty-printer, validação, driver e exemplos)
está em **[`RECIP-E.ipynb`](./RECIP-E.ipynb)** — arquivo único, kernel
Calysto Scheme.

### Como executar

```
pip install calysto-scheme
python -m calysto_scheme install --sys-prefix
jupyter notebook RECIP-E.ipynb
```

No menu: **Kernel → Restart & Run All**.

### Estrutura do notebook (6 seções, 64 células)

| Seção | Conteúdo |
|---|---|
| 0 | Contrato do token `(tipo lexema linha)` + tabelas léxicas |
| 1 | **Lexer** — `tokeniza` (texto → tokens com `INDENT`/`DEDENT`) |
| 2 | **Parser** — `parse-programa` (tokens → AST) + testes P1–P12 |
| 3 | **Pretty-printer + Validação** — `imprime-programa`, `valida` + testes V1–V6 |
| 4 | **Driver `run`** — pipeline ponta-a-ponta |
| 5 | **Exemplos selecionados** (Omelete + variações) |
| 6 | Referência da gramática (EBNF) |

## Exemplos Selecionados

Os três exemplos rodam via `(run exemplo-N)` na seção 5 do notebook.

### Exemplo 1 — Omelete com Queijo (5.1)

Receita canônica exercitando: metadados, ingredientes com sub-receita,
`USANDO` por argumento, `ESTE é`, `AO_MESMO_TEMPO`, `VERIFICAR SE/SENAO` e
parâmetros de controle.

**Esperado:** AST completa + relatório **`OK — Nenhum aviso`**.

### Exemplo 2 — Erro de estoque (5.2)

Mesma omelete, mas com `USANDO 50 g` de sal para um estoque de `20 g`.

**Esperado:** AST igual + **1 aviso `OVERUSE`** apontando que o uso (50 g)
excede o estoque (20 g).

### Exemplo 3 — Conflito de utensílio (5.3)

Bloco `AO_MESMO_TEMPO` com dois ramos usando `EM frigideira`.

**Esperado:** AST igual + **1 aviso `UTENSIL_CONFLICT`**.

### Códigos de validação cobertos

| Código | Severidade | Disparador |
|---|---|---|
| `UNDECLARED_INGREDIENT` | erro | argumento de verb-cmd não declarado nem criado por `ESTE é` |
| `UNDECLARED_UTENSIL` | erro | `EM` / `COM` referindo utensílio não declarado |
| `OVERUSE` | aviso | soma dos `USANDO` de um ingrediente excede o estoque declarado |
| `UTENSIL_CONFLICT` | aviso | dois ramos de `AO_MESMO_TEMPO` disputam o mesmo utensílio |
| `UNRESOLVED_SUBRECIPE` | aviso | sub-receita declarada (com aspas) e nunca usada |

## Discussão

Três decisões afastaram a implementação da EBNF original — e essa foi a parte
mais interessante do projeto:

1. **Indentação em vez de `{}` e `;`** — reduz ruído visual, aproxima a
   receita da leitura em prosa. Custo: o lexer precisou de uma pilha de
   níveis emitindo `INDENT`/`DEDENT` (modelo Python).
2. **`ESTE é` no lugar de `=`** — evita a leitura matemática e mantém a
   naturalidade da linguagem culinária (*"essa mistura é a MISTURA01"*).
   Custo: o parser trata as duas palavras como par-keyword contíguo.
3. **`AO_MESMO_TEMPO`** — construção nova (não existia na spec) que reflete
   passos simultâneos reais de cozinha e permite a análise
   `UTENSIL_CONFLICT` entre ramos paralelos.

O front-end cobre os cinco códigos de validação. **A validação é estática** —
opera sobre a AST, não simula a execução culinária; portanto o `OVERUSE`,
por exemplo, é a soma sintática dos `USANDO` de um ingrediente, sem noção
temporal de que ele poderia ser reposto no meio do preparo.

### Por que parser próprio em vez de macros Scheme?

RECIP-E é uma DSL **externa** (arquivo `.recipe` autônomo), não uma sintaxe
embutida em Scheme. Macros expandem no leitor Scheme e não sabem tokenizar
comentários, strings ou indentação. Palavras como `ESTE é` (com acento e
espaço) e `A 180 GRAUS` não sobrevivem ao leitor de S-expressions. Além
disso, os avisos precisam apontar a **linha** do erro — informação que só
um parser explícito preserva.

## Conclusão

Entregamos um front-end completo (lexer → parser → pretty-print → validação
estática) para uma DSL não trivial em português, tudo em Scheme puro num
único notebook.

**Principais desafios:**

- **Indentação no lexer** — resolvida com pilha de níveis emitindo
  `INDENT`/`DEDENT`.
- **`ESTE é`** — resolvida tratando as duas palavras como keywords
  contíguas no parser.
- **Portabilidade Calysto Scheme** — `filter` retorna objeto Python e
  `char>=?` não existe; reescrevemos as primitivas necessárias em Scheme
  puro (`my-filter`, `intersect`, `gera-pares`) na seção 3.2 do notebook.

**Lições aprendidas:**

- A validação estática é o componente de **maior valor por linha de
  código** — é o que separa "ler um arquivo" de "verificar uma receita".
- Escolher indentação como delimitador triplicou a complexidade do lexer
  mas divide pela metade o volume do código-fonte da DSL.
- Testar em pequenas células isoladas (P1–P12, V1–V6) foi essencial para
  diagnosticar bugs sem ter que rodar o notebook inteiro.

# Trabalhos Futuros

- **Construções da spec ainda não cobertas:** `REPETIR N VEZES`, loops
  `ATE` completos, anotações `@CRITICO` / `@OPCIONAL`,
  `EM_PONTO_DE <estado>`.
- **Sintaxe alternativa `{}` / `;`** — relaxar o lexer para aceitar a spec
  original como entrada compatível.
- **Avaliador / runtime** que simule passo a passo a receita, com consumo
  real de estoque ao longo da execução (hoje só estático).
- **Geração automática a partir da AST:** lista de compras, checklist de
  preparo, sumário de tempos, *grafo* de dependências de etapas.
- **Plugin de editor** (VSCode) com destaque de sintaxe e warnings inline
  da validação estática.
- **Mensagens de erro com sugestão** — apontar o problema *e* propor a
  correção mais próxima (estilo clang).
- **Composição real de sub-receitas** — hoje `UNRESOLVED_SUBRECIPE` só
  marca as declaradas e não usadas; poderia expandir a sub-receita inline
  na AST.

# Referências Bibliográficas

1. RECIP-E — Especificação de Sintaxe e Gramática, v1.0. MC346, UNICAMP, 2026.
2. *Calysto Scheme* — kernel Jupyter para Scheme.
   https://github.com/Calysto/calysto_scheme
3. Aho, A. V.; Lam, M. S.; Sethi, R.; Ullman, J. D. *Compilers: Principles,
   Techniques, and Tools*. 2ª ed. Pearson, 2006 (Caps. 3 e 4 — análise
   léxica e sintática descendente).
4. *The Python Language Reference* — seção *Lexical Analysis: Indentation*.
   https://docs.python.org/3/reference/lexical_analysis.html (modelo de
   `INDENT`/`DEDENT` que inspirou o lexer).
5. Wirth, N. *Extended Backus-Naur Form (EBNF)*. ISO/IEC 14977, 1996.
