# DSL RECIP-E — Restaurante do Chefe Remy

## Descrição Resumida da DSL

**RECIP-E** é uma Linguagem de Domínio Específico (DSL) imperativa, em
português, para descrever receitas culinárias como programas estruturados,
legíveis e validáveis. Ingredientes e utensílios são tratados como **recursos
declarados** e os passos do preparo são **instruções sequenciais** que operam
sobre esses recursos.

A motivação é eliminar a ambiguidade do texto livre (*"uma pitada"*, *"asse
até dourar"*) sem sacrificar legibilidade. Ao formalizar a estrutura de uma
receita, abre-se espaço para:

- **validação estática** (todos os ingredientes usados foram declarados?
  a soma das medidas excede o estoque?),
- **integração com sistemas externos** (apps de cardápio, geração automática
  de listas de compras, cozinhas robóticas),
- **reuso por composição** (sub-receitas — um molho bechamel pode ser
  componente de várias receitas).

A relevância está em ser um caso de DSL com público duplo: o cozinheiro, que
lê e escreve, e o sistema, que processa.

Este projeto implementa o **front-end** da linguagem (lexer → parser →
pretty-print → validação estática) em **Scheme**, executado em Jupyter via
kernel **Calysto Scheme**. Disciplina **MC346 — Paradigmas de Programação**,
UNICAMP.

## Slides

Apresentação final: `<link para slides.pdf>` _(adicionar)_

## Sintaxe da Linguagem

### Estrutura geral

Um programa RECIP-E é composto por até 4 blocos, nesta ordem (todos
opcionais — embora INGREDIENTES e METODOS sejam recomendados):

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

Os blocos são abertos pela palavra-chave e seu conteúdo é **delimitado por
indentação** (4 espaços ou 1 tab). Não há `{}` nem `;`.

### Comentários

```
// comentário de linha
/* comentário
   de bloco */
```

### Bloco RECEITA — metadados

Campos: `NOME` (string), `PORCOES` (número), `NIVEL` (`FACIL|MEDIO|AVANCADO`),
`TEMPO_PREPARO` (número + unidade de tempo).

### Bloco INGREDIENTES — declaração de insumos

```
INGREDIENTES
    2 unidade ovo
    200 ml leite
    10 g sal
    1 unidade "RecheioDeQueijo"   // sub-receita
```

Unidades aceitas: `g kg ml l unidade colher colher_cha xcara pitada a_gosto faca`.

Quando o nome do ingrediente vem entre aspas, ele é tratado como
**sub-receita** — outra receita RECIP-E referenciada pelo nome.

### Bloco UTENSILIOS — ambiente

```
UTENSILIOS
    1 frigideira
    1 garfo
    1 tigela
```

### Bloco METODOS — fluxo de execução

Instruções por linha. Os principais comandos:

#### Comando-verbo

```
ADICIONAR ovo, sal USANDO 5 g EM tigela
MISTURAR ovo, pimenta_do_reino USANDO 2 g EM tigela COM garfo
ASSAR massa EM forno A 180 GRAUS POR 40 MINUTOS
CORTAR cebola EM tabua NO_ESTILO BRUNOISE
FRITAR ovo EM frigideira A FOGO_MEDIO
```

Parâmetros (todos opcionais, podem combinar):

| Parâmetro | Significado |
|---|---|
| `USANDO <qty> <unidade>` | Medida específica neste passo (por argumento) |
| `EM <utensílio>` | Onde a operação acontece |
| `COM <utensílio>` | Ferramenta auxiliar |
| `A <n> GRAUS` ou `A FOGO_*` | Temperatura / nível de chama |
| `POR <n> MINUTOS\|HORAS\|SEGUNDOS` | Duração |
| `ATE <condição>` | Loop até condição (`FIRMAR`, `DOURAR`, etc.) |
| `NO_ESTILO <tipo>` | Técnica de corte (`CUBOS`, `FATIAS`, `BRUNOISE`...) |

#### Atribuição — `ESTE é`

```
MISTURAR ovo, sal EM tigela
ESTE é MISTURA01
```

`ESTE é` nomeia o resultado da **linha imediatamente anterior**. O nome
criado pode ser usado nas instruções seguintes como se fosse um ingrediente.

#### Espera — `AGUARDAR`

```
AGUARDAR 30 MINUTOS
```

#### Condicional — `VERIFICAR SE / SENAO`

```
VERIFICAR SE MISTURA01 ESTA FIRME
    CONTINUAR
SENAO
    ASSAR MISTURA01 EM frigideira POR 2 MINUTOS
```

#### Paralelismo — `AO_MESMO_TEMPO`

```
AO_MESMO_TEMPO
    ADICIONAR oleo EM frigideira
    AGUARDAR 1 MINUTOS
```

Cada linha do bloco indentado é um ramo independente. A validação estática
**alerta** se dois ramos paralelos disputam o mesmo utensílio.

### Verbos nativos

`ADICIONAR`, `MISTURAR`, `BATER`, `TEMPERAR`, `ASSAR`, `FRITAR`, `COZINHAR`,
`REFOGAR`, `GRELHAR`, `CORTAR`, `RALAR`, `ESCORRER`, `COLOCAR`, `DECORAR`.

## Gramática da Linguagem

EBNF estendida. `INDENT` e `DEDENT` são tokens sintéticos emitidos pelo lexer
ao crescer e diminuir o nível de indentação.

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

> **Diferenças vs. especificação original (v1.0):** o grupo optou por
> delimitação por **indentação** em vez de `{ } ;`, e por construções
> `ESTE é`, `USANDO`, `AO_MESMO_TEMPO`, `VERIFICAR SE/SENAO`. Discussão
> completa na seção *Discussão*.

## Notebook

Todo o código (lexer, parser, pretty-printer, validação, driver e exemplos)
está em **[`RECIP-E.ipynb`](./RECIP-E.ipynb)**.

### Como executar

```
pip install calysto-scheme
python -m calysto_scheme install --sys-prefix
jupyter notebook RECIP-E.ipynb
```

No menu: **Kernel → Restart & Run All**.

### Estrutura do notebook

| Seção | Conteúdo |
|---|---|
| 0 | Contrato do token + tabelas léxicas |
| 1 | **Lexer** — `tokeniza` (texto → tokens com INDENT/DEDENT) |
| 2 | **Parser** — `parse-programa` (tokens → AST) + testes P1–P11 |
| 3 | **Pretty-printer + Validação** — `imprime-programa`, `valida` + testes V1–V5 |
| 4 | **Driver `run`** — pipeline ponta-a-ponta |
| 5 | **Exemplos selecionados** (Omelete + variações) |

## Exemplos Selecionados

### Exemplo 1 — Omelete com Queijo (5.1)

Receita canônica que exercita: metadados, ingredientes com sub-receita,
`USANDO` por argumento, `ESTE é`, `AO_MESMO_TEMPO`, `VERIFICAR SE/SENAO` e
parâmetros de controle.

Esperado: AST completa, relatório **`OK — Nenhum aviso`**.

### Exemplo 2 — Receita com erro de estoque (5.2)

Mesma omelete, mas com `USANDO 50 g` de sal num estoque de `20 g`.

Esperado: AST igual; relatório com **1 aviso `OVERUSE`** apontando que o uso
total (50 g) excede o estoque (20 g).

### Exemplo 3 — Receita com conflito de utensílio (5.3)

Bloco `AO_MESMO_TEMPO` em que dois ramos usam `EM frigideira`.

Esperado: AST igual; **1 aviso `UTENSIL_CONFLICT`**.

Os três exemplos rodam na seção 5 do notebook chamando `(run exemplo-N)`.

## Discussão

Três decisões afastaram a implementação da EBNF original:

1. **Indentação no lugar de `{}` e `;`** — reduz ruído visual, aproxima a
   receita da leitura em prosa.
2. **`ESTE é` no lugar de `=`** — evita a leitura matemática; fica próximo
   da forma natural de nomear um preparo.
3. **`AO_MESMO_TEMPO`** — construção nova (não existia na spec) que reflete
   passos simultâneos reais de cozinha e permite a validação de conflito
   de utensílio entre ramos paralelos.

O front-end cobre todos os cinco códigos de validação
(`UNDECLARED_INGREDIENT`, `UNDECLARED_UTENSIL`, `OVERUSE`,
`UTENSIL_CONFLICT`, `UNRESOLVED_SUBRECIPE`). A validação é **estática**:
opera sobre a AST, não simula execução culinária.

### Por que parser próprio em vez de macros Scheme?

RECIP-E é uma DSL **externa** (arquivo `.recipe` autônomo), não uma sintaxe
embutida em Scheme. Macros expandem em tempo de leitura Scheme e não sabem
tokenizar comentários, strings ou indentação. Além disso, palavras como
`ESTE é` (com acento e espaço) e `A 180 GRAUS` não sobrevivem ao leitor de
S-expressions. Por fim, os avisos precisam apontar a linha do erro —
informação que só um parser explícito preserva.

## Conclusão

O projeto entrega um front-end completo (lexer → parser → pretty-print →
validação estática) para uma DSL não trivial em português. Os pontos mais
delicados foram:

- **Indentação no lexer** — resolvida com pilha de níveis emitindo
  `INDENT`/`DEDENT`.
- **`ESTE é`** — resolvida tratando as duas palavras como keywords
  contíguas no parser.
- **Portabilidade Calysto** — `filter` retorna objeto Python e `char>=?`
  não existe; reescrevemos as primitivas necessárias em Scheme puro.

A validação estática mostrou-se o componente de maior valor por linha de
código: é o que separa "ler um arquivo" de "verificar uma receita".

# Trabalhos Futuros

- **Construções da spec não cobertas**: `REPETIR N VEZES`, loops `ATE`
  completos, anotações `@CRITICO` / `@OPCIONAL`, e `EM_PONTO_DE <estado>`.
- **Suporte alternativo à sintaxe `{}` / `;`** — relaxar o lexer para
  aceitar a spec original como entrada compatível.
- **Avaliador / runtime** que simule o passo a passo da receita, incluindo
  consumo real de estoque ao longo da execução (hoje só estática).
- **Geração automática** a partir da AST: lista de compras, checklist de
  preparo, sumário de tempos, *grafo* de dependências de etapas.
- **Plugin de editor** (VSCode) com destaque de sintaxe e os warnings
  inline da validação estática.
- **Mensagens de erro com sugestão** — não só apontar o problema, mas
  sugerir a correção mais próxima (similar a clang).

# Referências Bibliográficas

1. RECIP-E — Especificação de Sintaxe e Gramática, v1.0. MC346, UNICAMP, 2026.
2. *Calysto Scheme* — kernel Jupyter para Scheme.
   https://github.com/Calysto/calysto_scheme
3. Aho, A. V.; Lam, M. S.; Sethi, R.; Ullman, J. D. *Compilers: Principles,
   Techniques, and Tools*. 2ª ed. Pearson, 2006. (Cap. 3 e 4 — análise
   léxica e sintática descendente).
4. *The Python Language Reference* — seção *Lexical Analysis: Indentation*.
   https://docs.python.org/3/reference/lexical_analysis.html (modelo de
   `INDENT`/`DEDENT` que inspirou o lexer).
5. Wirth, N. *Extended Backus-Naur Form (EBNF)*. ISO/IEC 14977, 1996.
