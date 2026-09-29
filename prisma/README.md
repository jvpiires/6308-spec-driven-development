# `prisma` — Modelo de dados e migrações

> Base: commit `b99a0b2` · Global: [Arquitetura do Sistema §7](../docs/ARQUITETURA_DO_SISTEMA.md)

## Objetivo
Definir o esquema do PostgreSQL (fonte da verdade do banco) e versionar suas migrações.

## Responsabilidade principal
`schema.prisma` descreve entidades, enums e relações; `migrations/` contém o SQL aplicado; `../prisma.config.ts` indica à CLI o schema, a pasta de migrações e a `DATABASE_URL` (Prisma 7).

## Funcionalidades existentes
- Enums: `Role (USER, ADMIN)`, `ProductStatus (ACTIVE, INACTIVE)`, `OrderStatus (PENDING, PAID, CANCELED)`.
- Modelos → tabelas: `User→users`, `Category→categories`, `Product→products`, `Order→orders`, `OrderItem→order_items`, `Payment→payments`.
- Restrições: `users.email` único; `products.sku` único; `payments.orderId` único (1:1 com pedido); índice `products.name`.
- FKs: `categories.parentId` `ON DELETE SET NULL`; todas as outras `ON DELETE RESTRICT`, `ON UPDATE CASCADE`.
- Migração única: `20260929025231_migrate`.

## Dependências
- **Externas:** `prisma` (CLI, devDependency), `@prisma/client`, `dotenv` (em `prisma.config.ts`).
- **Internas:** consumido por `src/config/database.js`.

## Módulos relacionados (dono de cada tabela)
| Tabela | Módulo dono | Outros módulos que escrevem/leem |
|---|---|---|
| `users` | users | auth (via `userRepository`) |
| `categories` | categories | products (FK) |
| `products` | products | cart (leitura direta, ARQ-04); orders (leitura via repository e decremento de estoque) |
| `orders`, `order_items` | orders | payments (escrita de status, ARQ-01) |
| `payments` | payments | orders (criação no checkout, ARQ-02) |

## Pontos de entrada
- `npm run migrate` → `prisma migrate dev`
- `npm run studio` → `prisma studio`
- Container: `npx prisma generate && npx prisma migrate deploy`

## Fluxos importantes
Alteração de modelo: editar `schema.prisma` → `npm run migrate` (gera uma nova pasta em `migrations/`) → atualizar os READMEs dos módulos donos e a Arquitetura §7.

## Arquivos críticos
`schema.prisma` e `migrations/*/migration.sql` (não editar migrações já aplicadas).

## Observações técnicas e débitos
- **DAD-04:** `Payment.status` é `String`, não enum.
- **ARQ-09:** `tenantId` (default `"default"`) em `users`, `products` e `orders`, mas **não** em `categories`, `order_items` e `payments`; nenhuma query filtra por ele.
- **DAD-08:** `SET NULL` em `categories.parentId` promove subcategorias a raiz quando a categoria pai é excluída.
- Não há `CHECK (stock >= 0)`; a proteção contra estoque negativo é feita em código (`orderRepository`).
- O comentário sobre full-text search no `Product` não tem implementação; a busca usa `contains` + `insensitive` (ILIKE).
- **OPS-05:** `.gitignore` ignora `/src/generated/prisma`, mas o generator não define `output` (usa `node_modules/.prisma`).
