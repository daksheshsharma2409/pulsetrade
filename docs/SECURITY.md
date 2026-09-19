# 🔐 Security & Authentication

> **Authentication, authorization, middleware, and security hardening.**

---

## 📌 Table of Contents

- [JWT Authentication](#-jwt-authentication)
- [Role-Based Access Control (RBAC)](#-role-based-access-control-rbac)
- [Attribute-Based Access Control (ABAC)](#-attribute-based-access-control-abac)
- [Security Checklist](#-security-checklist)
- [Middleware Chain](#-middleware-chain)
- [Rate Limiting](#-rate-limiting)
- [Input Validation](#-input-validation)

---

## 🔑 JWT Authentication

### Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          JWT AUTHENTICATION FLOW                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. LOGIN → POST /api/v1/auth/login { email, password }                    │
│  2. VALIDATE → bcrypt.compare(password, user.passwordHash)                 │
│  3. GENERATE TOKENS:                                                        │
│     - Access Token (JWT): expires in 15 minutes                             │
│     - Refresh Token (opaque): expires in 7 days, stored in DB               │
│  4. RETURN → { accessToken, refreshToken, expiresIn }                      │
│  5. SUBSEQUENT REQUESTS → Authorization: Bearer <accessToken>              │
│  6. TOKEN REFRESH → POST /api/v1/auth/refresh { refreshToken }             │
│     → Verify in DB, rotate, return new tokens                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation

```javascript
// services/authService.js
import jwt from "jsonwebtoken";
import bcrypt from "bcryptjs";
import { prisma } from "../lib/prisma.js";

const ACCESS_EXPIRY = "15m";
const REFRESH_EXPIRY_DAYS = 7;

export async function register(email, password, name) {
    const passwordHash = await bcrypt.hash(password, 10);
    const user = await prisma.user.create({
        data: { email, passwordHash, name },
    });
    return generateTokens(user);
}

export async function login(email, password) {
    const user = await prisma.user.findUnique({ where: { email } });
    if (!user) throw new Error("Invalid credentials");

    const valid = await bcrypt.compare(password, user.passwordHash);
    if (!valid) throw new Error("Invalid credentials");

    return generateTokens(user);
}

export async function generateTokens(user) {
    const accessToken = jwt.sign(
        { userId: user.id, role: user.role, email: user.email },
        process.env.JWT_ACCESS_SECRET,
        { expiresIn: ACCESS_EXPIRY },
    );

    const refreshToken = crypto.randomBytes(64).toString("hex");
    const expiresAt = new Date();
    expiresAt.setDate(expiresAt.getDate() + REFRESH_EXPIRY_DAYS);

    await prisma.refreshToken.create({
        data: {
            token: refreshToken,
            userId: user.id,
            expiresAt,
        },
    });

    return { accessToken, refreshToken, expiresIn: 900 };
}

export async function refresh(refreshToken) {
    const stored = await prisma.refreshToken.findUnique({
        where: { token: refreshToken },
        include: { user: true },
    });

    if (!stored || stored.revokedAt || stored.expiresAt < new Date()) {
        throw new Error("Invalid refresh token");
    }

    // Rotate: revoke old, issue new
    await prisma.refreshToken.update({
        where: { id: stored.id },
        data: { revokedAt: new Date() },
    });

    return generateTokens(stored.user);
}
```

### Auth Middleware

```javascript
// middleware/auth.js
import jwt from "jsonwebtoken";

export function authMiddleware(req, res, next) {
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith("Bearer ")) {
        return res.status(401).json({
            error: { code: "UNAUTHORIZED", message: "Authentication required" },
        });
    }

    const token = authHeader.split(" ")[1];

    try {
        const decoded = jwt.verify(token, process.env.JWT_ACCESS_SECRET);
        req.user = decoded;
        next();
    } catch (err) {
        return res.status(401).json({
            error: {
                code: "INVALID_TOKEN",
                message: "Invalid or expired token",
            },
        });
    }
}
```

---

## 👥 Role-Based Access Control (RBAC)

| Role          | Permissions                                                     |
| ------------- | --------------------------------------------------------------- |
| **TRADER**    | Create portfolios, place orders, view own data, social features |
| **MODERATOR** | All trader permissions + moderate content, view reports         |
| **ADMIN**     | Full access: manage users, view all data, system config         |

```javascript
// middleware/rbac.js
export const requireRole = (...roles) => {
    return (req, res, next) => {
        if (!req.user) {
            return res.status(401).json({
                error: {
                    code: "UNAUTHORIZED",
                    message: "Authentication required",
                },
            });
        }

        if (!roles.includes(req.user.role)) {
            return res.status(403).json({
                error: {
                    code: "FORBIDDEN",
                    message: "Insufficient permissions",
                },
            });
        }

        next();
    };
};

// Usage
router.get(
    "/admin/users",
    authMiddleware,
    requireRole("ADMIN"),
    adminController.listUsers,
);
```

---

## 🎯 Attribute-Based Access Control (ABAC)

For portfolio-level permissions:

```javascript
// middleware/abac.js
export const canAccessPortfolio = async (req, res, next) => {
    const portfolio = await prisma.portfolio.findUnique({
        where: { id: req.params.id },
    });

    if (!portfolio) {
        return res.status(404).json({ error: { code: "NOT_FOUND" } });
    }

    const canAccess =
        portfolio.userId === req.user.userId || req.user.role === "ADMIN";

    if (!canAccess) {
        return res.status(403).json({ error: { code: "FORBIDDEN" } });
    }

    req.portfolio = portfolio;
    next();
};
```

---

## ✅ Security Checklist

| Measure                | Implementation                 |
| ---------------------- | ------------------------------ |
| **Password Hashing**   | bcrypt (10 rounds)             |
| **JWT Security**       | Short-lived access + refresh   |
| **HTTPS**              | Enforced in production         |
| **Helmet**             | Security headers               |
| **CORS**               | Whitelist allowed origins      |
| **Rate Limiting**      | Per-IP + per-user              |
| **Input Validation**   | zod schemas                    |
| **SQL Injection**      | Prisma parameterized queries   |
| **XSS Protection**     | Input sanitization + CSP       |
| **CSRF Protection**    | SameSite cookies + CSRF tokens |
| **Session Management** | Refresh token rotation         |
| **Audit Logging**      | All sensitive actions logged   |

---

## 🔗 Middleware Chain

```
Request
   │
   ▼
1. CORS → 2. Helmet → 3. Rate Limiter → 4. Body Parser
   │
   ▼
5. Logger → 6. Auth → 7. RBAC → 8. Validation
   │
   ▼
9. Route Handler → Controller → Service → Model
   │
   ▼
10. Error Handler
```

```javascript
// middleware/index.js
export const setupMiddleware = (app) => {
    app.use(
        cors({
            origin: process.env.ALLOWED_ORIGINS?.split(",") || [
                "http://localhost:3000",
            ],
            credentials: true,
            methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
            allowedHeaders: ["Content-Type", "Authorization"],
        }),
    );

    app.use(helmet());

    app.use("/api/", globalLimiter);

    app.use(express.json({ limit: "10mb" }));
    app.use(express.urlencoded({ extended: true, limit: "10mb" }));

    app.use(requestLogger);
    app.use(compression());
    app.use(cookieParser());
};
```

---

## 🚦 Rate Limiting

```javascript
import rateLimit from "express-rate-limit";

const globalLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100,
    message: { error: { code: "RATE_LIMITED", message: "Too many requests" } },
});

const authLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 10,
});

const orderLimiter = rateLimit({
    windowMs: 60 * 1000, // 1 minute
    max: 30,
});

app.use("/api/", globalLimiter);
app.use("/api/v1/auth/login", authLimiter);
app.use("/api/v1/orders", orderLimiter);
```

---

## ✅ Input Validation

```javascript
// validators/orderValidator.js
import { z } from "zod";

export const createOrderSchema = z
    .object({
        portfolioId: z.string().uuid(),
        symbol: z.string().min(1).max(20),
        side: z.enum(["BUY", "SELL"]),
        type: z.enum(["MARKET", "LIMIT", "STOP_LOSS", "TAKE_PROFIT"]),
        quantity: z.number().positive().max(1000000),
        limitPrice: z.number().positive().optional(),
        stopPrice: z.number().positive().optional(),
    })
    .refine(
        (data) => {
            if (data.type === "LIMIT" && !data.limitPrice) return false;
            if (data.type === "STOP_LOSS" && !data.stopPrice) return false;
            return true;
        },
        { message: "Price required for this order type" },
    );

export const validate = (schema) => (req, res, next) => {
    try {
        schema.parse(req.body);
        next();
    } catch (error) {
        res.status(400).json({
            error: {
                code: "VALIDATION_ERROR",
                message: "Invalid input",
                details: error.errors,
            },
        });
    }
};
```
