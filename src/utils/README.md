# `src/utils` — Utilitários

> Base: commit `b99a0b2` · Global: [Arquitetura do Sistema](../../docs/ARQUITETURA_DO_SISTEMA.md)

## Objetivo
Oferecer funções e classes puras reutilizáveis, sem estado de infraestrutura.

## Responsabilidade principal
Padronizar erros de aplicação (`AppError`) e encapsular a emissão/validação de JWT (`token`).

## Funcionalidades existentes
| Arquivo | Exporta | Descrição |
|---|---|---|
| `AppError.js` | `class AppError extends Error` | `new AppError(message, statusCode = 500, code = 'INTERNAL_SERVER_ERROR', details = {})`; marca `isOperational = true` (campo **não** lido por ninguém). |
| `token.js` | `generateAccessToken(userId, role)` | JWT `{ sub, role }` assinado com `JWT_ACCESS_SECRET`, expira em `JWT_ACCESS_EXPIRATION`. |
| | `generateRefreshToken(userId)` | JWT `{ sub }` com `JWT_REFRESH_SECRET` / `JWT_REFRESH_EXPIRATION`. |
| | `verifyAccessToken(token)` | `jwt.verify` com o segredo de acesso (lança exceção se inválido). |
| | `verifyRefreshToken(token)` | **Não usado (órfão)**. |

## Dependências
- **Externas:** `jsonwebtoken`.
- **Internas:** nenhuma.
- **Variáveis de ambiente:** `JWT_ACCESS_SECRET`, `JWT_ACCESS_EXPIRATION`, `JWT_REFRESH_SECRET`, `JWT_REFRESH_EXPIRATION`.

## Módulos relacionados
- `AppError`: `server.js`, `middlewares/authMiddleware`, `middlewares/validateRequest`, todos os services e as rotas de `products`/`categories`.
- `token`: `auth/authService` (geração) e `middlewares/authMiddleware` (validação).

## Pontos de entrada
`require('../utils/AppError')`, `require('../utils/token')`.

## Fluxos importantes
Emissão no login/registro → validação a cada requisição protegida. O payload `sub` é o `User.id`, que os módulos usam como `req.user.sub`.

## Arquivos críticos
`token.js`: alterar o payload (`sub`, `role`) quebra `authMiddleware` e todas as checagens de papel/propriedade.

## Observações técnicas e débitos
- **SEC-05:** `verifyRefreshToken` existe, mas nenhum endpoint o usa.
- Se as variáveis `JWT_*` não estiverem definidas, `jwt.sign` lança exceção em runtime (não há validação de configuração na inicialização).
