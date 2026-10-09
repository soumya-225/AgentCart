# AgentCart

AgentCart is an AI-powered commerce platform for two sides of a marketplace:

- **Merchants** use AI agents to improve checkout conversion, recommend complementary products, and launch inventory campaigns.
- **AI buyers** discover merchants, read machine-readable catalogs, select products within a budget, and complete bounded purchases.

The project is a demonstration of **agentic commerce**: an AI system can discover a store, reason about products, request payment, and continue or recover from the workflow while every important action is audited and subject to safety rules.

## What the project demonstrates

### Merchant-side agents

- **Conversational checkout agent**: understands shopping messages, searches the catalog, applies coupons, creates orders, and creates Razorpay payment links.
- **Upsell and cross-sell agent**: finds higher-value products in the same category and complementary products in other categories, then creates bundle offers.
- **Campaign agent**: detects slow-moving or high-margin inventory, generates a promotion, creates a coupon, and publishes a payment link.

### Buyer-side agent

The autonomous buyer agent can:

1. Discover merchants and compare available deals.
2. Read products from a structured catalog.
3. Select products according to an objective and INR budget.
4. Start an x402-style checkout.
5. Simulate or execute a Razorpay payment.
6. Record the order and payment.
7. Reject an over-budget purchase and search for cheaper alternatives.

### Safety and reliability

Money-moving actions can be checked by a central safety service. It provides:

- Spending caps for autonomous actions.
- Human approval for high-value actions.
- Pre-execution and post-execution audit records.
- Plain-English explanations for agent decisions.
- Recovery from expired payment links.
- Webhook-driven order and inventory updates.

### Merchant dashboard

<!-- TODO: Add the customer dashboard screenshot at docs/screenshots/customer-dashboard.png -->
![alt text](<Screenshot from 2026-10-10 01-31-46.png>)

### Merchant Campaigns
![alt text](<Screenshot from 2026-10-10 01-30-28.png>)



### Customer dashboard

<!-- TODO: Add the merchant dashboard screenshot at docs/screenshots/merchant-dashboard.png -->
![alt text](<Screenshot from 2026-10-10 01-38-07.png>)

## Architecture

```mermaid
flowchart TD
    UI[React customer and merchant apps] --> API[Express API]
    API --> CHECKOUT[Checkout agent]
    API --> UPSELL[Upsell agent]
    API --> CAMPAIGN[Campaign agent]
    API --> BUYER[Autonomous buyer agent]
    CHECKOUT --> SAFETY[Safety service]
    UPSELL --> SAFETY
    CAMPAIGN --> SAFETY
    BUYER --> SAFETY
    SAFETY --> DB[(PostgreSQL via Prisma)]
    CHECKOUT --> PAY[Razorpay service]
    BUYER --> PAY
    PAY --> RZP[Razorpay test API or simulator]
    RZP --> WEBHOOK[Razorpay webhooks]
    WEBHOOK --> API
```

The frontend is a control surface. Core pricing, inventory, payment, and safety decisions are made on the server.

## Project structure

```text
RazorAgent/
├── client/                         React/Vite frontend
│   └── src/
│       ├── components/             Payment, approval, and navigation components
│       ├── context/                Merchant and customer authentication
│       └── pages/                  Dashboards and agent experiences
├── server/                         Express backend
│   ├── prisma/schema.prisma         Database models
│   └── src/
│       ├── agents/                  Checkout, buyer, upsell, campaign, and LLM code
│       ├── config/                  Database and environment configuration
│       ├── middleware/              JWT and merchant authentication
│       ├── routes/                  REST, protocol, webhook, and agent endpoints
│       ├── services/                Safety, Razorpay, SBMD, and registry services
│       └── utils/                   Shared campaign utilities
├── docker-compose.yml               PostgreSQL and API containers
├── scripts/start-dev.sh             Docker startup and health-check script
├── start-all.js                     Local server + frontend launcher
└── package.json                     Root development scripts
```

## Technology

| Area | Technology |
| --- | --- |
| Frontend | React 18, Vite, TailwindCSS |
| Backend | Node.js, Express, ECMAScript modules |
| Database | PostgreSQL |
| ORM | Prisma |
| AI | OpenAI API, normally `gpt-4o` |
| Payments | Razorpay Node SDK and a local simulator |
| Authentication | JWT and bcryptjs |
| Protocol surfaces | ACP-style Agent Card, Schema.org JSON-LD, x402-style payment challenge |

The AI layer is optional for local development. Without `OPENAI_API_KEY`, the agents use deterministic fallback logic so the application remains demonstrable.

## Running the project

### Prerequisites

- Node.js 18 or newer
- npm
- Docker Desktop or Docker Engine with Compose, for the recommended setup

### Recommended: Docker database and API

From the repository root:

```bash
npm run install:all
npm run dev:docker
```

The startup script:

1. Frees ports `5000` and `5173` when possible.
2. Starts PostgreSQL and the API with Docker Compose.
3. Pushes the Prisma schema and seeds the database from the API container.
4. Waits for `GET /api/health` to succeed.
5. Starts the Vite frontend.

Open these URLs after startup:

- Application: http://localhost:5173
- API health: http://localhost:5000/api/health
- ACP Agent Card: http://localhost:5000/.well-known/agent.json
- JSON-LD catalog: http://localhost:5000/api/catalog

Stop the frontend with `Ctrl+C`. Stop the containers with:

```bash
docker compose down
```

Useful Docker commands:

```bash
docker compose ps
docker compose logs -f app
docker compose logs -f db
```

PostgreSQL is exposed on host port `5433` by [docker-compose.yml](docker-compose.yml), while the API container connects to PostgreSQL internally as `db:5432`.

### Local Node.js processes

Use this path when PostgreSQL is already running locally on port `5433` and `server/.env` points to it:

```bash
npm run install:all
npm run db:push
npm run db:seed
npm start
```

The root `npm start` runs the backend and frontend together. To run them separately:

```bash
npm run server
npm run client
```

The server runs on port `5000`; Vite runs on port `5173` and proxies `/api` and `/.well-known` to the server.

## Environment configuration

Create `server/.env`. Never commit this file or place real credentials in the README.

```env
PORT=5000
DATABASE_URL="postgresql://postgres:postgres@localhost:5433/razoragent_db"
JWT_SECRET="replace-with-a-local-secret"

# Optional. Without this, deterministic agent fallbacks are used.
OPENAI_API_KEY=""
OPENAI_MODEL="gpt-4o"

# Optional Razorpay test credentials. Empty values use the local simulator.
RAZORPAY_KEY_ID=""
RAZORPAY_KEY_SECRET=""
RAZORPAY_WEBHOOK_SECRET="replace-with-a-webhook-secret"

CLIENT_URL="http://localhost:5173"
SBMD_ENABLED="true"
```

If credentials have ever been shared publicly, rotate them before using them again. The included Razorpay simulator is sufficient for local demos and does not require payment credentials.

## Important workflows

### 1. ACP-style merchant discovery

An AI buyer begins with:

```text
GET /.well-known/agent.json
```

The response describes the merchant, capabilities, supported protocols, catalog URL, checkout URL, settlement URL, currency, and payment provider. This is the machine-readable entry point for discovering what the merchant can do.

The implementation is in [protocol.routes.js](server/src/routes/protocol.routes.js).

### 2. Structured catalog discovery

```text
GET /api/catalog
GET /api/catalog/search?q=headphones&max_price=5000
```

`/api/catalog` returns Schema.org JSON-LD with product names, SKUs, prices, currency, categories, stock, and offers. This allows agents to consume catalog information without scraping the visual storefront.

### 3. x402-style checkout and settlement

The buyer sends products to:

```text
POST /api/protocol/checkout
```

The server resolves product prices from the database, creates an order, and responds with HTTP `402 Payment Required`. Important response headers include:

```text
X-Payment-Required: true
X-Payment-Amount: <amount in INR>
X-Payment-Currency: INR
X-Payment-Provider: razorpay
X-Payment-Order-Id: <Razorpay order ID>
X-Payment-Timeout: 300
```

After payment, the buyer sends the settlement request to:

```text
POST /api/protocol/pay
```

The server records the payment and marks the order as paid. The public route demonstrates the x402-style challenge and settlement contract. The built-in buyer playground also contains an internal simulation path, so its demo does not always make an HTTP request to its own checkout endpoint.

### 4. Customer checkout

```text
POST /api/agents/chat
POST /api/agents/checkout
```

The checkout agent resolves prices and inventory on the server, applies active coupons, creates a Razorpay order and payment link, and persists the order. Customer-initiated checkout bypasses autonomous-agent spending caps because the customer is the person authorizing the purchase.

### 5. Autonomous buyer budget control

```text
POST /api/agents/buyer/run
```

The buyer agent tracks its objective, budget, selected products, payment, and remaining budget. When a follow-up purchase exceeds the available amount, the safety layer blocks it and searches the catalog for alternatives within the remaining budget.

### 6. Campaign generation

```text
POST /api/agents/campaign/analyze
POST /api/agents/campaign/run
GET  /api/agents/campaign/insights
```

The campaign agent identifies slow-moving inventory using sales count and stock levels, calculates margins from price and cost, generates a coupon, creates a promotional payment link, and stores its reasoning with the campaign.

### 7. Webhook fulfillment and recovery

```text
POST /api/webhooks/razorpay
```

The webhook handler processes payment authorization, capture, failure, and expired-link events. Captured payments can update order status, create payment records, decrement inventory, increment sales counts, and append an audit entry. Expired payment links trigger a replacement link with a 30-minute validity window.

## Safety model

The central implementation is [safetyService.js](server/src/services/safetyService.js). Its `interceptAction` flow is:

1. Read the merchant or session spending limit.
2. Block an autonomous action if it exceeds the remaining budget.
3. Create an `ApprovalRequest` when the amount exceeds the human approval threshold.
4. Write a `PENDING` audit entry before execution.
5. Execute the approved action.
6. Write a `SUCCESS` or `FAILED` audit entry afterwards.

The database stores the governance information in `AuditLog` and `ApprovalRequest`. This gives the platform bounded autonomy: agents may reason and propose actions, but a policy layer controls money movement.

## Payments and SBMD

[razorpayService.js](server/src/services/razorpayService.js) provides one payment interface with two backends:

- **Razorpay test mode** when credentials are configured.
- **Local simulator mode** when credentials are absent. It generates test order, payment, customer, and payment-link IDs.

The SBMD service in [sbmdService.js](server/src/services/sbmdService.js) supports a saved-instrument flow for frictionless recurring payments. With a real Razorpay token it attempts a recurring payment and capture. In local demonstrations it can use a merchant spending reserve as a simulated payment cap.


## API reference

| Area | Endpoints |
| --- | --- |
| Health | `GET /api/health` |
| Discovery | `GET /.well-known/agent.json` |
| Catalog | `GET /api/catalog`, `GET /api/catalog/search` |
| Protocol payments | `POST /api/protocol/checkout`, `POST /api/protocol/pay` |
| Conversational checkout | `POST /api/agents/chat`, `POST /api/agents/checkout` |
| Recommendations | `POST /api/agents/upsell`, `POST /api/agents/bundle-checkout` |
| Campaigns | `POST /api/agents/campaign/analyze`, `POST /api/agents/campaign/run`, `GET /api/agents/campaign/insights` |
| Autonomous buyer | `POST /api/agents/buyer/run` |
| Webhooks | `POST /api/webhooks/razorpay` |

Authentication-protected merchant routes use a JWT in the `Authorization` header. Customer-facing routes can use optional merchant authentication and are also usable in the demo without a merchant login.

## Development commands

```bash
npm run install:all       # Install server and client dependencies
npm run db:push           # Apply Prisma schema to the configured database
npm run db:seed           # Seed merchants, products, and demo data
npm run server            # Start the API in watch mode
npm run client            # Start the Vite frontend
npm run dev:docker        # Start Docker services, seed, and frontend
npm run db:studio         # Open Prisma Studio through the server package
make test                 # Run the SBMD test script
```

## Demo and production limitations

This repository is designed for a local demonstration and challenge prototype. Before production use, the following areas need additional hardening:

- Verify payment settlement directly with Razorpay before marking an x402 order paid.
- Add idempotency keys for checkout, settlement, and webhook processing.
- Use a proper secret manager and rotate all exposed credentials.
- Restrict CORS instead of allowing every origin.
- Add rate limiting and stronger authorization checks around merchant resources.
- Add transactional inventory reservation to prevent concurrent overselling.
- Add automated tests for protocol, payment, safety, and webhook flows.
