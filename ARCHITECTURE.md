# Arquitetura — CryptoPay Brasil

## 1. Visão geral

CryptoPay Brasil é composto por três blocos principais que se comunicam por
HTTP/REST, WebSocket e uma fila de cache/pub-sub (Redis):

```
┌─────────────────────┐        HTTPS/WSS        ┌──────────────────────┐
│                      │ ───────────────────────▶│                      │
│   apps/web (Next.js) │                          │   apps/api (NestJS)  │
│   PWA - SSR/CSR       │◀───────────────────────│   REST + WebSocket    │
│                      │        JSON/JWT          │                      │
└─────────────────────┘                          └──────────┬───────────┘
                                                              │
                          ┌───────────────────────────────────┼───────────────────────────────┐
                          │                                    │                                 │
                 ┌────────▼────────┐                 ┌─────────▼─────────┐            ┌──────────▼──────────┐
                 │   PostgreSQL     │                 │       Redis        │            │  Provedores externos │
                 │  (dados persist.)│                 │ (cache, sessão,     │            │  Blockchain / Câmbio  │
                 │                  │                 │  rate limit, filas) │            │  (somente leitura)    │
                 └──────────────────┘                 └─────────────────────┘            └──────────────────────┘
```

Toda a stack roda em containers Docker orquestrados por `docker-compose`,
atrás de um Nginx que termina TLS (Let's Encrypt) e faz proxy reverso para o
frontend e a API.

## 2. Decisão: modelo não-custodial

O sistema **nunca armazena, processa ou solicita chaves privadas ou seed
phrases**. A "importação de carteira" citada nos requisitos é implementada
como cadastro de **endereço público** (ou leitura via QR Code de um endereço
público). Isso permite:

- Consulta de saldo
- Histórico de transações
- Favoritos e monitoramento de endereços

Sem que a plataforma assuma o risco, a responsabilidade regulatória e a
superfície de ataque de um custodiante de ativos de terceiros. Qualquer
evolução futura para custódia real (guarda de chaves, assinatura de
transações, movimentação de fundos) exige:

- Licenciamento como Instituição de Pagamento junto ao Banco Central do Brasil
- Programa de PLD/FT (Prevenção à Lavagem de Dinheiro e Financiamento ao Terrorismo)
- HSM (Hardware Security Module) ou MPC wallet para custódia de chaves
- Auditoria de segurança externa (pentest, SOC 2)

Esses pontos estão fora do escopo de código e são decisões de negócio/jurídicas.

## 3. Arquitetura de provedores de blockchain (plugável)

Para permitir adicionar novas redes sem alterar o núcleo do sistema, o backend
implementa o padrão **Strategy/Adapter**:

```
apps/api/src/blockchain/
├── interfaces/
│   └── blockchain-provider.interface.ts   # contrato único
├── providers/
│   ├── bitcoin.provider.ts
│   ├── ethereum.provider.ts
│   ├── litecoin.provider.ts
│   ├── dogecoin.provider.ts
│   ├── bnb-chain.provider.ts
│   ├── polygon.provider.ts
│   ├── tron.provider.ts
│   ├── erc20-token.provider.ts   # base para USDT/USDC na rede ETH/BSC/Polygon
│   └── trc20-token.provider.ts   # base para USDT na rede TRON
├── blockchain-provider.factory.ts # resolve o provider certo por moeda/rede
└── blockchain.module.ts
```

Todo provider implementa a mesma interface:

```typescript
interface IBlockchainProvider {
  readonly network: BlockchainNetwork;
  getBalance(address: string): Promise<BalanceResult>;
  getTransactions(address: string, options?: PaginationOptions): Promise<TransactionResult[]>;
  isValidAddress(address: string): boolean;
}
```

Adicionar uma nova moeda/rede = criar uma nova classe que implementa
`IBlockchainProvider` e registrá-la na factory. Nenhum outro módulo do sistema
precisa ser alterado (Open/Closed Principle).

### Provedores por rede (Módulo 5 detalhará a implementação completa)

| Rede | Provedor de dados | Tipo |
|---|---|---|
| Bitcoin (BTC) | Blockstream Esplora API | Nativa, pública |
| Litecoin (LTC) | Blockcypher API | Nativa, pública |
| Dogecoin (DOGE) | Blockcypher API | Nativa, pública |
| Ethereum (ETH) | Etherscan API | Nativa, pública |
| BNB Chain (BNB) | BscScan API | Nativa, pública |
| Polygon (MATIC) | PolygonScan API | Nativa, pública |
| TRON (TRX) | TronGrid API | Nativa, pública |
| USDT/USDC (ERC-20) | Etherscan (leitura de contrato) | Token sobre ETH |
| USDT (TRC-20) | TronGrid (leitura de contrato) | Token sobre TRON |

Todas as chamadas são **read-only** (sem uso de chave privada).

## 4. Módulos do backend (NestJS)

```
apps/api/src/
├── auth/               # login, registro, refresh token, 2FA
├── users/               # perfil, configurações
├── wallets/              # carteiras importadas (endereço público)
├── blockchain/            # provedores plugáveis (seção 3)
├── transactions/           # histórico normalizado + cache
├── quotes/                 # cotação BRL (câmbio)
├── favorites/               # endereços/moedas favoritas
├── notifications/            # notificações in-app + push
├── admin/                     # gestão de usuários, logs, relatórios
├── audit/                      # trilha de auditoria
├── common/                      # guards, interceptors, filters, decorators
├── config/                       # configuração tipada (env)
├── database/                      # PrismaService, módulo global
└── websocket/                      # gateway de tempo real (saldo, notificações)
```

## 5. Segurança — camadas aplicadas

| Camada | Mecanismo |
|---|---|
| Senhas | Argon2id (hash), nunca texto plano |
| Dados sensíveis em repouso | AES-256-GCM (ex: segredo do 2FA) |
| Sessão | JWT de acesso (curta duração) + Refresh Token (rotacionável, armazenado com hash no banco) |
| 2FA | TOTP (RFC 6238), compatível com Google Authenticator/Authy |
| Cabeçalhos HTTP | Helmet + CSP restritiva |
| Força bruta | Rate limiting por IP e por usuário (Redis) |
| CSRF | Double submit cookie em rotas que usam cookie de sessão |
| XSS | Sanitização de entrada, CSP, escaping automático do React |
| SQL Injection | Prisma (queries parametrizadas, sem SQL cru por padrão) |
| Auditoria | Log estruturado de eventos sensíveis (login, alteração de dados, 2FA, admin) |
| Backup | Dump automatizado do PostgreSQL, retenção configurável, envio para S3 |

## 6. Cloud e infraestrutura

- **AWS RDS** — PostgreSQL gerenciado em produção
- **AWS S3** — armazenamento de backups e assets estáticos
- **AWS ElastiCache / Redis próprio em container** — cache e sessões
- **Cloudflare** — proxy, DNS, proteção DDoS, cache de assets estáticos
- **Nginx** — proxy reverso interno + terminação TLS via Let's Encrypt
- **Docker Compose** — orquestração local e em VPS
- **GitHub Actions** — build, testes e deploy automatizado

## 7. Fluxo de dados — consulta de saldo (exemplo)

1. Usuário importa endereço público no frontend (`apps/web`)
2. Frontend chama `POST /wallets` na API
3. API valida o endereço via `IBlockchainProvider.isValidAddress()`
4. API persiste a carteira vinculada ao usuário (PostgreSQL)
5. Ao abrir o dashboard, frontend chama `GET /wallets/:id/balance`
6. API verifica cache no Redis (TTL curto, ex: 30s) → se ausente, consulta o
   provedor externo da rede correspondente
7. Resultado é convertido para BRL via `quotes` (cache próprio, TTL maior)
8. Resposta é enviada ao frontend e, quando há mudança relevante, replicada
   via WebSocket para atualização em tempo real

## 8. Diagramas

Os diagramas de entidade-relacionamento do banco de dados serão entregues no
**Módulo 2**, junto ao schema Prisma completo. O diagrama de serviços
detalhado (deployment) será entregue no **Módulo 14**, junto à infraestrutura
de produção.
