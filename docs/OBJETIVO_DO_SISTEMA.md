# Objetivo do Sistema

> **Status:** documento oficial de referência (Spec Driven Development)
> **Base analisada:** branch `main`, commit `b99a0b2` · análise de 2026-09-28
> **Relacionado:** [Arquitetura do Sistema](./ARQUITETURA_DO_SISTEMA.md)

Trechos marcados **Hipótese** são inferências e não comportamento implementado. Referências `SEC-xx`, `ARQ-xx`, `DAD-xx`, `OPS-xx` apontam para o catálogo de débitos da Arquitetura (§9).

---

## 1. Propósito principal

O sistema é o **back-end (API REST) de uma loja virtual**. Ele cobre o ciclo básico de venda online: cadastro de clientes, catálogo de produtos organizado em categorias, carrinho de compras, fechamento de pedido com baixa de estoque e registro de pagamento.

O README do projeto o define como "API RESTful modular construída com Node.js, Express, Prisma (PostgreSQL) e Redis". Não há front-end neste repositório.

**Hipótese:** pelo nome da pasta (`6308-spec-driven-development`), pelas branches `aula-2` a `aula-5` e pelo projeto "Curso Alura", trata-se de um projeto didático, usado como base para praticar Spec Driven Development, e não de um sistema em produção.

## 2. Problemas que resolve

| Problema | Como o sistema resolve (implementado) |
|---|---|
| Identificar clientes e proteger operações | Cadastro/login com senha em bcrypt e JWT de acesso (`auth`). |
| Manter um catálogo pesquisável | CRUD de produtos com SKU único, estoque e status; listagem paginada com filtro por categoria e busca textual em nome/descrição (`products`). |
| Organizar o catálogo | Categorias com hierarquia pai/filho (`categories`). |
| Guardar a intenção de compra antes do pedido | Carrinho por usuário no Redis, com validação de estoque e expiração em 30 dias (`cart`). |
| Evitar vender sem estoque e manter o preço cobrado | Checkout transacional que revalida estoque, decrementa quantidades e grava o preço de cada item no momento da compra (`orders`). |
| Controlar se o pedido foi pago | Pagamento associado 1:1 ao pedido, com confirmação e cancelamento que atualizam o status do pedido (`payments`). |
| Reduzir carga na consulta ao catálogo | Cache de 60 s da listagem de produtos no Redis. |

## 3. Atores

| Ator | Como é identificado no código | O que pode fazer hoje |
|---|---|---|
| **Visitante** (anônimo) | Sem header `Authorization` | Registrar-se, fazer login, listar/consultar produtos e categorias, `GET /health`. |
| **Cliente** | JWT com `role: USER` | Tudo do visitante + carrinho, checkout, listar os **próprios** pedidos, ver/editar/excluir o **próprio** usuário, operar pagamentos (atenção: de **qualquer** pedido, ver SEC-02). |
| **Administrador** | JWT com `role: ADMIN` | Tudo do cliente + CRUD de produtos e categorias; ver, editar (inclusive `role`) e excluir qualquer usuário. |
| **Gateway de pagamento** | — | **Não existe integração.** O campo `Payment.externalId` e o comentário "externalId gerado depois no Payment Gateway" indicam intenção. **Hipótese:** a confirmação via `POST /payments/:id/confirm` simula um retorno (callback) do gateway. |
| **Infraestrutura** (PostgreSQL, Redis) | `config/` | Persistência e cache. |

> Observação: hoje qualquer visitante consegue se registrar como `ADMIN` enviando `"role": "ADMIN"` (SEC-01). A tabela acima descreve o modelo de papéis **pretendido** pelas regras de rota; esse débito o invalida na prática.

## 4. Principais fluxos de negócio

### F1 — Cadastro e autenticação
1. Visitante envia `name`, `email`, `password` (mín. 6 caracteres) para `POST /api/v1/auth/register`.
2. O sistema recusa e-mail já usado (409), grava a senha com hash bcrypt e devolve o usuário + `accessToken` (expira em `JWT_ACCESS_EXPIRATION`, 15 min no `.env` atual) + `refreshToken`.
3. `POST /api/v1/auth/login` valida as credenciais (401 genérico em caso de falha) e devolve novos tokens.
4. Não existe renovação de token nem logout (SEC-05).

### F2 — Gestão de catálogo (Administrador)
1. Cria categorias (`POST /categories`, com `parentId` opcional).
2. Cria produtos (`POST /products`) com `sku` único, `price`, `stock`, `categoryId`, `status`.
3. Atualiza e remove. Uma categoria com produtos não pode ser removida (409). Um produto que já tem pedido não pode ser removido (hoje retorna 500, DAD-06).

### F3 — Navegação no catálogo (Visitante/Cliente)
1. `GET /products?page=&limit=&categoryId=&search=` retorna só produtos `ACTIVE`, do mais novo para o mais antigo, com categoria e paginação.
2. `GET /products/:id` retorna o produto com a categoria (inclusive `INACTIVE`).
3. `GET /categories` e `GET /categories/:id` retornam categorias com as filhas diretas.

### F4 — Carrinho (Cliente)
1. `POST /cart/items { productId, quantity }`: o produto precisa existir e ter estoque para a quantidade acumulada; o item é adicionado ou tem a quantidade somada; o total é recalculado.
2. `DELETE /cart/items/:productId` remove o item (o parâmetro se chama `itemId`, mas é o `productId`).
3. `GET /cart` retorna `{ items: [{productId, name, price, quantity}], total }`.

### F5 — Checkout (Cliente)
1. `POST /orders` (sem body) transforma o carrinho em pedido.
2. Recusa carrinho vazio (400) e produto removido, sem estoque ou inativo (409).
3. Em uma transação: baixa o estoque, cria o pedido `PENDING` com os itens a preço atual e cria o pagamento `AWAITING_CONFIRMATION`.
4. Esvazia o carrinho e registra o evento `order.created` em log.
5. `GET /orders` lista os pedidos do cliente com itens e pagamento.

### F6 — Pagamento (Cliente)
1. O pagamento já nasce no checkout. `POST /payments { orderId }` devolve o existente ou cria um, se não houver.
2. `POST /payments/:id/confirm` (header `Idempotency-Key` obrigatório) marca pagamento e pedido como `PAID`; chamadas repetidas devolvem o pagamento sem alterações.
3. `POST /payments/:id/cancel` marca pagamento e pedido como `CANCELED`, **sem devolver estoque** (DAD-03).
4. O identificador do pagamento é obtido no retorno de `GET /orders` (campo `payment.id`).

### F7 — Conta do usuário
1. `GET /users/:id`: o cliente vê só a si mesmo; o admin vê qualquer um. A senha nunca é retornada.
2. `PUT /users/:id`: o cliente altera só a si mesmo e não altera `role`. A alteração de senha por esta rota grava texto puro (SEC-03).
3. `DELETE /users/:id`: o cliente exclui a si mesmo; o admin exclui qualquer um. Falha com 500 se houver pedidos (DAD-06).

## 5. Funcionalidades centrais (inventário)

| # | Funcionalidade | Módulo | Estado |
|---|---|---|---|
| 1 | Registro e login com JWT | auth | Implementado (com SEC-01, SEC-05) |
| 2 | Perfil, atualização e exclusão de usuário | users | Implementado (com SEC-03) |
| 3 | CRUD de categorias hierárquicas | categories | Implementado |
| 4 | CRUD de produtos + listagem paginada, filtrada e cacheada | products | Implementado (com DAD-01) |
| 5 | Carrinho em Redis | cart | Implementado (com DAD-07) |
| 6 | Checkout transacional com baixa de estoque | orders | Implementado (com DAD-02) |
| 7 | Pagamento simulado (criar/confirmar/cancelar) | payments | Implementado, **sem gateway** |
| 8 | Healthcheck | server.js | Implementado (estático, OPS-01) |
| — | Listagem de usuários (admin) | users | Não exposto (há `findAll` no repository) |
| — | Refresh de token / logout | auth | Não implementado |
| — | Multi-tenancy | todos | Só colunas no banco (ARQ-09) |
| — | Documentação OpenAPI/Swagger | server.js | Comentado; arquivo inexistente |
| — | Eventos/webhooks | orders, payments, categories | Só log (ARQ-10) |
| — | Cupons de desconto | cart | **Hipótese** a partir do histórico git; não existe no código (OPS-04) |

## 6. Visão de produto

O que o código já entrega é um **MVP de e-commerce B2C** de catálogo único: um cliente navega, monta um carrinho, fecha o pedido e o pagamento é confirmado manualmente pela API.

Direções indicadas pelo próprio código (todas **Hipóteses** de evolução, não compromissos):
- **Multi-loja / SaaS:** as colunas `tenantId`, a chave de carrinho `cart:<tenantId>:<userId>` e o comentário de CORS "configurável por tenant futuramente" sugerem uma plataforma que atenda várias lojas.
- **Integração com gateway de pagamento real:** o campo `externalId` ("clientSecret ou id do gateway") e o header `Idempotency-Key` seguem o padrão de gateways como o Stripe.
- **Arquitetura orientada a eventos:** os logs "Evento Emitido" (`order.created`, `order.paid`) e o comentário de webhook `category.updated` indicam a intenção de publicar eventos para outros sistemas.
- **Documentação de API pública:** dependências de Swagger já instaladas.

## 7. Contexto operacional

| Aspecto | Situação atual |
|---|---|
| Implantação | Container Docker (`node:20-alpine`) orquestrado por `docker-compose` com Postgres 15 e Redis 7. No start, o container executa `prisma generate` e `prisma migrate deploy`. |
| Ambientes | Controlados por `NODE_ENV`. Em `development`: stack trace nas respostas de erro, logs de nível `debug` e log HTTP ativo. Fora dele: nível `warn` e sem log HTTP. |
| Configuração | `.env` (dotenv), 9 variáveis (ver Arquitetura §4.4). Não há `.env.example` (OPS-06). |
| Observabilidade | Winston em console + `logs/all.log` + `logs/error.log`; morgan para HTTP; `/health` estático. Sem métricas nem tracing. |
| Dados | PostgreSQL (fonte da verdade); Redis (carrinho, cache, tokens), com volumes persistentes no compose. Carrinho e tokens se perdem se o Redis for limpo. |
| Qualidade | Sem testes automatizados nem CI no repositório (OPS-04). |
| Escala | Stateless na API (sessão no JWT, carrinho no Redis), o que permite várias instâncias. **Hipótese:** o cache sem invalidação (DAD-01) e o N+1 do checkout (DAD-02) seriam os primeiros gargalos. |
| Segurança operacional | helmet e compression ativos; CORS aberto; sem rate limiting (SEC-06). |
