# `src/config` — Infraestrutura compartilhada

> Base: commit `b99a0b2` · Global: [Arquitetura do Sistema](../../docs/ARQUITETURA_DO_SISTEMA.md)

## Objetivo
Criar e exportar, como singletons, os clientes de infraestrutura usados por toda a API.

## Responsabilidade principal
Ser o **único** ponto de criação de conexões com PostgreSQL (Prisma), Redis e do logger (regra R5).

## Funcionalidades existentes
| Arquivo | Exporta | Comportamento |
|---|---|---|
| `database.js` | `prisma` (`PrismaClient`) | Instancia `PrismaPg` com `DATABASE_URL` e passa como `adapter` (obrigatório no Prisma 7). |
| `redis.js` | `redisClient` | `createClient` com `redis://REDIS_HOST:REDIS_PORT` (padrão `localhost:6379`); registra os eventos `error`/`connect` no logger; **conecta imediatamente no `require`**. |
| `logger.js` | `logger` (winston) | Níveis `error, warn, info, http, debug`; nível `debug` em `development` e `warn` nos demais; transports: Console, `logs/error.log` (só erro), `logs/all.log`. |

## Dependências
- **Externas:** `@prisma/client`, `@prisma/adapter-pg` (usa `pg`), `redis`, `winston`.
- **Internas:** `redis.js` → `logger.js`.
- **Variáveis de ambiente:** `DATABASE_URL`, `REDIS_HOST`, `REDIS_PORT`, `NODE_ENV`.

## Módulos relacionados (consumidores)
- `database.js`: repositories de `users`, `categories`, `products`, `orders`, `payments`; e, fora do padrão, `cart/cartService` (ARQ-04) e `payments/paymentService` (ARQ-01).
- `redis.js`: `auth/authService`, `cart/cartService`, `products/productService`, `orders/orderService` (ARQ-03).
- `logger.js`: `server.js`, `middlewares/errorHandler`, `middlewares/httpLogger`, `orders/orderService`, `payments/paymentService`.

## Pontos de entrada
Nenhum endpoint. São usados por `require('../../config/<arquivo>')`.

## Fluxos importantes
- **Inicialização:** o primeiro `require('./config/redis')` (feito indiretamente por `server.js` → `authRoutes` → `authService`) abre a conexão Redis antes do `app.listen`.
- **Prisma 7:** a URL do banco vem de `prisma.config.ts` na CLI (migrações) e de `DATABASE_URL` no runtime (adapter). As duas precisam apontar para o mesmo banco.

## Arquivos críticos
- `database.js`: mudar o adapter ou a versão do Prisma afeta todos os repositories.
- `redis.js`: sem ele, login, carrinho e checkout param.

## Observações técnicas e débitos
- **OPS-01:** a IIFE de conexão do Redis não tem `catch`; se a conexão inicial falhar, ocorre rejeição não tratada. Não existe `disconnect`/`$disconnect` em shutdown.
- **OPS-02:** `winston.format.colorize({ all: true })` também é aplicado aos arquivos, gravando códigos ANSI em `logs/*.log`; o caminho `logs/` é relativo ao `cwd`.
- **ARQ-10:** em produção (nível `warn`), `logger.info` é descartado, inclusive os "eventos" de domínio.
- **OPS-03:** com o `.env` atual (`localhost`), o serviço `api` do docker-compose não alcança `postgres`/`redis`.
