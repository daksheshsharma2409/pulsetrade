# 🗄️ Database Schema

> **PostgreSQL schema with Prisma ORM for stox.**

---

## 📌 Table of Contents

- [Overview](#-overview)
- [Prisma Schema](#-prisma-schema)
- [Entity Relationship Diagram](#-entity-relationship-diagram)
- [Key Design Decisions](#-key-design-decisions)
- [Migrations & Seeding](#-migrations--seeding)

---

## 📖 Overview

stox uses **PostgreSQL** as its primary database with **Prisma ORM** for type-safe database access. The schema is designed to handle:

- Multi-portfolio trading
- Real-time orders with various types
- Full audit trail of trades
- Social features (follows, copy trading)
- Gamification (achievements, leagues)
- Analytics (performance snapshots)

---

## 📋 Prisma Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ═══════════════════════════════════════════════════════════════
// USER & AUTHENTICATION
// ═══════════════════════════════════════════════════════════════

model User {
  id              String    @id @default(uuid())
  email           String    @unique
  passwordHash    String
  name            String
  avatarUrl       String?
  role            Role      @default(TRADER)
  isVerified      Boolean   @default(false)
  isSuspended     Boolean   @default(false)

  // Notification preferences
  emailAlerts     Boolean   @default(true)
  emailDigest     Boolean   @default(true)
  pushEnabled     Boolean   @default(false)
  pushToken       String?

  // Relations
  portfolios      Portfolio[]
  refreshTokens   RefreshToken[]
  alerts          Alert[]
  achievements    UserAchievement[]
  followers       Follow[]  @relation("Following")
  following       Follow[]  @relation("Followers")
  copyTraders     CopyTrader[] @relation("Copier")
  copiedBy        CopyTrader[] @relation("Copied")
  votes           SentimentVote[]
  leagueMembers   LeagueMember[]
  notifications   Notification[]
  sessions        Session[]

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  lastLoginAt     DateTime?

  @@index([email])
  @@index([role])
}

enum Role {
  TRADER
  ADMIN
  MODERATOR
}

model RefreshToken {
  id          String    @id @default(uuid())
  token       String    @unique
  userId      String
  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  expiresAt   DateTime
  createdAt   DateTime  @default(now())
  revokedAt   DateTime?
  userAgent   String?
  ipAddress   String?

  @@index([userId])
  @@index([token])
}

model Session {
  id          String    @id @default(uuid())
  userId      String
  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  socketId    String?
  isOnline    Boolean   @default(false)
  lastSeenAt  DateTime  @default(now())

  @@index([userId])
}

// ═══════════════════════════════════════════════════════════════
// PORTFOLIO & TRADING
// ═══════════════════════════════════════════════════════════════

model Portfolio {
  id              String    @id @default(uuid())
  userId          String
  user            User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  name            String    @default("Main Portfolio")
  description     String?

  startingBalance Decimal   @default(10000) @db.Decimal(18, 8)
  cashBalance     Decimal   @default(10000) @db.Decimal(18, 8)

  isDefault       Boolean   @default(false)
  isActive        Boolean   @default(true)

  holdings        Holding[]
  orders          Order[]
  trades          Trade[]
  performance     PerformanceSnapshot[]

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([userId])
  @@unique([userId, name])
}

model Holding {
  id              String    @id @default(uuid())
  portfolioId     String
  portfolio       Portfolio @relation(fields: [portfolioId], references: [id], onDelete: Cascade)
  symbol          String
  assetType       AssetType

  quantity        Decimal   @db.Decimal(18, 8)
  averageBuyPrice Decimal   @db.Decimal(18, 8)
  totalInvested   Decimal   @db.Decimal(18, 8)

  isShort         Boolean   @default(false)

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@unique([portfolioId, symbol])
  @@index([portfolioId])
  @@index([symbol])
}

enum AssetType {
  STOCK
  CRYPTO
  FOREX
}

model Order {
  id              String    @id @default(uuid())
  portfolioId     String
  portfolio       Portfolio @relation(fields: [portfolioId], references: [id], onDelete: Cascade)

  symbol          String
  assetType       AssetType
  side            OrderSide
  type            OrderType
  status          OrderStatus @default(PENDING)

  quantity        Decimal   @db.Decimal(18, 8)

  limitPrice      Decimal?  @db.Decimal(18, 8)
  stopPrice       Decimal?  @db.Decimal(18, 8)
  executedPrice   Decimal?  @db.Decimal(18, 8)

  executedAt      DateTime?
  expiresAt       DateTime?

  notes           String?
  source          OrderSource @default(MANUAL)

  trade           Trade?

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([portfolioId])
  @@index([status])
  @@index([symbol])
  @@index([createdAt])
}

enum OrderSide {
  BUY
  SELL
}

enum OrderType {
  MARKET
  LIMIT
  STOP_LOSS
  TAKE_PROFIT
}

enum OrderStatus {
  PENDING
  EXECUTED
  CANCELLED
  EXPIRED
  REJECTED
}

enum OrderSource {
  MANUAL
  COPY_TRADE
  DCA_BOT
  LEAGUE
}

model Trade {
  id              String    @id @default(uuid())
  portfolioId     String
  portfolio       Portfolio @relation(fields: [portfolioId], references: [id], onDelete: Cascade)
  orderId         String    @unique
  order           Order     @relation(fields: [orderId], references: [id], onDelete: Cascade)

  symbol          String
  assetType       AssetType
  side            OrderSide

  quantity        Decimal   @db.Decimal(18, 8)
  price           Decimal   @db.Decimal(18, 8)
  totalValue      Decimal   @db.Decimal(18, 8)
  fees            Decimal   @default(0) @db.Decimal(18, 8)

  realizedPnl     Decimal?  @db.Decimal(18, 8)
  realizedPnlPct  Decimal?  @db.Decimal(18, 8)

  executedAt      DateTime  @default(now())

  @@index([portfolioId])
  @@index([symbol])
  @@index([executedAt])
}

// ═══════════════════════════════════════════════════════════════
// MARKET DATA
// ═══════════════════════════════════════════════════════════════

model Symbol {
  id              String    @id @default(uuid())
  symbol          String    @unique
  name            String
  assetType       AssetType
  exchange        String?
  currency        String    @default("USD")

  sector          String?
  industry        String?
  logoUrl         String?
  description     String?

  isActive        Boolean   @default(true)
  isTradable      Boolean   @default(true)

  priceHistory    PriceHistory[]
  alerts          Alert[]
  sentimentVotes  SentimentVote[]

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([symbol])
  @@index([assetType])
}

model PriceHistory {
  id              String    @id @default(uuid())
  symbolId        String
  symbol          Symbol    @relation(fields: [symbolId], references: [id], onDelete: Cascade)

  price           Decimal   @db.Decimal(18, 8)
  open            Decimal?  @db.Decimal(18, 8)
  high            Decimal?  @db.Decimal(18, 8)
  low             Decimal?  @db.Decimal(18, 8)
  close           Decimal?  @db.Decimal(18, 8)
  volume          Decimal?  @db.Decimal(18, 8)

  interval        String    @default("1m")
  timestamp       DateTime

  @@index([symbolId, timestamp])
  @@index([symbolId, interval, timestamp])
}

// ═══════════════════════════════════════════════════════════════
// ALERTS & NOTIFICATIONS
// ═══════════════════════════════════════════════════════════════

model Alert {
  id              String    @id @default(uuid())
  userId          String
  user            User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  symbolId        String
  symbol          Symbol    @relation(fields: [symbolId], references: [id], onDelete: Cascade)

  condition       AlertCondition
  targetPrice     Decimal   @db.Decimal(18, 8)
  status          AlertStatus @default(ACTIVE)

  sendEmail       Boolean   @default(true)
  sendPush        Boolean   @default(false)

  triggeredAt     DateTime?
  message         String?

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([userId])
  @@index([status])
  @@index([symbolId])
}

enum AlertCondition {
  PRICE_ABOVE
  PRICE_BELOW
  PERCENT_CHANGE_UP
  PERCENT_CHANGE_DOWN
}

enum AlertStatus {
  ACTIVE
  TRIGGERED
  EXPIRED
  CANCELLED
}

model Notification {
  id              String    @id @default(uuid())
  userId          String
  user            User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  type            NotificationType
  title           String
  message         String
  data            Json?

  isRead          Boolean   @default(false)
  readAt          DateTime?

  createdAt       DateTime  @default(now())

  @@index([userId, isRead])
  @@index([createdAt])
}

enum NotificationType {
  ORDER_EXECUTED
  PRICE_ALERT
  ACHIEVEMENT_UNLOCKED
  COPY_TRADE_EXECUTED
  LEAGUE_UPDATE
  SYSTEM
}

// ═══════════════════════════════════════════════════════════════
// SOCIAL & GAMIFICATION
// ═══════════════════════════════════════════════════════════════

model Follow {
  id              String    @id @default(uuid())
  followerId      String
  follower        User      @relation("Following", fields: [followerId], references: [id], onDelete: Cascade)
  followingId     String
  following       User      @relation("Followers", fields: [followingId], references: [id], onDelete: Cascade)

  createdAt       DateTime  @default(now())

  @@unique([followerId, followingId])
  @@index([followerId])
  @@index([followingId])
}

model CopyTrader {
  id              String    @id @default(uuid())
  copierId        String
  copier          User      @relation("Copier", fields: [copierId], references: [id], onDelete: Cascade)
  copiedId        String
  copied          User      @relation("Copied", fields: [copiedId], references: [id], onDelete: Cascade)

  allocationPct   Decimal   @default(100) @db.Decimal(5, 2)
  maxTradeSize    Decimal?  @db.Decimal(18, 8)

  isActive        Boolean   @default(true)

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@unique([copierId, copiedId])
  @@index([copierId])
  @@index([copiedId])
}

model Achievement {
  id              String    @id @default(uuid())
  code            String    @unique
  name            String
  description     String
  iconUrl         String?
  category        String

  criteria        Json

  createdAt       DateTime  @default(now())

  @@index([code])
}

model UserAchievement {
  id              String    @id @default(uuid())
  userId          String
  user            User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  achievementId   String
  achievement     Achievement @relation(fields: [achievementId], references: [id], onDelete: Cascade)

  unlockedAt      DateTime  @default(now())

  @@unique([userId, achievementId])
  @@index([userId])
}

model League {
  id              String    @id @default(uuid())
  name            String
  description     String?

  startDate       DateTime
  endDate         DateTime

  startingBalance Decimal   @default(10000) @db.Decimal(18, 8)
  maxParticipants Int?

  status          LeagueStatus @default(UPCOMING)

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([status])
  @@index([startDate])
}

enum LeagueStatus {
  UPCOMING
  ACTIVE
  COMPLETED
  CANCELLED
}

model LeagueMember {
  id              String    @id @default(uuid())
  leagueId        String
  league          League    @relation(fields: [leagueId], references: [id], onDelete: Cascade)
  userId          String
  user            User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  portfolioId     String    @unique
  rank            Int?
  finalReturn     Decimal?  @db.Decimal(10, 4)

  joinedAt        DateTime  @default(now())

  @@unique([leagueId, userId])
  @@index([leagueId])
  @@index([userId])
}

model SentimentVote {
  id              String    @id @default(uuid())
  userId          String
  user            User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  symbolId        String
  symbol          Symbol    @relation(fields: [symbolId], references: [id], onDelete: Cascade)

  sentiment       Sentiment
  createdAt       DateTime  @default(now())

  @@unique([userId, symbolId])
  @@index([symbolId])
}

enum Sentiment {
  BULLISH
  BEARISH
  NEUTRAL
}

// ═══════════════════════════════════════════════════════════════
// ANALYTICS
// ═══════════════════════════════════════════════════════════════

model PerformanceSnapshot {
  id              String    @id @default(uuid())
  portfolioId     String
  portfolio       Portfolio @relation(fields: [portfolioId], references: [id], onDelete: Cascade)

  totalValue      Decimal   @db.Decimal(18, 8)
  cashBalance     Decimal   @db.Decimal(18, 8)
  holdingsValue   Decimal   @db.Decimal(18, 8)

  dailyPnl        Decimal   @db.Decimal(18, 8)
  dailyPnlPct     Decimal   @db.Decimal(10, 4)

  snapshotDate    DateTime  @db.Date

  createdAt       DateTime  @default(now())

  @@unique([portfolioId, snapshotDate])
  @@index([portfolioId])
  @@index([snapshotDate])
}

// ═══════════════════════════════════════════════════════════════
// ADMIN & LOGGING
// ═══════════════════════════════════════════════════════════════

model AuditLog {
  id              String    @id @default(uuid())
  userId          String?
  action          String
  resource        String
  resourceId      String?
  changes         Json?
  ipAddress       String?
  userAgent       String?

  createdAt       DateTime  @default(now())

  @@index([userId])
  @@index([action])
  @@index([createdAt])
}

model SystemMetric {
  id              String    @id @default(uuid())
  metric          String
  value           Decimal   @db.Decimal(18, 8)
  metadata        Json?

  recordedAt      DateTime  @default(now())

  @@index([metric])
  @@index([recordedAt])
}
```

---

## 🔗 Entity Relationship Diagram

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│    User     │───┬───│  Portfolio  │───┬───│  Holding    │
└─────────────┘   │   └─────────────┘   │   └─────────────┘
       │          │          │          │
       │          │          ▼          │
       │          │   ┌─────────────┐   │
       │          │   │    Order    │   │
       │          │   └─────────────┘   │
       │          │          │          │
       │          │          ▼          │
       │          │   ┌─────────────┐   │
       │          │   │    Trade    │   │
       │          │   └─────────────┘   │
       │          │                     │
       ▼          ▼                     ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   Alert     │ │   League    │ │Performance  │
└─────────────┘ └─────────────┘ │  Snapshot   │
       │                        └─────────────┘
       ▼
┌─────────────┐
│Notification │
└─────────────┘

┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│    User     │───┬───│    Follow   │───┬───│    User     │
└─────────────┘   │   └─────────────┘   │   └─────────────┘
                  │                     │
                  ▼                     ▼
           ┌─────────────┐       ┌─────────────┐
           │ CopyTrader  │       │Achievement  │
           └─────────────┘       └─────────────┘
```

---

## 🎯 Key Design Decisions

| Decision                                    | Why                                                |
| ------------------------------------------- | -------------------------------------------------- |
| **UUID primary keys**                       | Globally unique; safe for distributed systems      |
| **Decimal(18, 8) for money**                | 8 decimal places for crypto precision              |
| **Cascade deletes**                         | Deleting a user removes all their data             |
| **`@@unique([portfolioId, symbol])`**       | Prevents duplicate holdings for same symbol        |
| **`@@index([status])`** on Order            | Fast filtering of pending orders                   |
| **`@@unique([portfolioId, snapshotDate])`** | One snapshot per portfolio per day                 |
| **JSONB for achievements.criteria**         | Flexible rule definitions                          |
| **Separate `PriceHistory` table**           | Time-series data stored separately from live cache |
| **`isShort` flag on Holding**               | Same table handles both long and short positions   |

---

## 🔄 Migrations & Seeding

### Running Migrations

```bash
# Create a new migration
npx prisma migrate dev --name add_new_field

# Apply migrations in production
npx prisma migrate deploy

# Reset database (dev only)
npx prisma migrate reset
```

### Seed Data

```javascript
// prisma/seed.js
import { PrismaClient } from "@prisma/client";
import bcrypt from "bcryptjs";

const prisma = new PrismaClient();

async function main() {
    // Create admin user
    const adminPassword = await bcrypt.hash("admin123", 10);
    await prisma.user.upsert({
        where: { email: "admin@stox.com" },
        update: {},
        create: {
            email: "admin@stox.com",
            name: "Admin",
            passwordHash: adminPassword,
            role: "ADMIN",
        },
    });

    // Create symbols
    const symbols = [
        { symbol: "AAPL", name: "Apple Inc.", assetType: "STOCK" },
        { symbol: "TSLA", name: "Tesla Inc.", assetType: "STOCK" },
        { symbol: "MSFT", name: "Microsoft Corp.", assetType: "STOCK" },
        { symbol: "BINANCE:BTCUSDT", name: "Bitcoin", assetType: "CRYPTO" },
        { symbol: "BINANCE:ETHUSDT", name: "Ethereum", assetType: "CRYPTO" },
    ];

    for (const sym of symbols) {
        await prisma.symbol.upsert({
            where: { symbol: sym.symbol },
            update: {},
            create: sym,
        });
    }

    // Create achievements
    const achievements = [
        {
            code: "FIRST_TRADE",
            name: "First Trade",
            description: "Execute your first trade",
            category: "TRADING",
            criteria: {},
        },
        {
            code: "PROFIT_10",
            name: "Profit Machine",
            description: "Make 10 profitable trades",
            category: "TRADING",
            criteria: { count: 10 },
        },
        {
            code: "STREAK_5",
            name: "Streak Master",
            description: "Win 5 trades in a row",
            category: "TRADING",
            criteria: { streak: 5 },
        },
        {
            code: "WHALE",
            name: "Whale",
            description: "Portfolio exceeds $50,000",
            category: "WEALTH",
            criteria: { balance: 50000 },
        },
    ];

    for (const ach of achievements) {
        await prisma.achievement.upsert({
            where: { code: ach.code },
            update: {},
            create: ach,
        });
    }

    console.log("✅ Seed data created");
}

main()
    .catch((e) => {
        console.error(e);
        process.exit(1);
    })
    .finally(() => prisma.$disconnect());
```

Run with:

```bash
npx prisma db seed
```
