# CryptoPay Brasil

Plataforma fintech Web PWA para consulta de saldos, histórico de transações e
gestão de carteiras cripto (não-custodial), com conversão em tempo real para
BRL, painel administrativo e infraestrutura pronta para produção.

> **Modelo de custódia:** este sistema **não guarda chaves privadas**. Toda
> consulta de saldo/histórico é feita a partir do **endereço público** da
> carteira, via provedores de blockchain (leitura apenas). Isso é uma decisão
> de arquitetura deliberada — ver [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md).

## Stack

| Camada | Tecnologias |
|---|---|
| Frontend | Next.js 15, React 19, TypeScript, TailwindCSS, PWA, React Query, Zustand, Framer Motion, Chart.js, React Hook Form, Zod |
| Backend | NestJS, TypeScript, Prisma ORM, PostgreSQL, Redis, JWT + Refresh Token, Swagger, WebSocket |
| Infra | Docker, Docker Compose, Nginx, Let's Encrypt, GitHub Actions |
| Cloud | AWS (S3, RDS), Cloudflare |
| Segurança | Argon2, AES-256, Helmet, Rate Limit, CSRF, CSP, 2FA (TOTP), Auditoria |
| Blockchain | Arquitetura de provedores plugáveis — BTC, ETH, LTC, DOGE, USDT, USDC, BNB, MATIC, TRX |

## Estrutura do monorepo

```
cryptopay-brasil/
├── apps/
│   ├── api/                 # Backend NestJS
│   └── web/                 # Frontend Next.js
├── infra/
│   ├── docker/              # Dockerfiles de cada serviço
│   ├── nginx/                # Configuração de proxy reverso e SSL
│   └── scripts/              # Scripts de deploy, backup, restore
├── .github/workflows/        # Pipelines de CI/CD
├── docs/                     # Documentação técnica e diagramas
├── docker-compose.yml         # Ambiente de desenvolvimento
├── docker-compose.prod.yml    # Ambiente de produção
└── .env.example
```

## Roadmap de entrega (por módulos)

Este projeto está sendo construído e documentado em módulos incrementais.
Cada módulo é entregue com código completo, sem trechos omitidos.

- [x] **Módulo 1** — Arquitetura geral, estrutura de pastas, Docker Compose base, variáveis de ambiente
- [ ] **Módulo 2** — Banco de dados: schema Prisma, migrations, seed, índices, relacionamentos
- [ ] **Módulo 3** — Backend: Autenticação (JWT + Refresh Token + Argon2 + 2FA)
- [ ] **Módulo 4** — Backend: Usuários, Perfil, Configurações
- [ ] **Módulo 5** — Backend: Wallets/Blockchain (provedores plugáveis)
- [ ] **Módulo 6** — Backend: Transações, Cotação BRL, Favoritos, Notificações, WebSocket
- [ ] **Módulo 7** — Backend: Painel Admin (usuários, logs, relatórios)
- [ ] **Módulo 8** — Backend: Segurança transversal (Helmet, Rate Limit, CSRF, CSP, Guards, Interceptors, Filters)
- [ ] **Módulo 9** — Frontend: Setup Next.js 15 + PWA + Zustand + React Query + Design system
- [ ] **Módulo 10** — Frontend: Autenticação (login, cadastro, recuperação, 2FA)
- [ ] **Módulo 11** — Frontend: Dashboard, Scanner QR Code, Importação de carteira
- [ ] **Módulo 12** — Frontend: Histórico, gráficos, favoritos, notificações
- [ ] **Módulo 13** — Frontend: Painel Admin
- [ ] **Módulo 14** — Infra: Nginx, SSL/Let's Encrypt, Docker produção
- [ ] **Módulo 15** — CI/CD: GitHub Actions
- [ ] **Módulo 16** — Testes: unitários, integração, E2E
- [ ] **Módulo 17** — Documentação final: manuais e diagramas

## Início rápido (desenvolvimento)

```bash
cp .env.example .env
docker compose up -d
```

Serviços expostos em desenvolvimento:

| Serviço | URL |
|---|---|
| Frontend | http://localhost:3000 |
| API | http://localhost:3001 |
| Swagger | http://localhost:3001/docs |
| PostgreSQL | localhost:5432 |
| Redis | localhost:6379 |

## Licença

Uso interno / proprietário — CryptoPay Brasil.
