# 🏗️ Architecture

> **Tech stack rationale + system architecture + data flows for stox.**

---

## 📌 Table of Contents

- [Tech Stack](#-tech-stack)
- [Tech Stack Rationale](#-tech-stack-rationale)
- [High-Level Architecture](#-high-level-architecture)
- [Data Flow: Real-Time Price Update](#-data-flow-real-time-price-update)
- [Data Flow: Order Execution](#-data-flow-order-execution)
- [Deployment Architecture](#-deployment-architecture)

---

## 🛠️ Tech Stack

### Complete Technology Breakdown

| Layer              | Technology              | Version          | Purpose                      |
| ------------------ | ----------------------- | ---------------- | ---------------------------- |
| **Frontend**       | Next.js                 | 14+              | React framework with SSR/SSG |
|                    | React                   | 18+              | UI library                   |
|                    | Tailwind CSS            | 3+               | Utility-first styling        |
|                    | Lucide React            | Latest           | Icon library                 |
|                    | Recharts / Chart.js     | Latest           | Data visualization           |
|                    | Socket.IO Client        | 4+               | Real-time communication      |
|                    | Axios                   | Latest           | HTTP client                  |
|                    | React Hook Form         | Latest           | Form management              |
|                    | Zustand / Context API   | Latest           | State management             |
| **Backend**        | Node.js                 | 18+              | Runtime environment          |
|                    | Express.js              | 4+               | Web framework                |
|                    | Prisma ORM              | 5+               | Database ORM                 |
|                    | Socket.IO               | 4+               | WebSocket server             |
|                    | BullMQ                  | Latest           | Background job queue         |
|                    | ioredis                 | Latest           | Redis client                 |
|                    | jsonwebtoken            | Latest           | JWT creation/verification    |
|                    | bcryptjs                | Latest           | Password hashing             |
|                    | Helmet                  | Latest           | Security headers             |
|                    | express-rate-limit      | Latest           | Rate limiting                |
|                    | express-validator / zod | Latest           | Input validation             |
|                    | cors                    | Latest           | CORS handling                |
|                    | winston / pino          | Latest           | Logging                      |
| **Database**       | PostgreSQL              | 14+              | Primary database             |
|                    | Prisma Migrate          | Latest           | Schema migrations            |
|                    | Prisma Seed             | Latest           | Seed data                    |
| **Cache & Queue**  | Redis                   | 7+               | Caching + BullMQ backend     |
|                    | BullMQ                  | Latest           | Job queue                    |
| **Market Data**    | Finnhub                 | API v1           | US stocks, forex, news       |
|                    | Binance                 | Public API       | Crypto prices, order book    |
|                    | Yahoo Finance           | Fallback         | Historical data fallback     |
| **Authentication** | JWT                     | Access + Refresh | Stateless auth               |
|                    | bcrypt                  | 10 rounds        | Password hashing             |
| **Real-Time**      | Socket.IO               | 4+               | Rooms, namespaces            |
|                    | WebSocket (ws)          | Latest           | Upstream market data         |
| **Deployment**     | Vercel                  | —                | Frontend hosting             |
|                    | Render / Railway        | —                | Backend hosting              |
|                    | Neon / Supabase         | —                | Managed PostgreSQL           |
|                    | Upstash                 | —                | Managed Redis                |
| **DevOps**         | Docker                  | Latest           | Containerization             |
|                    | Docker Compose          | Latest           | Local orchestration          |
|                    | GitHub Actions          | —                | CI/CD                        |
|                    | ESLint + Prettier       | Latest           | Code quality                 |
| **Documentation**  | Swagger / OpenAPI       | —                | API documentation            |
|                    | Postman                 | —                | API testing                  |
|                    | Mermaid                 | —                | Architecture diagrams        |

---

## 🎯 Tech Stack Rationale

| Choice                       | Why                                                                          |
| ---------------------------- | ---------------------------------------------------------------------------- |
| **Next.js over plain React** | SSR for SEO, file-based routing, API routes, built-in optimization           |
| **Express over Fastify**     | Larger ecosystem, more learning resources, team familiarity                  |
| **PostgreSQL over MongoDB**  | Financial data is inherently relational; ACID compliance critical for trades |
| **Prisma over raw SQL**      | Type safety, migrations, seed data, excellent DX                             |
| **Redis for caching**        | Sub-millisecond reads, pub/sub for scaling, BullMQ backend                   |
| **Socket.IO over raw WS**    | Rooms, namespaces, reconnection handling built-in                            |
| **BullMQ over node-cron**    | Retries, scheduling, persistence, dashboard                                  |
| **Finnhub + Binance**        | Dual asset coverage; Finnhub for stocks, Binance for crypto                  |
| **JWT with refresh tokens**  | Stateless, scalable, industry standard                                       |

---

## 🏛️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Browser    │  │  Browser    │  │  Browser    │  │  Mobile     │        │
│  │  (Trader)   │  │  (Trader)   │  │  (Admin)    │  │  (Future)   │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
└─────────┼────────────────┼────────────────┼────────────────┼───────────────┘
          │                │                │                │
          ▼                ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PRESENTATION LAYER                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                      Next.js Frontend                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │Dashboard │  │ Trading  │  │Portfolio │  │ Leader-  │                   │
│  │          │  │  View    │  │Analytics │  │  board   │                   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │  Alerts  │  │  Copy    │  │  Social  │  │  Admin   │                   │
│  │          │  │ Trading  │  │   Feed   │  │  Panel   │                   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘                   │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
              ┌──────────┐    ┌──────────┐    ┌──────────┐
              │ REST API │    │Socket.IO │    │  Static  │
              │  (HTTPS) │    │  (WSS)   │    │  Assets  │
              └────┬─────┘    └────┬─────┘    └──────────┘
                   │               │
                   ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           APPLICATION LAYER                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                    Node.js + Express.js Backend                             │
│  ┌────────────────────────────────────────────────────────────┐            │
│  │                    Middleware Chain                         │            │
│  │  Helmet → CORS → Rate Limit → Logger → Auth → Validation   │            │
│  └────────────────────────────────────────────────────────────┘            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │  Routes  │→ │Controllers│→ │ Services │→ │  Models  │                  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘                  │
└──────┬──────────────────┬──────────────────┬──────────────────┬────────────┘
       │                  │                  │                  │
       ▼                  ▼                  ▼                  ▼
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────────┐
│ PostgreSQL │  │   Redis    │  │  BullMQ    │  │  External APIs     │
│            │  │            │  │  Workers   │  │                    │
│ - Users    │  │ - Cache    │  │ - Orders   │  │ - Finnhub (Stocks) │
│ - Portfolios│ │ - Pub/Sub  │  │ - Alerts   │  │ - Binance (Crypto) │
│ - Orders   │  │ - Sessions │  │ - Reports  │  │ - Yahoo (Fallback) │
│ - Trades   │  │ - Rate Lim │  │ - Emails   │  │                    │
│ - Alerts   │  │            │  │            │  │                    │
└────────────┘  └────────────┘  └────────────┘  └────────────────────┘
```

---

## 📡 Data Flow: Real-Time Price Update

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        REAL-TIME PRICE FLOW                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. EXTERNAL SOURCES                                                        │
│  ┌─────────────────┐         ┌─────────────────┐                          │
│  │    Finnhub      │         │    Binance      │                          │
│  │  WebSocket      │         │  WebSocket      │                          │
│  │  (US Stocks)    │         │  (Crypto)       │                          │
│  └────────┬────────┘         └────────┬────────┘                          │
│           │                           │                                    │
│           ▼                           ▼                                    │
│  2. INGESTION SERVICE                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  MarketDataService                                               │      │
│  │  - Connects to both WebSockets                                   │      │
│  │  - Normalizes data format                                        │      │
│  │  - Handles reconnection                                          │      │
│  │  - Manages subscriptions                                         │      │
│  └──────────────────────────┬──────────────────────────────────────┘      │
│                             │                                              │
│                             ▼                                              │
│  3. REDIS CACHE + PUB/SUB                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  SET price:AAPL { price: 175.50, ts: ... } EX 5                  │      │
│  │  PUBLISH price-updates { symbol, price, volume, timestamp }      │      │
│  └──────────────────────────┬──────────────────────────────────────┘      │
│                             │                                              │
│                             ▼                                              │
│  4. SOCKET.IO BROADCAST                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  io.to(`stock:AAPL`).emit('price-update', data)                  │      │
│  │  io.to(`stock:BTCUSDT`).emit('price-update', data)               │      │
│  └──────────────────────────┬──────────────────────────────────────┘      │
│                             │                                              │
│                             ▼                                              │
│  5. CLIENT RECEIVES                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  socket.on('price-update', (data) => {                           │      │
│  │    updateChart(data);                                            │      │
│  │    checkAlerts(data);                                            │      │
│  │    updatePortfolio(data);                                        │      │
│  │  });                                                             │      │
│  └─────────────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 📡 Data Flow: Order Execution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ORDER EXECUTION FLOW                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. USER PLACES ORDER                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  POST /api/v1/orders                                             │      │
│  │  { symbol: "AAPL", type: "LIMIT", side: "BUY",                   │      │
│  │    quantity: 10, limitPrice: 140.00 }                             │      │
│  └──────────────────────────┬──────────────────────────────────────┘      │
│                             │                                              │
│                             ▼                                              │
│  2. VALIDATION & PERSISTENCE                                                │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  - Validate input (zod)                                          │      │
│  │  - Check portfolio balance                                       │      │
│  │  - Create Order record (status: PENDING)                         │      │
│  │  - Return 201 Created                                            │      │
│  └──────────────────────────┬──────────────────────────────────────┘      │
│                             │                                              │
│                             ▼                                              │
│  3. BACKGROUND JOB (BullMQ)                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  orderQueue.add('check-order', { orderId })                      │      │
│  │  - Runs every 5 seconds                                          │      │
│  │  - Fetches pending orders                                        │      │
│  │  - Compares limit price with current price                       │      │
│  └──────────────────────────┬──────────────────────────────────────┘      │
│                             │                                              │
│                             ▼                                              │
│  4. ORDER EXECUTION                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  IF currentPrice <= limitPrice:                                  │      │
│  │    - BEGIN TRANSACTION                                           │      │
│  │    - Update portfolio balance                                    │      │
│  │    - Create Trade record                                         │      │
│  │    - Update Order status to EXECUTED                             │      │
│  │    - COMMIT                                                      │      │
│  └──────────────────────────┬──────────────────────────────────────┘      │
│                             │                                              │
│                             ▼                                              │
│  5. NOTIFICATION & REAL-TIME UPDATE                                         │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │  - notificationQueue.add('order-executed', { orderId })          │      │
│  │  - io.to(`user:${userId}`).emit('order-executed', order)         │      │
│  │  - Send email notification                                       │      │
│  └─────────────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🚀 Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PRODUCTION DEPLOYMENT                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         VERCEL (Frontend)                            │   │
│  │  Next.js App — SSR + CDN + Edge Functions                            │   │
│  │  URL: https://stox.vercel.app                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      RENDER / RAILWAY (Backend)                      │   │
│  │  Express API + Socket.IO + BullMQ Workers                            │   │
│  │  URL: https://stox-api.onrender.com                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                    ┌───────────────┼───────────────┐                        │
│                    ▼               ▼               ▼                        │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐           │
│  │  NEON / SUPABASE │ │     UPSTASH      │ │   CLOUDINARY     │           │
│  │  (PostgreSQL)    │ │     (Redis)      │ │   (File Storage) │           │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘           │
└─────────────────────────────────────────────────────────────────────────────┘
```
