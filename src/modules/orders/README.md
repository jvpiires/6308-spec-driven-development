# Módulo `orders`

> Base: commit `b99a0b2` · Global: [Arquitetura §6.2](../../../docs/ARQUITETURA_DO_SISTEMA.md) · [Objetivo — F5](../../../docs/OBJETIVO_DO_SISTEMA.md)

## Objetivo
Converter o carrinho do cliente em pedido, garantindo estoque e preço, e listar os pedidos do cliente.

## Responsabilidade principal
Dono das tabelas `orders` e `order_items`. Orquestra o checkout, que é o fluxo de maior acoplamento do sistema.

## Funcionalidades existentes
Todas autenticadas (`router.use(authMiddleware)`).

| Método | Rota | Comportamento |
|---|---|---|
| GET | `/api/v1/orders` | Pedidos do usuário do token, com `items` e `payment`, ordenados por `createdAt desc`. `{ data }`. |
| POST | `/api/v1/orders` | Checkout (sem body). 201 `{ data: order, message: 'Pedido criado com sucesso' }`. |

Erros do checkout: 400 `INVALID_PAYLOAD` (carrinho vazio); 409 `CONFLICT` (produto não existe mais ou está inativo); 409 `OUT_OF_STOCK`; 500 (falha na transação).

## Dependências
- **Internas:** `cart/cartService` (getCart), `products/productRepository` (findById), `config/database`, `config/redis` (require dentro da função), `config/logger`, `utils/AppError`, `middlewares/authMiddleware`.
- **Externas:** Prisma, redis.

## Módulos relacionados
- `cart`: fonte dos itens; o carrinho é apagado ao final.
- `products`: validação e decremento de estoque.
- `payments`: o checkout cria o registro `Payment` (ARQ-02); `payments` atualiza `Order.status` (ARQ-01).
- `users`: `Order.userId`.

## Pontos de entrada
`orderRoutes.js` (montado em `server.js:53`).

## Fluxos importantes — checkout (`orderService.checkout`)
1. `cartService.getCart(userId)`.
2. Para cada item: `productRepository.findById` → valida existência, `stock >= quantity` e `status === 'ACTIVE'` → preço atual (`Number(product.price)`).
3. `orderRepository.createOrderTransaction` (`prisma.$transaction`):
   - para cada item: `stock decrement` → relê o produto → se `stock < 0`, lança erro (a transação faz rollback);
   - `order.create` com `status: 'PENDING'`, `totalValue` e `items.create`;
   - `payment.create` com `status: 'AWAITING_CONFIRMATION'`.
4. Qualquer erro na transação → log + `AppError 500`.
5. `redisClient.del('cart:default:<userId>')`.
6. `logger.info('Evento Emitido: order.created ...')`.

## Arquivos críticos
- `orderRepository.js`: transação de estoque, pedido e pagamento.
- `orderService.js`: validações e orquestração.

## Observações técnicas e débitos
- **ARQ-02:** o repository cria `payments`, tabela de outro módulo.
- **ARQ-03:** usa o repository de `products` e manipula a chave Redis do `cart` diretamente (tenant `default` fixo).
- **DAD-02:** a limpeza do carrinho acontece fora da transação e o `POST` não é idempotente, o que permite pedido duplicado. As validações fazem uma consulta por item (N+1) e são repetidas dentro da transação só para o estoque.
- **DAD-05:** `totalValue` é calculado em ponto flutuante.
- A validação de estoque fora da transação (passo 2) é apenas uma pré-checagem; a proteção real é o decremento com a checagem de `stock < 0` dentro da transação.
- O erro de estoque dentro da transação é convertido em 500 genérico, em vez de 409 `OUT_OF_STOCK`.
- **ARQ-09:** o pedido é criado com `tenantId` padrão.
- **ARQ-10:** o evento `order.created` é só log e não é gravado quando `NODE_ENV` ≠ `development`.
- Não há endpoint de detalhe (`GET /orders/:id`) nem de cancelamento de pedido (cancelar só é possível via `payments`).
