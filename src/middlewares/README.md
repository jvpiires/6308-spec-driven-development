# `src/middlewares` — Middlewares HTTP

> Base: commit `b99a0b2` · Global: [Arquitetura do Sistema](../../docs/ARQUITETURA_DO_SISTEMA.md)

## Objetivo
Reunir os middlewares Express reutilizados pelos módulos e por `server.js`.

## Responsabilidade principal
Tratar preocupações transversais: autenticação, validação de payload, tratamento de erro e log HTTP.

## Funcionalidades existentes
| Arquivo | Função | Usado por |
|---|---|---|
| `authMiddleware.js` | Lê `Authorization`, extrai o token (`split(' ')[1]`), valida com `verifyAccessToken` e define `req.user = { sub, role, iat, exp }`. Sem header → 401 "Token de acesso não fornecido"; token inválido/expirado → 401. | `server.js` (cart), `users`, `orders`, `payments`, `products` e `categories` (rotas de escrita) |
| `validateRequest.js` | Lê `validationResult` do express-validator e, se houver erros, lança `AppError('Dados inválidos', 400, 'INVALID_PAYLOAD', { campo: msg })`. | Só `auth/authRoutes` |
| `errorHandler.js` | Handler final: `statusCode`/`code` do erro (padrão 500/`INTERNAL_SERVER_ERROR`); status ≥ 500 → `logger.error` + stack; os demais → `logger.warn`; responde `{ error: { code, message, details, stack? } }` (stack só em `development`). | `server.js` (último `app.use`) |
| `httpLogger.js` | morgan `:method :url :status :res[content-length] - :response-time ms` → `logger.http`; desligado fora de `development`. | `server.js` |
| `requestLogger.js` | `console.log` de método/URL. | **Ninguém: órfão (OPS-05)** |

## Dependências
- **Externas:** `express-validator`, `morgan`.
- **Internas:** `utils/token`, `utils/AppError`, `config/logger`.

## Módulos relacionados
Todos os módulos em `src/modules/` (via rotas) e `server.js`.

## Pontos de entrada
Funções `(req, res, next)` ou `(err, req, res, next)`, exportadas por `module.exports`.

## Fluxos importantes
- `authMiddleware` e `validateRequest` **lançam** `AppError` de forma síncrona. Isso funciona porque o Express 5 encaminha a exceção para o `errorHandler`; migrar para outro framework ou para o Express 4 exigiria usar `next(err)`.

## Arquivos críticos
`authMiddleware.js` (toda a proteção de rotas) e `errorHandler.js` (contrato de erro da API).

## Observações técnicas e débitos
- **SEC-05:** o esquema `Bearer` não é verificado (`"Xyz <token>"` também passa); não há checagem de usuário ainda existente.
- **ARQ-06:** falta um middleware de autorização por papel; `adminOnly` está duplicado em `products` e `categories`. Diretriz: criar `authorize.js` aqui.
- **DAD-06:** o `errorHandler` não traduz erros do Prisma (P2002/P2003/P2025), que chegam como 500.
- **OPS-05:** `requestLogger.js` pode ser removido.
