# Módulo `products`

> Base: commit `b99a0b2` · Global: [Arquitetura](../../../docs/ARQUITETURA_DO_SISTEMA.md) · [Objetivo — F2/F3](../../../docs/OBJETIVO_DO_SISTEMA.md)

## Objetivo
Manter e expor o catálogo de produtos da loja.

## Responsabilidade principal
Dono da tabela `products` e do cache `products:list:*`. CRUD administrativo e consulta pública paginada.

## Funcionalidades existentes
| Método | Rota | Acesso | Comportamento |
|---|---|---|---|
| GET | `/api/v1/products` | público | Query: `page` (padrão 1), `limit` (padrão 20), `categoryId`, `search` (ILIKE em `name` ou `description`). Só `status = ACTIVE`, ordenado por `createdAt desc`, com `category`. Resposta `{ data, pagination: { page, limit, totalItems, totalPages } }`. Cache Redis de 60 s por combinação de parâmetros. |
| GET | `/api/v1/products/:id` | público | Produto + `category` (qualquer status); 404. |
| POST | `/api/v1/products` | ADMIN | 409 se o `sku` já existir; cria com o body; 201. |
| PUT | `/api/v1/products/:id` | ADMIN | 404 se não existir; 409 se o novo `sku` já existir; atualiza. |
| DELETE | `/api/v1/products/:id` | ADMIN | 204; 404 se não existir (P2025). |

Repository: `create`, `update`, `delete`, `findById` (inclui `category`), `findBySku`, `findAll({skip,take,categoryId,search})` (usa `findMany` + `count` em paralelo).

## Dependências
- **Internas:** `config/database`, `config/redis`, `utils/AppError`, `middlewares/authMiddleware`.
- **Externas:** Prisma, redis.

## Módulos relacionados
- `categories`: FK `categoryId`.
- `orders`: importa `productRepository.findById` no checkout e decrementa `stock` em `orderRepository` (ARQ-03).
- `cart`: lê `prisma.product` diretamente (ARQ-04).

## Pontos de entrada
`productRoutes.js` (montado em `server.js:51`).

## Fluxos importantes
Listagem (cache-aside): monta a chave → `GET` no Redis (erro só é logado) → se houver cache, retorna → senão consulta o banco → `SET` com EX 60 → retorna.

## Arquivos críticos
- `productRepository.js`: usado também por `orders`; mudanças de assinatura afetam o checkout.
- `productService.js`: regra de SKU e cache.

## Observações técnicas e débitos
- **DAD-01:** o cache não é invalidado em create/update/delete; o `set` do cache fica fora do `try/catch` (Redis fora → 500 na listagem).
- **SEC-04:** sem validators; o body vai direto ao Prisma (`price`, `stock` negativo, `tenantId`, `status` arbitrário).
- **DAD-06:** excluir um produto com `order_items` (P2003) → 500; `categoryId` inválido no create → 500.
- **DAD-10:** `limit` sem valor máximo; `page`/`limit` negativos não são tratados.
- **ARQ-06:** `adminOnly` duplicado.
- **OPS-02:** usa `console.error` em vez do `logger`.
- `GET /:id` retorna produtos `INACTIVE`. **Hipótese:** comportamento intencional para o admin, mas também está exposto a visitantes.
