# Exchange

A real-time cryptocurrency trading platform modeled on Binance, built as a set of decoupled microservices communicating over Redis. It features a matching engine with live order books, real-time depth and trade streaming over WebSockets, candlestick (k-line) charts, and persistent trade history.

**Tech stack:** Next.js, React, TypeScript, Node.js, Express, WebSockets, Redis, TimescaleDB (PostgreSQL), Docker.

## Architecture

The system is split into independent services that talk to each other through Redis queues and pub/sub channels. This keeps the matching engine single-threaded and deterministic while everything else scales horizontally.

```
                ┌──────────┐
                │ frontend │  Next.js UI (charts, order book, trades)
                └────┬─────┘
           REST      │      WebSocket
        ┌────────────┴────────────┐
        ▼                         ▼
   ┌─────────┐               ┌─────────┐
   │   api   │               │   ws    │  streams depth/trade updates
   └────┬────┘               └────▲────┘
        │ push orders             │ pub/sub
        ▼                         │
   ┌──────────────── Redis ───────┴────────────┐
   │  queues: "messages", "db_processor"        │
   └────┬───────────────────────────┬──────────┘
        ▼                           ▼
   ┌─────────┐                 ┌─────────┐
   │ engine  │  matching       │   db    │  persists trades
   └─────────┘  engine         └────┬────┘
        ▲                           ▼
   ┌─────────┐                 ┌─────────────┐
   │   mm    │  market maker   │ TimescaleDB │
   └─────────┘                 └─────────────┘
```

### Services

| Service | Description | Port |
| --- | --- | --- |
| **`api`** | Express REST API. Accepts orders and serves order book depth, recent trades, k-lines, and tickers. Pushes order requests onto the Redis `messages` queue and awaits the engine's response. | 3000 |
| **`engine`** | The core matching engine. Consumes orders from the Redis `messages` queue, maintains the in-memory order book, matches trades, and publishes results. Supports state snapshots (`WITH_SNAPSHOT=true`) for recovery on restart. | — |
| **`ws`** | WebSocket server. Lets clients subscribe to channels (e.g. depth, trades) and pushes real-time updates via Redis pub/sub. | 3001 |
| **`db`** | Database processor. Consumes trade events from the Redis `db_processor` queue and writes them to TimescaleDB. Also refreshes materialized views used for k-line data. | — |
| **`mm`** | A market maker bot that places orders against the API to provide liquidity for testing/demo. | — |
| **`frontend`** | Next.js trading UI: candlestick charts (`lightweight-charts`), live order book depth, trade feed, markets page, and a swap/order form. | 3000 (Next dev) |

## Getting Started

### Prerequisites

- Node.js (18+)
- Docker & Docker Compose

### 1. Start infrastructure

Spin up Redis and TimescaleDB:

```bash
cd docker
docker-compose up -d
```

This starts:
- **Redis** on `localhost:6379`
- **TimescaleDB** on `localhost:5432` (db: `my_database`, user: `your_user`, password: `your_password`)

### 2. Install & run each backend service

Each service is a standalone Node package. In separate terminals:

```bash
# Matching engine
cd engine && npm install && npm run dev

# REST API
cd api && npm install && npm run dev

# WebSocket server
cd ws && npm install && npm run dev

# Database processor
cd db && npm install && npm run seed:db   # seed schema / tables
cd db && npm run dev

# Market maker (optional, provides liquidity)
cd mm && npm install && npm run dev
```

### 3. Run the frontend

```bash
cd frontend
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

## API Endpoints

Base URL: `http://localhost:3000/api/v1`

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/order` | Place a new order |
| `GET` | `/depth` | Current order book depth |
| `GET` | `/trades` | Recent trades |
| `GET` | `/klines` | Candlestick (k-line) data |
| `GET` | `/tickers` | Ticker information |

## Development

Each service is TypeScript with its own `tsconfig.json`. Common scripts:

- `npm run build` — compile TypeScript to `dist/`
- `npm run dev` — build and run
- `npm test` — run tests (the engine uses [Vitest](https://vitest.dev))

## License

ISC
