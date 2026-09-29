# Módulo `users`

> Base: commit `b99a0b2` · Global: [Arquitetura](../../../docs/ARQUITETURA_DO_SISTEMA.md) · [Objetivo — F7](../../../docs/OBJETIVO_DO_SISTEMA.md)

## Objetivo
Gerenciar os dados cadastrais dos usuários já registrados.

## Responsabilidade principal
Dono da tabela `users`. Consulta, atualização e exclusão com regras de propriedade (o próprio usuário ou um admin).

## Funcionalidades existentes
Todas exigem autenticação (`router.use(authMiddleware)`).

| Método | Rota | Regra de acesso | Resposta |
|---|---|---|---|
| GET | `/api/v1/users/:id` | USER só o próprio `id`; ADMIN qualquer um (checagem no **controller**) | 200 `{ data: user sem password }` · 403 `{ error: { message: 'Forbidden' } }` · 404 |
| PUT | `/api/v1/users/:id` | USER só a si mesmo; ADMIN qualquer um (checagem no **service**); `role` descartado se quem pede não é ADMIN | 200 `{ data: user sem password }` · 403 · 404 |
| DELETE | `/api/v1/users/:id` | Idem PUT | 204 |

Repository: `create`, `findByEmail`, `findById`, `findAll(skip, take)` (**sem uso**), `update`, `delete`.

## Dependências
- **Internas:** `config/database`, `utils/AppError`, `middlewares/authMiddleware`.
- **Externas:** Prisma.

## Módulos relacionados
- `auth`: usa `userRepository.create` e `findByEmail` (ARQ-05).
- `orders`: `Order.userId` referencia `users` (FK `RESTRICT`).

## Pontos de entrada
`userRoutes.js` (montado em `server.js:55`).

## Fluxos importantes
Atualização: valida o acesso → confirma que o usuário existe → remove `role` se quem pede não for admin → `prisma.user.update(data)` → remove `password` da resposta.

## Arquivos críticos
`userService.js` (regras de acesso) e `userRepository.js` (compartilhado com `auth`).

## Observações técnicas e débitos
- **SEC-03 (alto):** o body vai direto ao Prisma. `password` é gravado **sem hash** (o comentário no código reconhece), e `email`, `tenantId`, `createdAt` etc. podem ser alterados.
- **SEC-04:** sem validators.
- **ARQ-07:** autorização dividida entre controller (GET) e service (PUT/DELETE). No controller, `req.params.id || req.user.sub` tem um ramo inalcançável, porque a rota sempre tem `:id`. Não existe `/users/me`.
- **ARQ-08:** o 403 do GET não segue o envelope do `errorHandler` (não tem `code`).
- **DAD-06:** e-mail duplicado no update (P2002), exclusão de usuário inexistente (P2025) ou com pedidos (P2003) → 500.
- **SEC-05:** após a exclusão, os tokens emitidos continuam válidos até expirar e `refresh:<id>` permanece no Redis.
- Não há endpoint de listagem de usuários, apesar de existir `findAll`.
