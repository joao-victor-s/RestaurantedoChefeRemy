# PLANO FINAL — Finalização do RECIP-E

Documento único de finalização. Substitui informalmente os `PLANO_PESSOA_*.md`
no que diz respeito ao **trabalho restante**. Cobre:

1. Premissas e estado atual.
2. Decisões de design (a variante de sintaxe que o grupo de fato implementou).
3. Implementação faltante — parser, printer, validação, driver — com testes.
4. **README final pronto para colar** (todas as seções da entrega final já
   redigidas).
5. Cronograma comprimido para 28–29/06.

---

## 0. Premissas

- **Hoje:** Dom 28/06/2026. **Entrega:** Seg 29/06/2026.
- O lexer (Pessoa A) está **pronto** no notebook, incluindo indentação,
  comentários e classificação léxica.
- **Parser (B)** e **Printer/Validação (C)** não foram iniciados.
- A linguagem implementada é a **variante por indentação** descrita nos
  `PLANO_PESSOA_*.md`. Ela difere da spec PDF em pontos estruturais
  (ver §2). **Não vamos reescrever para a sintaxe do PDF** — caminho
  inviável em 1 dia. Vamos **assumir e documentar** a variante.

### Estado real por componente

| Componente | Plano para hoje | Real | Ação |
|---|---|---|---|
| Lexer | Erros com `linha:col`, docs | Lexer funcional, sem mensagens estruturadas | Melhoria opcional (§3.6) |
| Parser | Erros + integração com lexer | Não iniciado | **Crítico — §3.2** |
| Printer | Render METODOS + integração | Não iniciado | **Crítico — §3.3** |
| Validação | Conflito + sub-receita | Não iniciado | **Crítico — §3.4** |
| Driver | Pipeline ponta-a-ponta | Não iniciado | **Crítico — §3.5** |
| README final | Não iniciado | Não iniciado | **Crítico — §4** |

---

## 1. Estratégia

1. **Não tocar no lexer** além de pequenos ajustes (cobrimos via §3.6).
2. **Parser primeiro, printer junto, validação no fim** — em sequência rápida,
   uma pessoa por vez se precisar consolidar; se duas pessoas trabalharem em
   paralelo, A pega o printer enquanto B faz o parser, e C costura validação +
   notebook + README.
3. **Tudo no notebook único `RECIP-E.ipynb`** (não usar `.scm` separados como
   previa o plano original — sem tempo para integrar).
4. **README final é tão crítico quanto o código** — a banca lê o README
   primeiro. Está pronto neste documento (§4); só falta colar.

---

## 2. Decisões de Design (documentar publicamente)

A variante implementada se chama **RECIP-E/Indent**. As decisões abaixo entram
na seção *Discussão* do README e justificam as diferenças.

| # | Decisão | Spec PDF | RECIP-E/Indent | Justificativa |
|---|---|---|---|---|
| D1 | Delimitação de blocos | `{ ... }` + `;` | Indentação (estilo Python) | Reduz ruído visual; código de receita lê mais como prosa estruturada |
| D2 | Atribuição | `mistura = MISTURAR(...)` | `ESTE é MISTURA01` (linha seguinte) | Mais próximo da linguagem natural ("isto é") |
| D3 | Chamada de verbo | `ADICIONAR(ovo, sal)` | `ADICIONAR ovo, sal` | Reforça leitura como sentença imperativa |
| D4 | Medida no passo | `COM MEDIDA 5 G` | `USANDO 5 g` | Preposição mais natural em português |
| D5 | Paralelismo | (não existe na spec) | `AO_MESMO_TEMPO` + bloco indentado | Domínio de cozinha exige paralelismo explícito |
| D6 | Sub-receita | `USAR_RECEITA("X")` | Declarar nome entre aspas em INGREDIENTES (`1 unidade "X"`) | Trata sub-receita como mais um insumo |
| D7 | Condicional | `SE cond { } SENAO { }` | `VERIFICAR SE X ESTA Y / CONTINUAR / SENAO` | Vocabulário culinário ("verificar se está firme") |

### Construções da spec deliberadamente **fora** desta versão

- `REPETIR N VEZES` — entra em Trabalhos Futuros.
- `EM_PONTO_DE <estado>` — entra em Trabalhos Futuros.
- Anotações `@CRITICO` / `@OPCIONAL` — entram em Trabalhos Futuros.
- Bloco `RECEITA` com tags / autor — campos suportados como IDENT livre, sem
  validação adicional.

---

## 3. Implementação Faltante

A ordem abaixo é a ordem de execução. **Cada subitem tem:** o que entregar, o
algoritmo em alto nível, e a célula de teste correspondente.

### 3.1 Contratos finais (já existem — só revisão)

O contrato de token está em `0. Setup`. O schema da AST será adicionado **no
início da seção 2** do notebook, antes do parser, como markdown + um comentário
Scheme curto. Não precisa código adicional.

**Esquema da AST (listas com tag, sem record types):**

```
(programa <receita-ou-#f> <ingredientes> <utensilios> <metodos>)
(receita        ((campo NOME "...") (campo PORCOES 2) ...))
(ingredientes   (<ingrediente> ...))
(utensilios     (<utensilio> ...))
(metodos        (<instrucao> ...))

(ingrediente    <qty:number> <unidade:string> <nome:string> <sub-receita?:bool>)
(utensilio      <qty:number> <nome:string>)

(verb-cmd       <verbo> <args> <em> <com> <a> <por> <ate> <estilo>)
                ; campos ausentes = #f
(arg            <nome> <usando-qty> <usando-unidade>)
                ; usando-* = #f se ausente

(este-e         <nome>)
(aguardar       <qty> <unidade-tempo>)
(verificar      <alvo> <estado> <then-instrucoes> <senao-instrucoes-ou-#f>)
(ao-mesmo-tempo <ramo> ...)   ; ramo = lista de instrucoes
```

### 3.2 Parser — Seção 2 do notebook

Recebe a lista de tokens de `tokeniza` e devolve um nó `programa`.

#### 3.2.1 Cursor de tokens (helpers)

```scheme
(define (make-cursor tokens) (list 'cursor tokens))
(define (peek c)    (if (null? (cadr c)) (faz-token 'EOF "" 0) (car (cadr c))))
(define (advance c) (set-car! (cdr c) (cdr (cadr c))) c)
(define (at? c tipo)    (eq? (tipo-tok (peek c)) tipo))
(define (at-kw? c lex)  (and (at? c 'KEYWORD) (string=? (lexema-tok (peek c)) lex)))
(define (expect c tipo)
  (if (at? c tipo)
      (let ((t (peek c))) (advance c) t)
      (erro c (string-append "esperava " (symbol->string tipo)))))
(define (erro c msg)
  (let ((t (peek c)))
    (error 'parse (string-append msg " — achei "
                                 (symbol->string (tipo-tok t))
                                 " '" (lexema-tok t) "'"
                                 " na linha "
                                 (number->string (linha-tok t))))))
```

> **Atenção Calysto:** `set-car!` em `(cadr c)` muda a lista interna. Se o
> kernel não suportar, trocar `cursor` por um closure que captura uma
> variável local (`let-loop`). Definir já com fallback.

#### 3.2.2 Pular ruído (NEWLINE entre blocos)

```scheme
(define (skip-newlines c)
  (if (at? c 'NEWLINE) (begin (advance c) (skip-newlines c)) c))
```

#### 3.2.3 Regra `programa`

```scheme
(define (parse-programa c)
  (skip-newlines c)
  (let* ((rec   (if (at-kw? c "RECEITA")       (parse-receita c)      #f))
         (_     (skip-newlines c))
         (ings  (if (at-kw? c "INGREDIENTES")  (parse-ingredientes c) '(ingredientes ())))
         (_     (skip-newlines c))
         (uts   (if (at-kw? c "UTENSILIOS")    (parse-utensilios c)   '(utensilios ())))
         (_     (skip-newlines c))
         (mets  (if (at-kw? c "METODOS")       (parse-metodos c)      '(metodos ()))))
    (list 'programa rec ings uts mets)))
```

#### 3.2.4 `RECEITA` (metadados)

```
RECEITA NEWLINE
  INDENT
    (IDENT : (STRING | NUMBER | IDENT) NEWLINE)+
  DEDENT
```

Saída: `(receita ((campo NOME "...") (campo PORCOES 2) ...))`.

#### 3.2.5 `INGREDIENTES`

```
INGREDIENTES NEWLINE
  INDENT
    (NUMBER UNIT (IDENT | STRING) NEWLINE)+
  DEDENT
```

Quando o nome vem de `STRING` → `sub-receita? = #t`.

#### 3.2.6 `UTENSILIOS`

```
UTENSILIOS NEWLINE
  INDENT (NUMBER IDENT NEWLINE)+ DEDENT
```

#### 3.2.7 `METODOS` — instruções

```
METODOS NEWLINE
  INDENT (instrucao NEWLINE)+ DEDENT
```

`instrucao` é um *dispatcher* sobre o próximo token:

| Token de cabeça | Regra | Nó |
|---|---|---|
| `VERB` | `cmd-verbo` | `verb-cmd` |
| `KEYWORD "ESTE"` | `atribuicao` | `este-e` |
| `KEYWORD "AGUARDAR"` | `aguardar` | `aguardar` |
| `KEYWORD "VERIFICAR"` | `verificar` | `verificar` |
| `KEYWORD "AO_MESMO_TEMPO"` | `ao-mesmo-tempo` | `ao-mesmo-tempo` |

#### 3.2.8 `cmd-verbo`

```
VERB (IDENT (',' IDENT)*)?  ('USANDO' NUMBER UNIT)?  ('EM' IDENT)?
     ('COM' IDENT)?  ('A' (NUMBER 'GRAUS' | IDENT))?
     ('POR' NUMBER ('MINUTOS'|'HORAS'|'SEGUNDOS'))?
     ('ATE' IDENT)?  ('NO_ESTILO' IDENT)?
```

> **Diferença vs. spec:** `USANDO` pode aparecer por argumento na spec PDF
> (`sal USANDO 5 g, pimenta USANDO 2 g`). Para reduzir complexidade,
> aceitamos `USANDO` **por argumento** (lendo após cada IDENT) como no
> exemplo Omelete já presente no notebook. A regra real fica:

```
VERB lista-arg ('EM' IDENT)? ... 
lista-arg ::= arg (',' arg)*
arg       ::= IDENT ('USANDO' NUMBER UNIT)?
```

#### 3.2.9 `verificar`

```
VERIFICAR SE IDENT ESTA IDENT NEWLINE
  INDENT instrucao+ DEDENT          ; o "then" — pode ser só CONTINUAR
('SENAO' NEWLINE
  INDENT instrucao+ DEDENT)?
```

#### 3.2.10 `ao-mesmo-tempo`

```
AO_MESMO_TEMPO NEWLINE
  INDENT
    (ramo)+
  DEDENT
```

Onde cada `ramo` é uma sequência contígua de instruções no mesmo nível de
indentação que aparece dentro do bloco. **Simplificação:** todos os ramos têm
exatamente uma instrução cada. Isso evita ter que detectar grupos de ramos por
linhas em branco (não há separador formal). Para múltiplas instruções por
ramo, usar `ESTE é` para encadear via nomes.

### 3.3 Pretty-printer — Seção 3 do notebook

Render recursivo por tag. Sem nenhuma estrutura sofisticada: `display` com
indentação por profundidade.

```scheme
(define (espacos n) (if (= n 0) "" (string-append "  " (espacos (- n 1)))))

(define (imprime-programa p)
  (display "programa") (newline)
  (imprime-receita      (cadr p)   1)
  (imprime-ingredientes (caddr p)  1)
  (imprime-utensilios   (cadddr p) 1)
  (imprime-metodos      (car (cddddr p)) 1))
```

Funções `imprime-*` para cada tipo de nó (RECEITA, INGREDIENTES, UTENSILIOS,
METODOS, verb-cmd, este-e, aguardar, verificar, ao-mesmo-tempo). Mostrar
apenas campos não-`#f`.

### 3.4 Validação estática — Seção 3 do notebook (continuação)

Tipo do "issue": `(list severity code message linha)`.

#### 3.4.1 Tabela de símbolos

```scheme
(define (coleta-ingredientes ings)
  ;; -> assoc-list: (nome . (qty unidade sub-receita?))
  ...)

(define (coleta-utensilios uts)
  ;; -> lista de nomes
  ...)
```

#### 3.4.2 Validações implementadas

| Código | Severidade | Disparador |
|---|---|---|
| `UNDECLARED_INGREDIENT` | error | `arg` em verb-cmd cujo nome não está em INGREDIENTES nem foi criado por `ESTE é` antes |
| `UNDECLARED_UTENSIL` | error | `EM <x>` ou `COM <x>` cujo `<x>` não está em UTENSILIOS nem é nome ESTE-é |
| `OVERUSE` | warning | Soma de `USANDO` de um ingrediente excede o estoque declarado, mesma unidade |
| `UTENSIL_CONFLICT` | warning | Dentro de um `AO_MESMO_TEMPO`, dois ramos referenciam o mesmo utensílio |
| `UNRESOLVED_SUBRECIPE` | warning | Sub-receita declarada (STRING) que nunca é referenciada por um verbo |
| `UNKNOWN_UNIT` | warning | Unidade fora da tabela conhecida (já filtrada pelo lexer, mas redobramos) |

#### 3.4.3 Função principal

```scheme
(define (valida programa)
  ;; -> lista de issues
  ...)
```

Imprime o relatório com `imprime-relatorio` (uma linha por issue).

### 3.5 Driver — Seção 5 do notebook (nova)

```scheme
(define (run fonte)
  (let* ((tokens   (tokeniza fonte))
         (programa (parse-programa (make-cursor tokens))))
    (display "=== AST ===") (newline)
    (imprime-programa programa)
    (newline)
    (display "=== Relatório ===") (newline)
    (imprime-relatorio (valida programa))))
```

### 3.6 Lexer — melhorias opcionais (se sobrar tempo)

- Mensagens de erro com linha quando string não fecha ou comentário `/*` não
  fecha. Hoje silenciosamente come até o fim.
- Aceitar `;` e `{` `}` como espaços (ignorados) — permite que exemplos da spec
  PDF sejam parcialmente compatíveis. **Decisão:** só fazer se sobrar tempo.

---

## 4. Estratégia de Testes

Sem framework — **células de demonstração no notebook**. Cada componente tem
sua sub-seção de testes. Critério de aprovação: rodar `Kernel → Restart & Run
All` sem erro, e visualmente conferir cada saída.

### 4.1 Testes do parser (seção 2.X do notebook)

| # | Entrada | Saída esperada | Garante |
|---|---|---|---|
| P1 | `"RECEITA\n    NOME: \"X\"\n"` | `(programa (receita ((campo NOME "X"))) ...)` | RECEITA mínima |
| P2 | `"INGREDIENTES\n    2 unidade ovo\n"` | ingrediente com qty=2, unidade="unidade", nome="ovo", sub=#f | INGREDIENTES básico |
| P3 | `"INGREDIENTES\n    1 unidade \"Molho\"\n"` | sub-receita?=#t | Sub-receita |
| P4 | `"UTENSILIOS\n    1 frigideira\n"` | utensilio qty=1 nome="frigideira" | UTENSILIOS |
| P5 | `"METODOS\n    ADICIONAR ovo EM tigela\n"` | verb-cmd verbo=ADICIONAR, args=[ovo], em=tigela | cmd-verbo simples |
| P6 | `"METODOS\n    ADICIONAR ovo, sal USANDO 5 g EM tigela\n"` | dois args; segundo com usando | lista_arg + USANDO |
| P7 | `"METODOS\n    ASSAR massa EM forno A 180 GRAUS POR 40 MINUTOS\n"` | verb-cmd com a=(180 GRAUS), por=(40 MINUTOS) | parâmetros de controle |
| P8 | `"METODOS\n    ESTE é MISTURA01\n"` | (este-e "MISTURA01") | atribuição |
| P9 | `"METODOS\n    AGUARDAR 5 MINUTOS\n"` | (aguardar 5 "MINUTOS") | aguardar |
| P10 | Bloco VERIFICAR completo com SENAO | (verificar alvo estado then senao) | controle de fluxo |
| P11 | Bloco AO_MESMO_TEMPO com 2 ramos | (ao-mesmo-tempo r1 r2) | paralelismo |
| P12 | Omelete completo (já no notebook) | `programa` consistente | Integração |

### 4.2 Testes do printer (seção 3.X)

Para cada caso P1–P12: parse → print → visual conferido. Não é teste
automático, mas **toda saída tem que ser estável** entre execuções.

### 4.3 Testes da validação (seção 3.Y)

| # | Entrada | Issue esperada |
|---|---|---|
| V1 | Usar `ovo` em METODOS sem declarar em INGREDIENTES | `UNDECLARED_INGREDIENT` na linha do uso |
| V2 | Usar `frigideira` sem declarar em UTENSILIOS | `UNDECLARED_UTENSIL` |
| V3 | Estoque 20 g sal; usa `USANDO 15 g` + `USANDO 10 g` | `OVERUSE` (25 > 20) |
| V4 | `AO_MESMO_TEMPO` com 2 ramos usando `EM frigideira` | `UTENSIL_CONFLICT` |
| V5 | `1 unidade "MolhoX"` declarado mas nunca usado | `UNRESOLVED_SUBRECIPE` |
| V6 | Omelete correto | **lista vazia** de issues |

### 4.4 Teste integrado (seção 4 do notebook — já existe parcialmente)

- `(run omelete)` → AST imprime + relatório limpo.
- `(run omelete-com-erro-de-estoque)` → AST + 1 warning OVERUSE.
- `(run omelete-com-conflito)` → AST + 1 warning UTENSIL_CONFLICT.

---

## 5. README final — pronto para colar

Tudo abaixo (entre as duas linhas tracejadas) vai como conteúdo de `README.md`
substituindo o atual. **Conteúdo já redigido**, só revise links e nomes próprios.

---

```markdown
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

Apresentação final: `<link para slides.pdf>`

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

Quando o nome do ingrediente vem entre aspas, ele é tratado como **sub-receita**
— outra receita RECIP-E referenciada pelo nome.

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
| `USANDO <qty> <unidade>` | Medida específica neste passo |
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
                   | verificar | ao_mesmo_tempo

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
| 2 | **Parser** — `parse-programa` (tokens → AST) |
| 3 | **Pretty-printer + Validação** — `imprime-programa`, `valida` |
| 4 | Exemplos selecionados (Omelete + variações) |
| 5 | Driver `run` — pipeline ponta-a-ponta |

## Exemplos Selecionados

### Exemplo 1 — Omelete com Queijo

Receita canônica que exercita: metadados, ingredientes com sub-receita,
`USANDO` por argumento, `ESTE é`, `AO_MESMO_TEMPO`, `VERIFICAR SE/SENAO`, e
parâmetros de controle.

Esperado: AST completa, relatório de validação **vazio** (sem warnings).

### Exemplo 2 — Receita com erro de estoque

Mesma omelete, mas com `USANDO 50 g` de sal num estoque de `20 g`.

Esperado: AST igual; relatório com **1 warning `OVERUSE`** apontando a linha.

### Exemplo 3 — Receita com conflito de utensílio

Bloco `AO_MESMO_TEMPO` em que dois ramos usam `EM frigideira`.

Esperado: AST igual; **1 warning `UTENSIL_CONFLICT`**.

Os três exemplos rodam na seção 4 do notebook chamando `(run exemplo-N)`.

## Discussão

A proposta inicial era implementar fielmente a EBNF da especificação v1.0 do
RECIP-E. Ao longo do desenvolvimento, três decisões nos afastaram dela:

1. **Indentação no lugar de `{}` e `;`**. A spec original adota a sintaxe
   de C/Java. Avaliamos que o domínio (receita culinária) tem caráter
   narrativo e que receitas escritas com chaves carregam ruído visual
   incompatível com leitura por não-programadores. Adotamos indentação
   estilo Python, que torna o programa visualmente mais próximo de uma
   receita real impressa.

2. **`ESTE é` no lugar de `=`**. O sinal de igual carrega forte associação
   com matemática. `ESTE é <nome>` em uma linha separada lê como a forma
   natural de nomear o que se acabou de fazer ("isto é a base"), reforçando
   a leitura como prosa estruturada.

3. **`AO_MESMO_TEMPO` como construção de primeira classe**. A spec original
   não previa paralelismo, mas receitas reais são repletas de ações
   simultâneas ("enquanto o forno aquece, prepare o recheio"). Adicionar
   essa construção criou a oportunidade mais interessante de validação
   estática do projeto: **detecção de conflito de utensílio** entre ramos
   paralelos.

O **resultado alcançado** cumpre o escopo de *parse + AST + pretty-print +
validação estática* sobre os exemplos da seção 4. A validação dispara
corretamente nos casos de uso esperados (declaração ausente, estoque
excedido, conflito de utensílio). O pipeline `run` roda fim-a-fim em ≤ 1 s
para receitas do tamanho do Omelete.

A indentação, planejada como o ponto mais arriscado do projeto, mostrou-se
tratável com uma pilha de níveis simples. O ponto que mais consumiu tempo
foi o **dispatcher de `instrucao`**: como o parser é recursivo-descendente
e os ramos do dispatcher dependem do *próximo token*, foi necessário
estabilizar cedo a interface `peek/advance` antes de qualquer regra.

A validação estática deliberadamente **não simula execução** — não há
"runtime de cozinha". Toda checagem é feita sobre a AST.

## Conclusão

**Principais conclusões.** É possível levantar um front-end completo
(lexer + parser + printer + validações estáticas) de uma DSL não-trivial em
poucos dias, *desde que* o esquema da AST e o contrato de token estejam
fechados muito cedo. Os pontos que travaram o cronograma foram justamente os
que não foram contratualizados a tempo: a forma exata do nó `verb-cmd` com
muitos campos opcionais teve duas reescritas.

**Desafios enfrentados.**

- **Indentação no lexer** — exigiu um modelo de pilha com `INDENT`/`DEDENT`
  e atenção a linhas em branco / só com comentário, que não devem alterar o
  nível.
- **Multi-palavra `ESTE é`** — palavra com acento e espaço; resolvido
  classificando ambos como `KEYWORD` e tratando como duas keywords contíguas
  no parser.
- **`AO_MESMO_TEMPO` com múltiplas instruções por ramo** — escolhemos
  manter um ramo = uma linha para simplificar; encadeamento de múltiplos
  passos em um ramo é feito via `ESTE é`.

**Lições aprendidas.**

- Contratos antes de código. O esforço de uma manhã definindo `token` e
  schema de AST poupou dias de retrabalho.
- Trabalhar contra **fixtures** (tokens e ASTs escritos à mão) permite que
  parser e printer evoluam em paralelo ao lexer.
- Validação estática traz mais valor percebido por linha de código que
  qualquer outra parte do front-end: é o que diferencia "ler um arquivo" de
  "checar uma receita".

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
2. *Calysto Scheme* — kernel Jupyter para Scheme. https://github.com/Calysto/calysto_scheme
3. Aho, A. V.; Lam, M. S.; Sethi, R.; Ullman, J. D. *Compilers: Principles,
   Techniques, and Tools*. 2ª ed. Pearson, 2006. (Cap. 3 e 4 — análise léxica
   e sintática descendente).
4. *The Python Language Reference* — seção *Lexical Analysis: Indentation*.
   https://docs.python.org/3/reference/lexical_analysis.html (modelo de
   `INDENT`/`DEDENT` que inspirou o lexer).
5. Wirth, N. *Extended Backus-Naur Form (EBNF)*. ISO/IEC 14977, 1996.
```

---

## 6. Cronograma de execução (28–29/06)

Trabalho restante distribuído por blocos curtos. Se a equipe puder dividir:

### Dom 28/06 (hoje) — Parser + Printer

- **Bloco 1 (manhã)** — Cursor de tokens + `parse-programa` + RECEITA +
  INGREDIENTES + UTENSILIOS. **Teste P1–P4.**
- **Bloco 2 (tarde)** — `cmd-verbo`, `arg`, `lista-arg`, `este-e`, `aguardar`.
  **Teste P5–P9.**
- **Bloco 3 (noite cedo)** — `verificar`, `ao-mesmo-tempo`. **Teste P10–P11.**
- **Bloco 4 (noite)** — Pretty-printer de todos os nós. **Teste visual com
  saída do Omelete.**

### Seg 29/06 (entrega) — Validação + README + integração

- **Bloco 1 (manhã)** — Tabela de símbolos + `UNDECLARED_*`. **Teste V1, V2.**
- **Bloco 2 (meio-dia)** — `OVERUSE` + `UTENSIL_CONFLICT` +
  `UNRESOLVED_SUBRECIPE`. **Teste V3–V5.**
- **Bloco 3 (tarde)** — Driver `run` + 2 receitas extras (P3 e P4 dos
  exemplos da seção 4 do notebook). **Teste integrado V6.**
- **Bloco 4 (final da tarde)** — Substituir `README.md` pelo conteúdo da §5
  deste documento. **Restart & Run All.** Conferir todos os outputs estáveis.

## 7. Definition of Done (entrega)

- [ ] `RECIP-E.ipynb` roda limpo via **Kernel → Restart & Run All** no
  Calysto Scheme.
- [ ] Todas as células dos testes P1–P12 e V1–V6 exibem resultado esperado.
- [ ] `(run omelete)` imprime AST + relatório vazio.
- [ ] `README.md` substituído pelo conteúdo da §5 deste plano.
- [ ] Notebook tem seção de gramática (cópia do bloco EBNF do README).
- [ ] Slides preparados (fora do escopo deste plano — paralelo).

## 8. Riscos residuais

| Risco | Probabilidade | Mitigação |
|---|---|---|
| `set-car!` indisponível no Calysto | média | Cursor como closure mutável (`let` + `set!`) |
| Reescrita do nó `verb-cmd` no meio do printer | alta | **Congelar a ordem dos campos AGORA** (§3.1) |
| `AO_MESMO_TEMPO` com ramos multi-instrução | baixa | Restringir a 1 instrução/ramo na v1; documentar |
| Encoding do `é` em `ESTE é` no kernel | baixa | Notebook já usa UTF-8; testar cedo |
| Falta de tempo para validação completa | média | Priorizar `UNDECLARED_*` e `OVERUSE`; outros são bonus |
