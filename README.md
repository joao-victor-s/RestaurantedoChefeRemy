# RECIP-E — Restaurante do Chefe Remy

Front-end (lexer → parser → impressão/validação) da DSL **RECIP-E** para receitas
de cozinha, implementado em **Scheme**. Projeto da disciplina **MC346 —
Paradigmas de Programação** (UNICAMP).

Todo o código vive em um único notebook: **`RECIP-E.ipynb`**.

## Como rodar

1. Instale o kernel **Calysto Scheme**:
   ```
   pip install calysto-scheme
   python -m calysto_scheme install --sys-prefix
   ```
2. Abra o notebook:
   ```
   jupyter notebook RECIP-E.ipynb
   ```
3. No menu: **Kernel → Restart & Run All**.

## Organização do notebook

O notebook é dividido em seções, na ordem do pipeline:

| Seção | Conteúdo | Responsável |
|-------|----------|-------------|
| 0. Setup / Contrato | formato do token + tabelas de palavras reservadas | A |
| 1. Lexer | `tokeniza`: texto → lista de tokens | A |
| 2. Parser | tokens → AST | B |
| 3. Pretty-print + Validação | impressão da AST + checagens estáticas | C |
| 4. Exemplos | receitas de teste | todos |

## Estado atual

- **Dia 1 (Sáb 27/06):** seção **1. Lexer** implementada — números, strings,
  identificadores, palavras reservadas e remoção de comentários (`//` e `/* */`).
- **Próximos:** indentação `INDENT`/`DEDENT` (Dia 2), parser (B) e
  pretty-print/validação (C).
