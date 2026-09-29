# `src/modules` — Módulos de domínio

> Base: commit `b99a0b2` · Global: [Arquitetura do Sistema](../../docs/ARQUITETURA_DO_SISTEMA.md) · [Objetivo do Sistema](../../docs/OBJETIVO_DO_SISTEMA.md)

## Objetivo
Agrupar a lógica de negócio da API, com um diretório por domínio.

## Estrutura padrão de um módulo
```
<dominio>/
├── <x>Routes.js       # express.Router: caminhos + middlewares (auth, adminOnly, validators)
├── <x>Controller.js   # lê req, chama service, formata { data }
├── <x>Service.js      # regras de negócio, lança AppError
├── <x>Repository.js   # (opcional) acesso Prisma
└── <x>Validators.js   # (opcional; hoje só em auth) regras express-validator
```

## Módulos
| Módulo | Prefixo | Tem repository | Depende de outros módulos |
|---|---|---|---|
| [auth](./auth/README.md) | `/api/v1/auth` | não | users |
| [users](./users/README.md) | `/api/v1/users` | sim | — |
| [categories](./categories/README.md) | `/api/v1/categories` | sim | — |
| [products](./products/README.md) | `/api/v1/products` | sim | — |
| [cart](./cart/README.md) | `/api/v1/cart` | não (Redis + Prisma direto) | products (tabela) |
| [orders](./orders/README.md) | `/api/v1/orders` | sim | cart, products, payments (tabela) |
| [payments](./payments/README.md) | `/api/v1/payments` | sim | orders (tabela) |

## Grafo de dependências
```
auth ──► users
orders ──► cart (service)      orders ──► products (repository)   orders ──► payments (tabela, ARQ-02)
cart ──► products (tabela, ARQ-04)
payments ──► orders (tabela, ARQ-01)
```
Não há ciclos por `require`. Existe, porém, um **ciclo de dados** entre `orders` e `payments`: cada um escreve na tabela do outro.

## Checklist para criar um novo módulo (SDD)
1. Escrever o `README.md` do módulo (seções deste padrão) **antes** do código.
2. Criar `Routes/Controller/Service` (+ `Repository` se houver tabela própria, + `Validators` se houver input).
3. Montar o router em `src/server.js` sob `/api/v1/<dominio>`.
4. Consumir outros domínios só pelos services deles (regra R4).
5. Atualizar a Arquitetura §5, §6.3 e §12 e o Objetivo §5.
