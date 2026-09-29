# Módulo `cart`

> Base: commit `b99a0b2` · Global: [Arquitetura](../../../docs/ARQUITETURA_DO_SISTEMA.md) · [Objetivo — F4](../../../docs/OBJETIVO_DO_SISTEMA.md)

## Objetivo
Guardar os itens que o cliente pretende comprar antes do checkout.

## Responsabilidade principal
Dono das chaves Redis `cart:<tenantId>:<userId>` (TTL 30 dias). Não tem tabela no Postgres.

## Funcionalidades existentes
Todas autenticadas: o `authMiddleware` é aplicado em `server.js:50`, não no router.

| Método | Rota | Body/Params | Comportamento |
|---|---|---|---|
| GET | `/api/v1/cart` | — | `{ data: { items: [{productId, name, price, quantity}], total } }`; carrinho vazio se não existir. |
| POST | `/api/v1/cart/items` | `{ productId, quantity }` | 404 se o produto não existir; 409 `OUT_OF_STOCK` se `stock < quantity` ou `stock < quantidade acumulada`; soma a quantidade se o item já existir, senão adiciona (com `name` e `price` atuais); recalcula `total`; renova o TTL. |
| DELETE | `/api/v1/cart/items/:itemId` | `itemId` = **productId** | Remove o item (não dá erro se ele não existir); recalcula `total`. |

`cartService` (API usada por outros módulos): `getCart(userId, tenantId='default')`, `addItem(...)`, `removeItem(...)`.

## Dependências
- **Internas:** `config/redis`, `config/database` (**acesso direto a `product`**), `utils/AppError`.
- **Externas:** redis, Prisma.

## Módulos relacionados
- `products`: lê `stock`, `name` e `price` (ARQ-04: deveria usar o `productService`).
- `orders`: chama `cartService.getCart` no checkout e **apaga a chave diretamente** no Redis (ARQ-03).

## Pontos de entrada
`cartRoutes.js` (montado em `server.js:50` com `authMiddleware`).

## Fluxos importantes
`addItem`: busca o produto no Postgres → valida estoque → `GET` do carrinho → merge → recalcula total → `SET` com EX 30 dias.

## Arquivos críticos
`cartService.js`: o formato do JSON e da chave é um contrato implícito com `orders`.

## Observações técnicas e débitos
- **DAD-07:** sem validação. `quantity` como string concatena (`"2" + "2" = "22"`); aceita valores zero ou negativos; não verifica `status === 'ACTIVE'`; `productId` ausente → Prisma lança exceção → 500.
- **ARQ-03:** falta `clearCart()`; `orders` duplica o formato da chave (`cart:default:<userId>`).
- **ARQ-08:** o controller responde `400 { error: 'UserId required' }` fora do padrão. O fallback `req.query.userId` é código morto, porque o `authMiddleware` sempre define `req.user`. **Se** o middleware for retirado, esse fallback permitiria ler e alterar o carrinho de qualquer usuário.
- **ARQ-07:** a auth é aplicada em `server.js`, diferente dos outros módulos (há um `require` comentado no router).
- **ARQ-09:** `tenantId` sempre `'default'`.
- **DAD-05:** `price` e `total` em ponto flutuante.
- Não há controle de concorrência (read-modify-write no Redis sem `WATCH`/transação): duas adições simultâneas podem perder uma delas.
- **OPS-04:** os testes unitários deste módulo existiram e foram removidos no commit `a9d5718`. **Hipótese:** as mensagens de commit ("coupon behavior") indicam uma regra de cupom planejada para o carrinho, inexistente no código atual.
