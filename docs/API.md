# 🔌 API Documentation

> **RESTful API design for PulseTrade.**

---

## 📌 Table of Contents

- [API Conventions](#-api-conventions)
- [Authentication Endpoints](#-authentication-endpoints)
- [User Endpoints](#-user-endpoints)
- [Portfolio Endpoints](#-portfolio-endpoints)
- [Order Endpoints](#-order-endpoints)
- [Trade Endpoints](#-trade-endpoints)
- [Market Data Endpoints](#-market-data-endpoints)
- [Alert Endpoints](#-alert-endpoints)
- [Social Endpoints](#-social-endpoints)
- [Copy Trading Endpoints](#-copy-trading-endpoints)
- [Leaderboard & Leagues](#-leaderboard--leagues)
- [Achievement Endpoints](#-achievement-endpoints)
- [Sentiment Endpoints](#-sentiment-endpoints)
- [Notification Endpoints](#-notification-endpoints)
- [Admin Endpoints](#-admin-endpoints)
- [Sample Requests](#-sample-requests)

---

## 📐 API Conventions

| Aspect              | Rule                                    |
| ------------------- | --------------------------------------- |
| **Base URL**        | `/api/v1`                               |
| **Versioning**      | URL path (`/v1/`, `/v2/`)               |
| **Resource Naming** | Plural nouns (`/users`, `/orders`)      |
| **HTTP Methods**    | GET, POST, PUT, PATCH, DELETE           |
| **Status Codes**    | 200, 201, 400, 401, 403, 404, 429, 500  |
| **Authentication**  | Bearer token in `Authorization` header  |
| **Pagination**      | `?page=1&limit=20`                      |
| **Sorting**         | `?sort=createdAt&order=desc`            |
| **Filtering**       | `?status=PENDING&symbol=AAPL`           |
| **Error Format**    | `{ error: { code, message, details } }` |

---

## 🔐 Authentication Endpoints

| Method | Endpoint                | Description          | Auth          |
| ------ | ----------------------- | -------------------- | ------------- |
| POST   | `/api/v1/auth/register` | Register new user    | No            |
| POST   | `/api/v1/auth/login`    | Login, get tokens    | No            |
| POST   | `/api/v1/auth/refresh`  | Refresh access token | Refresh token |
| POST   | `/api/v1/auth/logout`   | Logout, revoke token | Yes           |
| GET    | `/api/v1/auth/me`       | Get current user     | Yes           |

---

## 👤 User Endpoints

| Method | Endpoint                      | Description          | Auth |
| ------ | ----------------------------- | -------------------- | ---- |
| GET    | `/api/v1/users/:id`           | Get user profile     | Yes  |
| PATCH  | `/api/v1/users/me`            | Update profile       | Yes  |
| GET    | `/api/v1/users/:id/portfolio` | Get public portfolio | Yes  |
| GET    | `/api/v1/users/:id/trades`    | Get user's trades    | Yes  |

---

## 💼 Portfolio Endpoints

| Method | Endpoint                             | Description            | Auth |
| ------ | ------------------------------------ | ---------------------- | ---- |
| GET    | `/api/v1/portfolios`                 | List user's portfolios | Yes  |
| POST   | `/api/v1/portfolios`                 | Create new portfolio   | Yes  |
| GET    | `/api/v1/portfolios/:id`             | Get portfolio details  | Yes  |
| PATCH  | `/api/v1/portfolios/:id`             | Update portfolio       | Yes  |
| DELETE | `/api/v1/portfolios/:id`             | Delete portfolio       | Yes  |
| GET    | `/api/v1/portfolios/:id/holdings`    | Get holdings           | Yes  |
| GET    | `/api/v1/portfolios/:id/performance` | Get performance data   | Yes  |
| GET    | `/api/v1/portfolios/:id/pnl-heatmap` | Get daily P&L heatmap  | Yes  |

---

## 📋 Order Endpoints

| Method | Endpoint             | Description          | Auth |
| ------ | -------------------- | -------------------- | ---- |
| GET    | `/api/v1/orders`     | List orders          | Yes  |
| POST   | `/api/v1/orders`     | Place new order      | Yes  |
| GET    | `/api/v1/orders/:id` | Get order details    | Yes  |
| PATCH  | `/api/v1/orders/:id` | Modify pending order | Yes  |
| DELETE | `/api/v1/orders/:id` | Cancel order         | Yes  |

---

## 💱 Trade Endpoints

| Method | Endpoint             | Description       | Auth |
| ------ | -------------------- | ----------------- | ---- |
| GET    | `/api/v1/trades`     | List trades       | Yes  |
| GET    | `/api/v1/trades/:id` | Get trade details | Yes  |

---

## 📊 Market Data Endpoints

| Method | Endpoint                           | Description                 | Auth |
| ------ | ---------------------------------- | --------------------------- | ---- |
| GET    | `/api/v1/market/search`            | Search symbols              | Yes  |
| GET    | `/api/v1/market/quote/:symbol`     | Get current quote           | Yes  |
| GET    | `/api/v1/market/history/:symbol`   | Get historical candles      | Yes  |
| GET    | `/api/v1/market/news/:symbol`      | Get news (Finnhub)          | Yes  |
| GET    | `/api/v1/market/orderbook/:symbol` | Get order book (Binance)    | Yes  |
| GET    | `/api/v1/market/trades/:symbol`    | Get recent trades (Binance) | Yes  |

---

## 🔔 Alert Endpoints

| Method | Endpoint             | Description        | Auth |
| ------ | -------------------- | ------------------ | ---- |
| GET    | `/api/v1/alerts`     | List user's alerts | Yes  |
| POST   | `/api/v1/alerts`     | Create alert       | Yes  |
| PATCH  | `/api/v1/alerts/:id` | Update alert       | Yes  |
| DELETE | `/api/v1/alerts/:id` | Delete alert       | Yes  |

---

## 👥 Social Endpoints

| Method | Endpoint                        | Description     | Auth |
| ------ | ------------------------------- | --------------- | ---- |
| GET    | `/api/v1/social/feed`           | Get social feed | Yes  |
| POST   | `/api/v1/social/follow/:userId` | Follow a user   | Yes  |
| DELETE | `/api/v1/social/follow/:userId` | Unfollow a user | Yes  |
| GET    | `/api/v1/social/followers`      | Get followers   | Yes  |
| GET    | `/api/v1/social/following`      | Get following   | Yes  |

---

## 🔄 Copy Trading Endpoints

| Method | Endpoint                   | Description             | Auth |
| ------ | -------------------------- | ----------------------- | ---- |
| GET    | `/api/v1/copy-trading`     | List copy relationships | Yes  |
| POST   | `/api/v1/copy-trading`     | Start copying a trader  | Yes  |
| PATCH  | `/api/v1/copy-trading/:id` | Update copy settings    | Yes  |
| DELETE | `/api/v1/copy-trading/:id` | Stop copying            | Yes  |

---

## 🏆 Leaderboard & Leagues

| Method | Endpoint                          | Description         | Auth |
| ------ | --------------------------------- | ------------------- | ---- |
| GET    | `/api/v1/leaderboard`             | Global leaderboard  | Yes  |
| GET    | `/api/v1/leaderboard/friends`     | Friends leaderboard | Yes  |
| GET    | `/api/v1/leagues`                 | List active leagues | Yes  |
| POST   | `/api/v1/leagues/:id/join`        | Join a league       | Yes  |
| GET    | `/api/v1/leagues/:id/leaderboard` | League leaderboard  | Yes  |

---

## 🏅 Achievement Endpoints

| Method | Endpoint                    | Description           | Auth |
| ------ | --------------------------- | --------------------- | ---- |
| GET    | `/api/v1/achievements`      | List all achievements | Yes  |
| GET    | `/api/v1/achievements/mine` | User's achievements   | Yes  |

---

## 📊 Sentiment Endpoints

| Method | Endpoint                         | Description              | Auth |
| ------ | -------------------------------- | ------------------------ | ---- |
| GET    | `/api/v1/sentiment/:symbol`      | Get sentiment for symbol | Yes  |
| POST   | `/api/v1/sentiment/:symbol/vote` | Vote on sentiment        | Yes  |

---

## 🔔 Notification Endpoints

| Method | Endpoint                         | Description        | Auth |
| ------ | -------------------------------- | ------------------ | ---- |
| GET    | `/api/v1/notifications`          | List notifications | Yes  |
| PATCH  | `/api/v1/notifications/:id/read` | Mark as read       | Yes  |
| PATCH  | `/api/v1/notifications/read-all` | Mark all as read   | Yes  |

---

## 🛡️ Admin Endpoints

| Method | Endpoint                          | Description        | Auth  |
| ------ | --------------------------------- | ------------------ | ----- |
| GET    | `/api/v1/admin/users`             | List all users     | Admin |
| PATCH  | `/api/v1/admin/users/:id/suspend` | Suspend user       | Admin |
| GET    | `/api/v1/admin/analytics`         | Platform analytics | Admin |
| GET    | `/api/v1/admin/health`            | System health      | Admin |
| POST   | `/api/v1/admin/reports`           | Generate report    | Admin |

---

## 📝 Sample Requests

### Place Order

**Request:**

```http
POST /api/v1/orders
Authorization: Bearer eyJhbGci...
Content-Type: application/json

{
  "portfolioId": "uuid-here",
  "symbol": "AAPL",
  "side": "BUY",
  "type": "LIMIT",
  "quantity": 10,
  "limitPrice": 140.00
}
```

**Response:**

```json
{
    "success": true,
    "data": {
        "id": "order-uuid",
        "portfolioId": "uuid-here",
        "symbol": "AAPL",
        "assetType": "STOCK",
        "side": "BUY",
        "type": "LIMIT",
        "status": "PENDING",
        "quantity": 10,
        "limitPrice": 140.0,
        "createdAt": "2026-09-19T14:32:15Z"
    },
    "meta": {
        "message": "Order placed successfully"
    }
}
```

### Error Response

```json
{
    "success": false,
    "error": {
        "code": "INSUFFICIENT_BALANCE",
        "message": "Insufficient cash balance",
        "details": {
            "required": 1400.0,
            "available": 1200.0
        }
    }
}
```

### Register User

**Request:**

```http
POST /api/v1/auth/register
Content-Type: application/json

{
  "email": "trader@example.com",
  "password": "SecurePass123!",
  "name": "John Trader"
}
```

**Response:**

```json
{
    "success": true,
    "data": {
        "user": {
            "id": "user-uuid",
            "email": "trader@example.com",
            "name": "John Trader",
            "role": "TRADER"
        },
        "accessToken": "eyJhbGci...",
        "refreshToken": "rt_abc123...",
        "expiresIn": 900
    }
}
```
