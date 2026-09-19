# ⚡ Real-Time Features

> **Socket.IO implementation for PulseTrade.**

---

## 📌 Table of Contents

- [Overview](#-overview)
- [Namespaces & Rooms](#-namespaces--rooms)
- [Event Reference](#-event-reference)
- [Market Data Service](#-market-data-service)
- [Client Integration](#-client-integration)
- [Reconnection Handling](#-reconnection-handling)
- [Scaling with Redis Adapter](#-scaling-with-redis-adapter)

---

## 📖 Overview

PulseTrade uses **Socket.IO** for real-time communication between the server and clients. This powers:

- Live price streaming (stocks + crypto)
- Real-time order execution notifications
- Live portfolio updates
- Order book depth (Binance)
- Trade tape (Binance)
- Price alert notifications
- Social feed updates

---

## 🏠 Namespaces & Rooms

### Namespaces

| Namespace        | Purpose                                      |
| ---------------- | -------------------------------------------- |
| `/market`        | Price updates, order book, trade tape        |
| `/trading`       | Order execution, portfolio updates           |
| `/notifications` | Personal notifications, alerts, achievements |

### Rooms

| Room               | Pattern                   | Purpose                          |
| ------------------ | ------------------------- | -------------------------------- |
| **Stock room**     | `stock:{symbol}`          | Users watching a specific symbol |
| **User room**      | `user:{userId}`           | Personal notifications           |
| **Portfolio room** | `portfolio:{portfolioId}` | Portfolio-specific updates       |

### Example Room Joins

```javascript
// User watches AAPL
socket.join("stock:AAPL");

// User watches BTCUSDT
socket.join("stock:BINANCE:BTCUSDT");

// Personal room for notifications
socket.join(`user:${userId}`);

// Portfolio updates
socket.join(`portfolio:${portfolioId}`);
```

---

## 📡 Event Reference

### `/market` Namespace

| Event              | Direction       | Payload                                        |
| ------------------ | --------------- | ---------------------------------------------- |
| `subscribe`        | Client → Server | `{ symbol }`                                   |
| `unsubscribe`      | Client → Server | `{ symbol }`                                   |
| `price-update`     | Server → Client | `{ symbol, price, volume, timestamp }`         |
| `orderbook-update` | Server → Client | `{ symbol, bids, asks }`                       |
| `trade-tape`       | Server → Client | `{ symbol, price, quantity, side, timestamp }` |

### `/trading` Namespace

| Event              | Direction       | Payload                                |
| ------------------ | --------------- | -------------------------------------- |
| `order-executed`   | Server → Client | `{ orderId, symbol, price, quantity }` |
| `portfolio-update` | Server → Client | `{ portfolioId, totalValue, pnl }`     |

### `/notifications` Namespace

| Event                  | Direction       | Payload                                 |
| ---------------------- | --------------- | --------------------------------------- |
| `alert-triggered`      | Server → Client | `{ alertId, symbol, price, condition }` |
| `achievement-unlocked` | Server → Client | `{ achievementId, name, icon }`         |
| `notification`         | Server → Client | `{ id, type, title, message }`          |

---

## 📡 Market Data Service

```javascript
// services/marketDataService.js
import WebSocket from "ws";
import Redis from "ioredis";

class MarketDataService {
    constructor(io) {
        this.io = io;
        this.redis = new Redis(process.env.REDIS_URL);
        this.finnhubSocket = null;
        this.binanceSocket = null;
        this.subscriptions = new Map();
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 10;
    }

    // ═══════════════════════════════════════════════════════════
    // FINNHUB (US Stocks)
    // ═══════════════════════════════════════════════════════════
    connectFinnhub() {
        const token = process.env.FINNHUB_API_KEY;
        this.finnhubSocket = new WebSocket(
            `wss://ws.finnhub.io?token=${token}`,
        );

        this.finnhubSocket.on("open", () => {
            console.log("✅ Finnhub connected");
            this.reconnectAttempts = 0;

            for (const symbol of this.subscriptions.keys()) {
                if (!symbol.startsWith("BINANCE:")) {
                    this.finnhubSocket.send(
                        JSON.stringify({
                            type: "subscribe",
                            symbol: symbol,
                        }),
                    );
                }
            }
        });

        this.finnhubSocket.on("message", (data) => {
            const parsed = JSON.parse(data);
            if (parsed.type === "trade") {
                parsed.data.forEach((trade) => {
                    this.handlePriceUpdate({
                        symbol: trade.s,
                        price: trade.p,
                        volume: trade.v,
                        timestamp: trade.t,
                    });
                });
            }
        });

        this.finnhubSocket.on("close", () => {
            console.log("⚠️ Finnhub disconnected");
            this.scheduleReconnect(() => this.connectFinnhub());
        });

        this.finnhubSocket.on("error", (err) => {
            console.error("Finnhub error:", err.message);
        });
    }

    // ═══════════════════════════════════════════════════════════
    // BINANCE (Crypto)
    // ═══════════════════════════════════════════════════════════
    connectBinance() {
        const streams = [
            "btcusdt@trade",
            "ethusdt@trade",
            "solusdt@trade",
            "bnbusdt@trade",
            "xrpusdt@trade",
            "adausdt@trade",
            "dogeusdt@trade",
            "maticusdt@trade",
        ];
        const url = `wss://stream.binance.com:9443/stream?streams=${streams.join("/")}`;

        this.binanceSocket = new WebSocket(url);

        this.binanceSocket.on("open", () => {
            console.log("✅ Binance connected");
            this.reconnectAttempts = 0;
        });

        this.binanceSocket.on("message", (data) => {
            const parsed = JSON.parse(data);
            const trade = parsed.data;

            if (trade && trade.e === "trade") {
                this.handlePriceUpdate({
                    symbol: `BINANCE:${trade.s}`,
                    price: parseFloat(trade.p),
                    volume: parseFloat(trade.q),
                    timestamp: trade.T,
                });
            }
        });

        this.binanceSocket.on("close", () => {
            console.log("⚠️ Binance disconnected");
            this.scheduleReconnect(() => this.connectBinance());
        });

        this.binanceSocket.on("error", (err) => {
            console.error("Binance error:", err.message);
        });
    }

    // ═══════════════════════════════════════════════════════════
    // PRICE HANDLER
    // ═══════════════════════════════════════════════════════════
    async handlePriceUpdate(data) {
        const priceData = {
            symbol: data.symbol,
            price: data.price,
            volume: data.volume,
            timestamp: data.timestamp || Date.now(),
        };

        // 1. Cache in Redis (TTL: 5 seconds)
        await this.redis.setex(
            `price:${data.symbol}`,
            5,
            JSON.stringify(priceData),
        );

        // 2. Publish to Redis Pub/Sub
        await this.redis.publish("price-updates", JSON.stringify(priceData));

        // 3. Emit to Socket.IO room
        this.io
            .of("/market")
            .to(`stock:${data.symbol}`)
            .emit("price-update", priceData);
    }

    // ═══════════════════════════════════════════════════════════
    // SUBSCRIPTION MANAGEMENT
    // ═══════════════════════════════════════════════════════════
    async subscribe(symbol) {
        const count = this.subscriptions.get(symbol) || 0;
        this.subscriptions.set(symbol, count + 1);

        if (count === 0) {
            if (
                !symbol.startsWith("BINANCE:") &&
                this.finnhubSocket?.readyState === WebSocket.OPEN
            ) {
                this.finnhubSocket.send(
                    JSON.stringify({
                        type: "subscribe",
                        symbol: symbol,
                    }),
                );
            }
        }

        const cached = await this.redis.get(`price:${symbol}`);
        return cached ? JSON.parse(cached) : null;
    }

    async unsubscribe(symbol) {
        const count = this.subscriptions.get(symbol) || 0;

        if (count <= 1) {
            this.subscriptions.delete(symbol);
            if (
                !symbol.startsWith("BINANCE:") &&
                this.finnhubSocket?.readyState === WebSocket.OPEN
            ) {
                this.finnhubSocket.send(
                    JSON.stringify({
                        type: "unsubscribe",
                        symbol: symbol,
                    }),
                );
            }
        } else {
            this.subscriptions.set(symbol, count - 1);
        }
    }

    // ═══════════════════════════════════════════════════════════
    // RECONNECTION
    // ═══════════════════════════════════════════════════════════
    scheduleReconnect(connectFn) {
        if (this.reconnectAttempts >= this.maxReconnectAttempts) {
            console.error("Max reconnect attempts reached");
            return;
        }

        const delay = Math.min(
            1000 * Math.pow(2, this.reconnectAttempts),
            30000,
        );
        this.reconnectAttempts++;

        console.log(
            `Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`,
        );
        setTimeout(connectFn, delay);
    }
}

export default MarketDataService;
```

---

## 🖥️ Socket.IO Server Setup

```javascript
// sockets/index.js
import { Server } from "socket.io";
import MarketDataService from "../services/marketDataService.js";

export function setupSockets(server) {
    const io = new Server(server, {
        cors: {
            origin: process.env.ALLOWED_ORIGINS?.split(",") || [
                "http://localhost:3000",
            ],
            credentials: true,
        },
        transports: ["websocket", "polling"],
    });

    const marketDataService = new MarketDataService(io);
    marketDataService.connectFinnhub();
    marketDataService.connectBinance();

    // ═══════════════════════════════════════════════════════════
    // /market Namespace
    // ═══════════════════════════════════════════════════════════
    const marketNamespace = io.of("/market");

    marketNamespace.on("connection", (socket) => {
        console.log(`Market client connected: ${socket.id}`);

        socket.on("subscribe", async (symbol) => {
            socket.join(`stock:${symbol}`);
            const latestPrice = await marketDataService.subscribe(symbol);
            if (latestPrice) socket.emit("price-update", latestPrice);
        });

        socket.on("unsubscribe", (symbol) => {
            socket.leave(`stock:${symbol}`);
            marketDataService.unsubscribe(symbol);
        });

        socket.on("disconnect", () => {
            console.log(`Market client disconnected: ${socket.id}`);
        });
    });

    // ═══════════════════════════════════════════════════════════
    // /trading Namespace
    // ═══════════════════════════════════════════════════════════
    const tradingNamespace = io.of("/trading");

    tradingNamespace.on("connection", (socket) => {
        console.log(`Trading client connected: ${socket.id}`);

        socket.on("join:portfolio", (portfolioId) => {
            socket.join(`portfolio:${portfolioId}`);
        });

        socket.on("leave:portfolio", (portfolioId) => {
            socket.leave(`portfolio:${portfolioId}`);
        });

        socket.on("disconnect", () => {
            console.log(`Trading client disconnected: ${socket.id}`);
        });
    });

    // ═══════════════════════════════════════════════════════════
    // /notifications Namespace
    // ═══════════════════════════════════════════════════════════
    const notificationNamespace = io.of("/notifications");

    notificationNamespace.on("connection", (socket) => {
        console.log(`Notification client connected: ${socket.id}`);

        socket.on("join:user", (userId) => {
            socket.join(`user:${userId}`);
        });

        socket.on("disconnect", () => {
            console.log(`Notification client disconnected: ${socket.id}`);
        });
    });

    return { io, marketDataService };
}
```

---

## 🖥️ Client Integration

### React Hook for Market Data

```typescript
// hooks/useMarketData.ts
import { useEffect, useState } from "react";
import { io, Socket } from "socket.io-client";

interface PriceData {
    symbol: string;
    price: number;
    volume: number;
    timestamp: number;
}

export function useMarketData(symbols: string[]) {
    const [prices, setPrices] = useState<Record<string, PriceData>>({});
    const [socket, setSocket] = useState<Socket | null>(null);
    const [isConnected, setIsConnected] = useState(false);

    useEffect(() => {
        const newSocket = io(`${process.env.NEXT_PUBLIC_API_URL}/market`, {
            transports: ["websocket"],
            reconnection: true,
            reconnectionDelay: 1000,
            reconnectionAttempts: 10,
        });

        newSocket.on("connect", () => {
            setIsConnected(true);
            symbols.forEach((symbol) => newSocket.emit("subscribe", symbol));
        });

        newSocket.on("price-update", (data: PriceData) => {
            setPrices((prev) => ({ ...prev, [data.symbol]: data }));
        });

        newSocket.on("disconnect", () => setIsConnected(false));

        newSocket.on("reconnect", () => {
            symbols.forEach((symbol) => newSocket.emit("subscribe", symbol));
        });

        setSocket(newSocket);

        return () => {
            symbols.forEach((symbol) => newSocket.emit("unsubscribe", symbol));
            newSocket.disconnect();
        };
    }, [symbols.join(",")]);

    return { prices, isConnected, socket };
}
```

---

## 🔄 Reconnection Handling

**Server-side:**

- Exponential backoff on upstream WebSocket reconnection
- Max 10 reconnection attempts
- Auto re-subscribe to all active symbols on reconnect

**Client-side:**

- Socket.IO handles reconnection automatically
- On `reconnect` event, re-subscribe to all symbols
- Latest cached prices emitted immediately after rejoin

---

## 📈 Scaling with Redis Adapter

For horizontal scaling across multiple backend instances:

```javascript
import { createAdapter } from "@socket.io/redis-adapter";
import { createClient } from "redis";

const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

io.adapter(createAdapter(pubClient, subClient));
```

This allows Socket.IO to broadcast events across all backend instances via Redis Pub/Sub.
