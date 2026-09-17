# Boxer Dashboard

App de consulta da disponibilidade de materiais Boxer para Representantes e Time comercial.

Aplicação estática (HTML + JavaScript puro, sem build/bundler), hospedada como três páginas independentes que compartilham o mesmo backend Supabase. Parte dos dados de estoque/vendas/reservas é buscada em tempo real na API do ERP **ZenERP**.

## Sumário

- [Arquitetura](#arquitetura)
- [Autenticação (index.html)](#autenticação-indexhtml)
- [Painel de Disponibilidade (disponibilidade.html)](#painel-de-disponibilidade-disponibilidadehtml)
- [Sales 2.0 (sales2.html)](#sales-20-sales2html)
- [Integração com o ZenERP](#integração-com-o-zenerp)
- [Tabelas do Supabase](#tabelas-do-supabase)
- [Limitações conhecidas / pontos de atenção](#limitações-conhecidas--pontos-de-atenção)

## Arquitetura

- **Front-end**: 3 arquivos HTML independentes (`index.html`, `disponibilidade.html`, `sales2.html`), sem framework — JS puro embutido em `<script>`. Parsing de planilhas Excel feito no navegador com SheetJS (`xlsx.full.min.js`, via CDN).
- **Backend**: Supabase (Postgres + REST via PostgREST + Auth). Não existe backend próprio — todas as chamadas são feitas direto do navegador para a API REST do Supabase, autenticadas com o `access_token` da sessão do usuário.
- **Sessão**: após login, os dados ficam em `sessionStorage` (`boxer_session` com o token de acesso, `boxer_email`). Todas as páginas checam essa sessão ao carregar e redirecionam para `index.html` se ela não existir.
- **ZenERP**: integração feita client-side, com chamadas `POST` diretas para `https://api.zenerp.app.br/system/data/dataSourceOpRead`, usando um token (`ZENERP_TOKEN`) fixo no código de `disponibilidade.html` e `sales2.html`.

## Autenticação (index.html)

Tela de login/cadastro.

- **Login**: valida e-mail/senha via Supabase Auth (`POST /auth/v1/token?grant_type=password`). A validação da senha em si é toda delegada ao Supabase — a aplicação não faz nenhuma checagem própria.
- **Cadastro**: em duas etapas —
  1. Verifica se o e-mail está autorizado a criar conta, consultando a tabela `usuarios_permitidos` (`GET /rest/v1/usuarios_permitidos?email=eq.<email>`). Se não encontrar, bloqueia o cadastro.
  2. Se autorizado, cria o usuário de fato via `POST /auth/v1/signup` (Supabase Auth).
- Ao logar/cadastrar com sucesso, salva em `sessionStorage`: `boxer_session` (resposta completa do Auth, incluindo `access_token`) e `boxer_email`.

## Painel de Disponibilidade (disponibilidade.html)

Página principal, com a **Tabela FUP** ("Materiais em Trânsito — Visão Representantes") e uma tabela de "Entradas Boxer — Últimos 30 dias". Os dados ficam persistidos na tabela `dashboard_data` do Supabase (linha única).

### Upload da planilha COMEX (`processFile`)

Lê a aba **`FUP Chegadas`** da planilha (cabeçalho na linha 4, dados a partir da linha 5) e filtra as linhas por:
- `CATEGORIA` precisa estar em `CATS_ALLOWED` = `["Grandes","Médias","Pequenas","Automação/Laser","Máscaras","Tochas","Acessórios de solda"]` (qualquer categoria fora dessa lista é descartada silenciosamente).
- `ITEM` precisa estar preenchido.

Para cada código único de item, o estoque e as vendas dos últimos 90 dias são buscados **em tempo real no ZenERP** (a antiga leitura de uma aba "Data" da planilha foi substituída e ficou comentada no código, para reverter facilmente):
- `fetchEstoqueZenerp()` → estoque atual, endpoint `/salesbreath/stockAvailabilityCube`.
- `fetchVendas90Zenerp()` → vendas dos últimos 90 dias, endpoint `/fiscal/report/invoiceCube`.

Com esses valores é calculada a **cobertura** (dias de estoque): `cobertura = (estoque / (vendas90 / 3)) * 30`. A partir disso, `stockStatus` é classificado como:
- `zero` — estoque igual a 0.
- `low` — cobertura menor que 60 dias.
- `ok` — caso contrário (esses itens ficam ocultos da tabela por padrão).

Também é lida uma segunda aba (nome contendo "boxer"/"entrada") via `parseBoxerSheet()`, para montar a tabela de entradas dos últimos 30 dias.

### Upload de reservas (`processReservas`)

Lê uma planilha separada (aba `sheet1`/`Query Mês`), soma a quantidade por código de produto para linhas com status "Aguardando importação" ou "APPROVED" (excluindo linhas da Tekweld), e mescla o resultado (`reservasMap`) nos dados da Tabela FUP.

> Observação: esse fluxo de upload manual de reservas coexiste hoje com a sincronização automática de reservas feita a partir do ZenERP em `sales2.html` (ver seção seguinte), que também grava na mesma tabela `dashboard_data`.

### KPIs e filtros

- Cards de KPI: **Estoque Zerado**, **Estoque Baixo** (cobertura < 60 dias) e **Chegando em 30 dias** (previsão do representante dentro dos próximos 30 dias) — cada um abre um modal com os itens correspondentes.
- Filtros disponíveis na tabela: busca por código/descrição, categoria, status logístico (em trânsito, atracado no porto, liberado transportadora, canal amarelo/vermelho, aguardando embarque, pedido solicitado/confirmado), filtro de estoque (zero/baixo/ambos) e ordenação por data prevista do representante.

## Sales 2.0 (sales2.html)

Página voltada para consulta rápida de saldo disponível por produto. Os dados de upload ficam na tabela `sales2_data`.

### Uploads

- **ENDEREÇO MAQ** (`processEnderecos`): planilha com quantidade física por endereço (colunas: código, descrição, quantidade). Alimenta `enderecosMap`.
- **STATUS ESTOQUE** (`processStatus`): planilha de pedidos (aba `Report`). Alimenta `statusMap`, hoje usado apenas como fonte auxiliar de descrição de produto e para identificar quais códigos já são conhecidos — **não é mais usado para calcular saldo** (o cálculo antigo, `enderecosMap - statusMap`, está comentado no código para reverter facilmente se necessário).

### KPIs (Total de Produtos / Disponíveis / Indisponíveis)

Calculados com estoque em tempo real do ZenERP: a cada acesso à página, `refreshEstoqueZen()` chama `fetchTodosEstoqueZenerp()` (endpoint `/salesbreath/stockAvailabilityCube`, sem filtrar por código — traz todos os produtos) e:
- Atualiza o cache `estoqueZenMap` usado para calcular `disponivel = saldo > 0`.
- Detecta e cadastra automaticamente no `enderecosMap` (salvando no banco) qualquer código que exista no ZenERP mas não conste em nenhuma planilha carregada até então.

### Consulta de disponibilidade

Campo de busca com autocomplete (por código ou descrição) + campo de quantidade. A cada consulta (com debounce de 400ms), `verificarDisponibilidade()` busca o estoque atual do item direto no ZenERP e compara com a quantidade solicitada, exibindo "DISPONÍVEL PARA REQUISIÇÃO" ou "QUANTIDADE INDISPONÍVEL — CONSULTAR INTERNAMENTE" (sem expor ao usuário o número exato de estoque). Um `consultaRequestId` evita que uma resposta antiga sobrescreva uma consulta mais recente.

### Reservas

`fetchReservasZenerp()` busca no ZenERP a soma de quantidade dos pedidos com status **PREPARED**, via endpoint `/sale/report/saleCube`. `refreshReservasZen()` roda a cada acesso à página e grava esse resultado na tabela `dashboard_data` (mesma usada pela Tabela FUP de disponibilidade.html), via `syncReservasToDisponibilidade()`.

## Integração com o ZenERP

Todas as chamadas são `POST https://api.zenerp.app.br/system/data/dataSourceOpRead`, com headers `Tenant: boxer` e `Authorization: Bearer <ZENERP_TOKEN>` (token fixo no código-fonte de `disponibilidade.html` e `sales2.html`).

| Endpoint (`code`) | Usado para | Onde |
|---|---|---|
| `/salesbreath/stockAvailabilityCube` | Estoque atual (`quantity_balance` por `product_code`) | `fetchEstoqueZenerp()` (disponibilidade.html e sales2.html) e `fetchTodosEstoqueZenerp()` (sales2.html) |
| `/fiscal/report/invoiceCube` | Vendas dos últimos 90 dias (`sum_quantity`), para calcular cobertura | `fetchVendas90Zenerp()` (disponibilidade.html) |
| `/sale/report/saleCube` | Quantidade reservada — soma de pedidos com `STATUS_LIST: ["PREPARED"]` (`sum_quantity`) | `fetchReservasZenerp()` (sales2.html) |

## Tabelas do Supabase

- **`usuarios_permitidos`** — whitelist de e-mails autorizados a criar conta. Coluna usada: `email`.
- **`auth.users`** — gerenciada pelo Supabase Auth (login/cadastro via `/auth/v1/token` e `/auth/v1/signup`); não é acessada diretamente via REST pela aplicação.
- **`dashboard_data`** (linha única) — dados da Tabela FUP e entradas Boxer. Colunas: `id`, `all_data` (array JSON: `codigo, descricao, categoria, status, qtd, prevBoxer, prevRep, estoque, stockStatus, cobertura, reservas`), `reservas_map` (JSON código→quantidade), `boxer_data` (array JSON das entradas), `last_update`.
- **`sales2_data`** (linha única) — dados do Sales 2.0. Colunas: `id`, `enderecos_map` (JSON código→`{descricao, quantidade}`), `status_map` (JSON código→`{descricao, quantidade}`), `last_update_enderecos`, `last_update_status`.

## Limitações conhecidas / pontos de atenção

- Não há paginação em nenhuma consulta ao Supabase — as tabelas de dados são sempre lidas/gravadas como linha única (`limit=1`).
- O token do ZenERP (`ZENERP_TOKEN`) está hardcoded no HTML, visível a qualquer usuário logado que inspecione o código-fonte.
- A lista `CATS_ALLOWED` (disponibilidade.html) está duplicada entre o filtro de importação e um `<select>` no HTML — precisa ser atualizada nos dois lugares.
- Vários cálculos antigos (saldo local em sales2.html, leitura da aba "Data" em disponibilidade.html, extração de reservas da planilha STATUS ESTOQUE) foram substituídos por chamadas ao ZenERP, mas o código original foi mantido comentado no lugar, propositalmente, para facilitar reverter caso necessário.
