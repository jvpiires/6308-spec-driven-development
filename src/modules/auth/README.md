# Módulo `auth`

> Base: commit `b99a0b2` · Global: [Arquitetura](../../../docs/ARQUITETURA_DO_SISTEMA.md) · [Objetivo — F1](../../../docs/OBJETIVO_DO_SISTEMA.md)

## Objetivo
Permitir que visitantes criem conta e se autentiquem, recebendo tokens JWT.

## Responsabilidade principal
Registro, verificação de credenciais e emissão de access/refresh tokens. Não gerencia dados de perfil (isso é do módulo `users`).

## Funcionalidades existentes
| Método | Rota | Validação | Resposta |
|---|---|---|---|
| POST | `/api/v1/auth/register` | `name` não vazio; `email` válido; `password` ≥ 6 caracteres | 201 `{ data: { user: {id,name,email,role}, tokens: {accessToken, refreshToken} }, meta: { timestamp } }` · 409 `CONFLICT` se o e-mail já existir · 400 `INVALID_PAYLOAD` |
| POST | `/api/v1/auth/login` | `email` válido; `password` não vazio | 200 (mesmo formato) · 401 "Credenciais inválidas" |

Regras implementadas:
- Senha com hash `bcrypt` de custo 10.
- Access token `{ sub: userId, role }`; refresh token `{ sub: userId }`.
- Refresh token gravado no Redis em `refresh:<userId>` com TTL fixo de 7 dias (sobrescrito a cada login).
- Login responde com a mesma mensagem para usuário inexistente e senha errada.

## Dependências
- **Internas:** `users/userRepository` (`findByEmail`, `create`), `utils/token`, `utils/AppError`, `config/redis`, `middlewares/validateRequest`.
- **Externas:** `bcrypt`, `express-validator`, `redis`.

## Módulos relacionados
- `users`: dono da tabela `users` (ARQ-05: o acesso deveria passar pelo `userService`).
- `middlewares/authMiddleware`: valida os tokens emitidos aqui.

## Pontos de entrada
`authRoutes.js` (montado em `server.js:49`, **sem** `authMiddleware`).

## Fluxos importantes
`register`: busca e-mail → hash → cria usuário → gera tokens → `SET refresh:<id>` → responde.
`login`: busca e-mail → `bcrypt.compare` → gera tokens → `SET refresh:<id>` → responde.

## Arquivos críticos
- `authService.js`: regras de registro e login.
- `authValidators.js`: único exemplo de validação no projeto; serve de modelo para os outros módulos.

## Observações técnicas e débitos
- **SEC-01 (crítico):** `register` aceita `role` do body (`authService.js:9,22`), permitindo criar usuários `ADMIN`. Correção: ignorar `role` no registro.
- **SEC-05:** não existem `/refresh` nem `/logout`; o refresh token armazenado nunca é lido nem validado.
- **SEC-06:** sem rate limiting no login.
- **DAD-09:** TTL do Redis (7 dias) desacoplado de `JWT_REFRESH_EXPIRATION`.
- Se o Redis estiver indisponível, o registro cria o usuário no banco e **depois** falha no `SET`: o cliente recebe erro, mas a conta já existe.
- O `email` não é normalizado (maiúsculas/minúsculas geram contas distintas).
