# Módulo `categories`

> Base: commit `b99a0b2` · Global: [Arquitetura](../../../docs/ARQUITETURA_DO_SISTEMA.md) · [Objetivo — F2/F3](../../../docs/OBJETIVO_DO_SISTEMA.md)

## Objetivo
Organizar o catálogo em categorias hierárquicas (pai/filho).

## Responsabilidade principal
Dono da tabela `categories`. CRUD, com leitura pública e escrita restrita a ADMIN.

## Funcionalidades existentes
| Método | Rota | Acesso | Comportamento |
|---|---|---|---|
| GET | `/api/v1/categories` | público | Todas as categorias, cada uma com `children` (1 nível). |
| GET | `/api/v1/categories/:id` | público | Categoria + `children`; 404 se não existir. |
| POST | `/api/v1/categories` | ADMIN | Cria com o body recebido (`name`, `parentId?`); 201. |
| PUT | `/api/v1/categories/:id` | ADMIN | Checa existência (404) e atualiza. |
| DELETE | `/api/v1/categories/:id` | ADMIN | 204; 409 `CONFLICT` se houver produtos (P2003). |

## Dependências
- **Internas:** `config/database`, `utils/AppError`, `middlewares/authMiddleware`.
- **Externas:** Prisma.

## Módulos relacionados
- `products`: `Product.categoryId` → `categories` (FK `RESTRICT`); é usado como filtro em `GET /products?categoryId=`.

## Pontos de entrada
`categoryRoutes.js` (montado em `server.js:52`). Os GETs são declarados antes de `router.use(authMiddleware)`.

## Fluxos importantes
Exclusão: `prisma.category.delete` → se houver FK de produto (P2003) → 409. As subcategorias têm `parentId` definido como `NULL` pelo banco.

## Arquivos críticos
`categoryRoutes.js` (ordem público/protegido) e `categoryService.js`.

## Observações técnicas e débitos
- **ARQ-06:** `adminOnly` definido localmente (cópia idêntica em `products`).
- **SEC-04:** sem validators; o body vai direto ao Prisma.
- **DAD-06:** `parentId` inexistente no create/update e exclusão de id inexistente (P2025) → 500.
- **DAD-08:** a listagem retorna filhas também no nível raiz (duplicadas dentro de `children` do pai); a exclusão do pai promove as filhas a raiz; não há proteção contra ciclos (`parentId` = próprio id ou descendente).
- **ARQ-10:** o comentário "Webhook: category.updated -> Simulado" não tem implementação.
- Não possui `tenantId` (diferente de `products`), ver ARQ-09.
- **ARQ-07:** métodos não mapeados (ex.: `PATCH`) respondem 401 em vez de 404 para usuários não autenticados.
