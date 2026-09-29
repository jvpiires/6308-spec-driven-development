# Módulo `payments`

> Base: commit `b99a0b2` · Global: [Arquitetura](../../../docs/ARQUITETURA_DO_SISTEMA.md) · [Objetivo — F6](../../../docs/OBJETIVO_DO_SISTEMA.md)

## Objetivo
Registrar e mudar o estado do pagamento de um pedido.

## Responsabilidade principal
Dono da tabela `payments` (1:1 com `orders`). **Não integra nenhum gateway real**: a confirmação é feita chamando a própria API.

## Funcionalidades existentes
Todas autenticadas (`router.use(authMiddleware)`).

| Método | Rota | Entrada | Comportamento |
|---|---|---|---|
| POST | `/api/v1/payments` | `{ orderId }` | Retorna o pagamento existente do pedido ou cria um com `AWAITING_CONFIRMATION`. 201. |
| POST | `/api/v1/payments/:id/confirm` | header `Idempotency-Key` (obrigatório) | 400 sem o header; 404 se não existir; se já estiver `PAID`, retorna sem alterar; senão, transação: `payment.status = 'PAID'` e `order.status = 'PAID'`; log `order.paid`. |
| POST | `/api/v1/payments/:id/cancel` | — | 404 se não existir; transação: `payment.status = 'CANCELED'` e `order.status = 'CANCELED'`; `{ data: { message } }`. |

Repository: `create`, `findByOrderId`, `findById` (inclui `order`), `updateStatus` (**sem uso**).

## Dependências
- **Internas:** `config/database` (**acesso direto a `order`**), `config/logger`, `utils/AppError`, `middlewares/authMiddleware`.
- **Externas:** Prisma.

## Módulos relacionados
- `orders`: o checkout cria o `Payment` (ARQ-02); este módulo altera `Order.status` (ARQ-01).

## Pontos de entrada
`paymentRoutes.js` (montado em `server.js:54`).

## Fluxos importantes
Estados observados no código: `AWAITING_CONFIRMATION` → `PAID` | `CANCELED`. Não há bloqueio de transições a partir de estados finais.

## Arquivos críticos
`paymentService.js`: altera o status do pedido, ou seja, determina se um pedido é considerado pago.

## Observações técnicas e débitos
- **SEC-02 (crítico):** nenhuma rota verifica se o pedido/pagamento pertence a `req.user.sub`. Qualquer usuário autenticado pode confirmar ou cancelar pagamentos de terceiros.
- **SEC-07:** `Idempotency-Key` não é persistido nem comparado.
- **DAD-03:** é possível cancelar um pagamento `PAID` e confirmar um `CANCELED`; cancelar não devolve estoque (o comentário "Devolver Estoque (opcional, mas recomendado)" não está implementado).
- **DAD-04:** o status é `String`, sem enum.
- **DAD-06:** `POST /payments` com `orderId` inexistente ou ausente → erro do Prisma → 500.
- **ARQ-01:** escreve em `orders` via `prisma` direto (o comentário na linha 2 reconhece o atalho).
- `externalId` nunca é preenchido. **Hipótese:** reservado para o id/clientSecret de um gateway (ex.: Stripe).
- **OPS-05:** `paymentRepository.updateStatus` é órfão.
