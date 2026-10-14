# Lógica de cálculo do ESTOQUE_ATUAL via ZenERP

Este documento descreve, em detalhe, como o campo `ESTOQUE_ATUAL` é obtido e
calculado no `logistica.html` (função `fetchEstoqueZenerp`), para que a mesma
lógica possa ser replicada em outro programa/arquivo.

## Visão geral

O estoque atual de cada produto é calculado a partir de **duas chamadas** à
API do ZenERP:

1. **Estoque físico por endereço** (`/material/report/stockCube`) — de onde
   vem a quantidade física (`quantity`) de cada produto, filtrada para contar
   apenas os endereços marcados como `MAQUINAS` ou `PECAS`.
2. **Disponibilidade/saída de estoque** (`/salesbreath/stockAvailabilityCube`)
   — de onde vem `quantity_outgoing`, a quantidade já comprometida para saída
   (valor negativo).

A fórmula final, por produto, é:

```
ESTOQUE_ATUAL = max(0, soma(quantity nos registros MAQUINAS/PECAS) + soma(quantity_outgoing))
```

Como `quantity_outgoing` já vem negativo, **somá-lo** (e não subtraí-lo) ao
estoque físico é que desconta corretamente a quantidade pendente de saída —
subtrair um valor já negativo teria o efeito oposto (somaria o valor absoluto
de volta ao estoque, inflando-o). O resultado nunca é menor que zero.

> **Correção (2026-10-14):** a versão original deste documento descrevia a
> fórmula com subtração (`- soma(quantity_outgoing)`), o que é um bug — como
> `quantity_outgoing` já é negativo, subtraí-lo soma seu valor absoluto de
> volta ao estoque em vez de descontá-lo. A fórmula correta usa soma (`+`),
> como mostrado acima.

## Autenticação e endpoint base

- Base URL: `https://api.zenerp.app.br`
- Header `Tenant: boxer`
- Header `Authorization: Bearer <ZENERP_TOKEN>` (token fixo, hardcoded no
  cliente — não há backend próprio)
- Header `Content-Type: application/json`

## Passo 1 — Estoque físico (stockCube)

**Requisição:**

```
POST https://api.zenerp.app.br/system/data/dataSourceOpRead
```

**Corpo:**

```json
{
  "code": "/material/report/stockCube",
  "parameters": {
    "SHOW_PRODUCT": true,
    "SHOW_PRODUCT_PACKING": true,
    "SHOW_ADDRESS": true
  }
}
```

**Resposta esperada:** array de objetos, cada um representando um registro de
estoque físico em um endereço, contendo (entre outros campos) `product_code`,
`quantity` e `address_tags`.

**Processamento:**

1. Validar que a resposta é um array (`Array.isArray`). Se não for, tratar
   como falha.
2. Filtrar a lista para manter **somente** os registros cujo `address_tags`
   seja exatamente `'MAQUINAS'` ou `'PECAS'` (outros endereços não entram no
   cálculo de estoque atual). Chame esse resultado filtrado de `jsonFisico`.
3. Para cada `product_code` (código do produto), um mesmo produto pode
   aparecer em **mais de um registro** dentro de `jsonFisico` (por exemplo,
   itens de categoria PECAS costumam ter múltiplos registros/lotes para o
   mesmo código). Por isso, o valor de estoque físico de um produto é a
   **soma** de `quantity` de todos os registros de `jsonFisico` cujo
   `product_code` bate (comparação exata, como string) com o código do
   produto — nunca pegar apenas o primeiro registro encontrado.

## Passo 2 — Disponibilidade/saída de estoque (stockAvailabilityCube)

**Requisição:**

```
POST https://api.zenerp.app.br/system/data/dataSourceOpRead
```

**Corpo:**

```json
{
  "code": "/salesbreath/stockAvailabilityCube",
  "parameters": {
    "SHOW_PRODUCT": true,
    "SHOW_PRODUCT_PACKING": true,
    "SHOW_SCHEDULE": true
  }
}
```

**Resposta esperada:** array de objetos, cada um contendo (entre outros
campos) `product_code` e `quantity_outgoing` (valor negativo, representando
saída de estoque já programada/comprometida).

**Processamento:**

1. Validar que a resposta é um array. Se não for, tratar como falha.
2. Um mesmo `product_code` pode aparecer em mais de um registro aqui também
   (ex.: diferentes `schedule`/qualidade). Somar `quantity_outgoing` de todos
   os registros cujo `product_code` bate (comparação exata) com o código do
   produto.

## Passo 3 — Cálculo final por produto

Para cada código de produto da lista de entrada (`codigos`):

```js
const somaFisico = jsonFisico
  .filter(row => String(row.product_code) === String(codigo))
  .reduce((s, row) => s + (Number(row.quantity) || 0), 0);

const somaSaida = linhasDaAvailabilityCube
  .reduce((s, row) => s + (Number(row.quantity_outgoing) || 0), 0);

const estoqueAtual = Math.max(somaFisico + somaSaida, 0); // somaSaida já é negativo
```

Onde `linhasDaAvailabilityCube` são os registros do Passo 2 cujo
`product_code` bate com o código (sem o filtro de `address_tags`, que só se
aplica ao Passo 1).

Se um código não aparecer em nenhum registro do Passo 2 (resposta do
`stockAvailabilityCube`), ele é considerado "não encontrado" e seu valor final
é `0`, independentemente de aparecer ou não em `jsonFisico`. Em outras
palavras: a existência do item é determinada pela resposta do
`stockAvailabilityCube` (Passo 2); o `stockCube` físico (Passo 1) só contribui
quantidade quando o item já foi confirmado como existente no Passo 2.

## Regras de matching

- A comparação de código de produto é sempre **exata**, convertendo ambos os
  lados para string: `String(row.product_code) === String(codigo)`.
- Nunca usar correspondência por substring (`.includes()`) ou varredura de
  todos os campos do objeto — isso pode gerar falsos positivos.
- Nunca usar `.find()` (que pega só o primeiro registro). Sempre usar
  `.filter()` seguido de soma (`.reduce()`), pois um mesmo produto pode ter
  múltiplos registros em ambas as chamadas.

## Tratamento de falhas e fallback

- Cada chamada HTTP é feita através de um wrapper genérico (`chamarApiJson`)
  que:
  - Tem timeout curto (10s) por tentativa.
  - Tenta no máximo 2 vezes, com 3s de espera entre tentativas.
  - Só tenta de novo em falha de rede/timeout — **não** tenta de novo em
    resposta HTTP de erro (4xx/5xx), pois nesse caso a falha não é transitória.
  - Nunca lança exceção: sempre retorna `{ ok: boolean, data: any }`.
- Se a chamada do Passo 1 (`stockCube` físico) falhar (`ok: false`), a função
  inteira retorna `null` imediatamente, **sem** chamar o Passo 2.
- Se a chamada do Passo 1 tiver sucesso mas a do Passo 2 falhar, a função
  também retorna `null`.
- Se qualquer uma das respostas não for um array (formato inesperado), a
  função retorna `null`.
- Quando a função retorna `null` (ou quando retorna um resultado mas nenhum
  código bateu — `matched === 0`), quem chamou (`parseStock`) **não altera**
  o valor de `ESTOQUE_ATUAL` daquele produto, mantendo o valor já presente
  (vindo da planilha Excel, ou `0` se não houver planilha). Esse é o
  mecanismo de fallback: cada campo (`RECEBIMENTO`, `ESTOQUE_ATUAL`) falha de
  forma independente, sem corromper os outros campos.
- Um toast "Coletando dados..." (persistente, só some quando o próximo toast
  aparece) é exibido antes de iniciar as chamadas, para dar feedback visual
  ao usuário de que a busca está em andamento.
- Ao final, um toast resume o resultado (ex.: `ZenERP — Recebimento 42/50 ·
  Estoque 48/50`, ou `Estoque: fallback` se a API falhou/não bateu nada).

## Retorno da função

A função retorna um dos dois formatos:

- `null` — em caso de falha de qualquer uma das duas chamadas de API, ou
  resposta em formato inesperado.
- `{ values, matched, total }`, onde:
  - `values` é um array **paralelo** à lista de códigos de entrada (`codigos`),
    contendo o `ESTOQUE_ATUAL` calculado para cada um (na mesma ordem).
  - `matched` é a quantidade de códigos que tiveram correspondência na
    resposta do Passo 2 (`stockAvailabilityCube`).
  - `total` é o total de códigos (`codigos.length`).

## Pseudocódigo completo

```
function fetchEstoqueZenerp(codigos):
    mostrar_toast("Coletando dados...", persistente)

    // Passo 1: estoque físico
    respostaFisico = chamarApi(POST stockCube, { SHOW_PRODUCT, SHOW_PRODUCT_PACKING, SHOW_ADDRESS })
    se respostaFisico falhou: retorna null

    // Passo 2: disponibilidade/saída
    respostaDisponibilidade = chamarApi(POST stockAvailabilityCube, { SHOW_PRODUCT, SHOW_PRODUCT_PACKING, SHOW_SCHEDULE })
    se respostaDisponibilidade falhou: retorna null

    se respostaFisico.data não é array ou respostaDisponibilidade.data não é array:
        retorna null

    jsonFisico = respostaFisico.data.filtrar(row => row.address_tags in ["MAQUINAS", "PECAS"])

    matched = 0
    values = []
    para cada codigo em codigos:
        linhasDisponibilidade = respostaDisponibilidade.data.filtrar(row => row.product_code == codigo)
        se linhasDisponibilidade vazio:
            values.adicionar(0)
            continuar
        matched += 1

        somaFisico = soma(jsonFisico.filtrar(row => row.product_code == codigo).mapear(row => row.quantity))
        somaSaida  = soma(linhasDisponibilidade.mapear(row => row.quantity_outgoing))  // já é negativo

        values.adicionar(max(0, somaFisico + somaSaida))  // somaSaida já é negativo

    retorna { values, matched, total: codigos.length }
```

## Integração com o restante do sistema (contexto)

- `fetchEstoqueZenerp` é chamada dentro de `parseStock(wb)`, em paralelo com
  `fetchRecebimentoZenerp(codigos)` (via `Promise.all`), usando a mesma lista
  de códigos de produto (`allStock.map(r => r.CODIGO)`).
- A lista de códigos/nomes de produto (`allStock`) vem preferencialmente do
  catálogo do ZenERP (`fetchCatalogoZenerp`, endpoint
  `/catalog/product/productPacking`), com fallback para a aba "Data" de uma
  planilha Excel carregada pelo usuário, caso a API do catálogo falhe.
- Antes de qualquer chamada de API, cada produto já tem um valor "base" de
  `ESTOQUE_ATUAL` vindo da planilha Excel (ou `0`, se não houver planilha). O
  resultado de `fetchEstoqueZenerp` só **sobrescreve** esse valor se
  `matched > 0`.
