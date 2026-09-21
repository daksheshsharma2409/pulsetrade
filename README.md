# 📈 stox

> **A real-time paper trading simulator for US stocks and cryptocurrency. Practice trading with virtual money, real market data, and zero financial risk.**

---

## 📖 What Is stox?

stox is a full-stack web application where users practice buying and selling **US stocks** (via Finnhub) and **cryptocurrencies** (via Binance) using **virtual money**. No real money is involved. Users compete on leaderboards, set price alerts, place advanced orders, copy other traders, and analyze their portfolio performance.

Think of it as a **video game for trading** — realistic, competitive, and completely safe.

---

## 🎯 Why stox?

| For Users                          | For Evaluators                     |
| ---------------------------------- | ---------------------------------- |
| Risk-free trading practice         | Real-time WebSocket architecture   |
| Real market data (stocks + crypto) | Complex background job processing  |
| Competitive gamification           | Redis caching with measurable wins |
| Portfolio analytics                | Event-driven notification pipeline |
| Learning tool for finance          | Full production-grade deployment   |

### What Makes It Unique

- **Dual asset classes** — stocks AND crypto in one platform
- **Real-time price streaming** — sub-second updates via WebSocket
- **Advanced order types** — limit, stop-loss, take-profit
- **Copy trading** — social trading with real-time trade mirroring
- **Background job engine** — order execution, alerts, reports all async
- **Measurable caching** — Redis with latency benchmarks shown in dashboard

---

## ✨ Core Features

### Trading

- Market orders (buy/sell at current price)
- Limit orders (buy/sell at target price)
- Stop-loss orders (auto-sell to prevent losses)
- Take-profit orders (auto-sell at target gain)
- Short selling (profit when prices drop)
- Multiple portfolios

### Real-Time

- Live price streaming (stocks + crypto)
- Live portfolio value updates
- Live order execution notifications
- Order book depth (Binance)
- Trade tape (Binance)

### Social

- Follow other traders
- Copy trading (auto-mirror trades)
- Social feed
- Leaderboards
- Trading leagues

### Analytics

- P&L heatmap (calendar view)
- Risk metrics (Sharpe ratio, max drawdown)
- Asset allocation charts
- Performance vs benchmarks
- Win rate analysis

### Gamification

- Achievement badges
- Trading competitions
- Sentiment voting

---

## 🛠️ Tech Stack

| Layer             | Technology                                                              |
| ----------------- | ----------------------------------------------------------------------- |
| **Frontend**      | Next.js 14, React 18, Tailwind CSS, Socket.IO Client, Recharts          |
| **Backend**       | Node.js 18, Express.js, Socket.IO, BullMQ, Prisma ORM                   |
| **Database**      | PostgreSQL 14+                                                          |
| **Cache & Queue** | Redis 7+                                                                |
| **Auth**          | JWT (access + refresh), bcrypt                                          |
| **Market Data**   | Finnhub (stocks), Binance (crypto)                                      |
| **Deployment**    | Vercel (frontend), Render/Railway (backend), Neon (DB), Upstash (Redis) |

---

## 🏗️ High-Level Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Next.js    │────▶│  Express    │────▶│ PostgreSQL  │
│  Frontend   │◀────│  Backend    │◀────│   Prisma    │
└─────────────┘     └──────┬──────┘     └─────────────┘
       ▲                   │
       │                   ▼
       │            ┌─────────────┐
       │            │    Redis    │
       │            │  Cache+Queue│
       │            └──────┬──────┘
       │                   │
       │                   ▼
       │            ┌─────────────┐     ┌─────────────┐
       └────────────│  Socket.IO  │◀────│  Finnhub +  │
                    │  Real-Time  │     │  Binance    │
                    └─────────────┘     └─────────────┘
```

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for full details.

---

## 📚 Documentation Index

| Document                                      | Description                                             |
| --------------------------------------------- | ------------------------------------------------------- |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md)       | Tech stack rationale + system architecture + data flows |
| [FEATURES.md](docs/FEATURES.md)               | Complete feature list + signature challenges            |
| [DATABASE.md](docs/DATABASE.md)               | Prisma schema + ER diagram                              |
| [API.md](docs/API.md)                         | API conventions + all endpoints                         |
| [REALTIME.md](docs/REALTIME.md)               | Socket.IO namespaces, rooms, events                     |
| [BACKGROUND_JOBS.md](docs/BACKGROUND_JOBS.md) | BullMQ queues + workers                                 |
| [CACHING.md](docs/CACHING.md)                 | Redis caching strategy + benchmarks                     |
| [NOTIFICATIONS.md](docs/NOTIFICATIONS.md)     | Event-driven notification pipeline                      |
| [SECURITY.md](docs/SECURITY.md)               | Auth, RBAC, ABAC, middleware, security                  |
| [MARKET_DATA.md](docs/MARKET_DATA.md)         | Finnhub + Binance integration                           |
| [FRONTEND.md](docs/FRONTEND.md)               | Frontend architecture + folder structure                |
| [DEPLOYMENT.md](docs/DEPLOYMENT.md)           | Deployment guide                                        |
| [PROJECT_PLAN.md](docs/PROJECT_PLAN.md)       | Team division, timeline, metrics, future scope          |
| [SETUP.md](docs/SETUP.md)                     | Local development setup                                 |

---

## 🚀 Quick Start

```bash
# 1. Clone repository
git clone https://github.com/your-team/stox.git
cd stox

# 2. Install backend dependencies
cd backend && npm install

# 3. Install frontend dependencies
cd ../frontend && npm install

# 4. Set up environment variables
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env.local

# 5. Start PostgreSQL and Redis
docker-compose up -d postgres redis

# 6. Run migrations and seed
cd backend
npx prisma migrate dev
npx prisma db seed

# 7. Start backend (terminal 1)
npm run dev

# 8. Start workers (terminal 2)
npm run workers

# 9. Start frontend (terminal 3)
cd ../frontend && npm run dev
```

**Access Points:**

- Frontend → http://localhost:3000
- Backend API → http://localhost:5000/api/v1
- Prisma Studio → http://localhost:5555

Full setup guide: [docs/SETUP.md](docs/SETUP.md)

---

## 🎓 What This Project Demonstrates

✅ Full-stack development (Next.js + Express)  
✅ Real-time WebSocket architecture (Socket.IO)  
✅ Background job processing (BullMQ)  
✅ Redis caching with measurable wins  
✅ Event-driven notification pipeline  
✅ Complex database schema (Prisma + PostgreSQL)  
✅ Production-grade authentication (JWT + refresh)  
✅ Role-based + attribute-based access control  
✅ Security hardening (Helmet, rate limiting, validation)  
✅ Deployment on modern cloud platforms

---

## 🏆 Signature Challenges

| Challenge                 | Implementation                              |
| ------------------------- | ------------------------------------------- |
| **Real-Time Sync**        | Socket.IO namespaces + rooms + reconnection |
| **Background Jobs**       | BullMQ with retries + scheduled jobs        |
| **Caching**               | Redis with 95% latency improvement          |
| **Notification Pipeline** | Event-driven email + push + in-app          |
| **Dual Market Data**      | Finnhub + Binance with fallback             |
| **Complex Transactions**  | Atomic order execution with row locking     |
| **Multi-Portfolio**       | ABAC-level isolation                        |

See [docs/FEATURES.md](docs/FEATURES.md) for full details.
