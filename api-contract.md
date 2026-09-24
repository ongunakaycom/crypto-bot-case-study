# 🔌 API Contract — Crypto Bot

> Complete HTTP API specification for the Crypto Bot backend.
> This document is part of the [Crypto Bot Case Study](./README.md).

**Base URL (production):** `https://deep-seek-chat-bot-python.onrender.com`  
**Content-Type:** `application/json`  
**Authentication:** Bearer token (UUID v4) in `Authorization` header  
**Rate limiting:** Not currently enforced ⚠️

---

## 📌 Table of Contents

1. [Conventions](#-conventions)
2. [Authentication](#-authentication)
3. [Chat](#-chat)
4. [System](#-system)
5. [Error Handling](#-error-handling)
6. [Status Codes](#-status-codes)
7. [Examples](#-examples)

---

## 📐 Conventions

### Request Headers

| Header | Required | Value |
|--------|----------|-------|
| `Content-Type` | ✅ (POST/PUT) | `application/json` |
| `Authorization` | ✅ (protected) | `Bearer <token>` |
| `Origin` | Auto | Set by browser for CORS |

### Response Envelope

Every response — success or failure — follows the same shape:

```json
{
  "success": true,
  "data": { /* payload */ },
  "error": null
}
```

or on failure:

```json
{
  "success": false,
  "data": null,
  "error": "Human-readable error message"
}
```

> Some endpoints (e.g. `/auth/login`) return the payload inline
> (`{success, token, user}`) instead of nesting under `data`. This is a
> known inconsistency and a planned refactor. Documented honestly here
> rather than hidden.

### Timestamps

All timestamps are **ISO 8601 UTC** strings:

```
"2025-09-24T14:32:11.284Z"
```

### Identifiers

- `user_id` — MongoDB `ObjectId` (24-char hex)
- `session_id` (chat) — application-generated string: `user-<id>-<timestamp>`
- `token` (auth) — UUID v4

---

## 🔐 Authentication

### `POST /auth/signup`

Create a new user account.

**Body:**

```json
{
  "email": "user@example.com",
  "password": "minimum-6-chars",
  "name": "Jane Doe"
}
```

**Validation rules:**

| Field | Rule |
|-------|------|
| `email` | Must be valid format, unique in `users` |
| `password` | Minimum 6 characters |
| `name` | Non-empty string |

**Response — 201 Created:**

```json
{
  "success": true,
  "token": "8f3b2a1c-...-uuid-v4",
  "user": {
    "id": "6512a...",
    "email": "user@example.com",
    "name": "Jane Doe",
    "photoURL": null,
    "created_at": "2025-09-24T14:32:11.284Z"
  }
}
```

**Errors:**

| Status | Body | Cause |
|--------|------|-------|
| 400 | `{success:false, error:"password too short"}` | Password < 6 chars |
| 400 | `{success:false, error:"invalid email"}` | Malformed email |
| 409 | `{success:false, error:"email already registered"}` | Duplicate email |

---

### `POST /auth/login`

Authenticate an existing user.

**Body:**

```json
{
  "email": "user@example.com",
  "password": "minimum-6-chars"
}
```

**Response — 200 OK:**

```json
{
  "success": true,
  "token": "8f3b2a1c-...-uuid-v4",
  "user": {
    "id": "6512a...",
    "email": "user@example.com",
    "name": "Jane Doe",
    "photoURL": null,
    "created_at": "2025-09-24T14:32:11.284Z"
  }
}
```

**Errors:**

| Status | Body | Cause |
|--------|------|-------|
| 400 | `{success:false, error:"missing fields"}` | Empty email or password |
| 401 | `{success:false, error:"invalid credentials"}` | Wrong email or password |

> ⚠️ **Security note:** The error message is intentionally the same for
> "email not found" and "wrong password" to prevent user enumeration.

---

### `GET /auth/me`

Return the authenticated user's profile.

**Headers:**

```
Authorization: Bearer <token>
```

**Response — 200 OK:**

```json
{
  "success": true,
  "user": {
    "id": "6512a...",
    "email": "user@example.com",
    "name": "Jane Doe",
    "photoURL": "https://...",
    "created_at": "2025-09-24T14:32:11.284Z"
  }
}
```

**Errors:**

| Status | Body | Cause |
|--------|------|-------|
| 401 | `{success:false, error:"unauthorized"}` | Missing or invalid token |

---

### `POST /auth/logout`

Invalidate the current session token.

**Body:**

```json
{
  "token": "8f3b2a1c-...-uuid-v4"
}
```

**Response — 200 OK:**

```json
{ "success": true }
```

**Side effect:** Deletes the session document from MongoDB. Subsequent
requests with the same token return `401`.

---

### `PUT /auth/profile`

Update the authenticated user's profile.

**Headers:**

```
Authorization: Bearer <token>
```

**Body (all fields optional):**

```json
{
  "name": "Jane A. Doe",
  "photoURL": "data:image/png;base64,iVBOR..."
}
```

**Response — 200 OK:**

```json
{
  "success": true,
  "user": {
    "id": "6512a...",
    "email": "user@example.com",
    "name": "Jane A. Doe",
    "photoURL": "data:image/png;base64,iVBOR...",
    "created_at": "2025-09-24T14:32:11.284Z"
  }
}
```

**Errors:**

| Status | Body | Cause |
|--------|------|-------|
| 400 | `{success:false, error:"no fields to update"}` | Empty body |
| 400 | `{success:false, error:"name cannot be empty"}` | Whitespace name |
| 401 | `{success:false, error:"unauthorized"}` | Missing or invalid token |

> ⚠️ **Known limitation:** `photoURL` is stored as base64 in MongoDB.
> 16 MB document limit applies. Planned migration to S3 / GridFS.

---

## 💬 Chat

All chat endpoints operate on a **`session_id`** — an application-level
identifier that groups messages into a conversation thread. The same
`session_id` may span multiple HTTP requests; it is **not** the same as
the auth session token.

### `POST /chat`

Send a message to the AI and receive a contextual reply.

**Body:**

```json
{
  "message": "How is BTC looking today?",
  "session_id": "user-6512a...-1727188331"
}
```

**Server-side flow:**

1. Read chat history from `chat_sessions` (by `session_id`)
2. Fetch BTC metrics (from cache; refresh if stale — 5 min TTL)
3. Build prompt: system + metrics + history + new message
4. Call Gemini 3.5 Flash Lite
5. Persist both the user message and the AI reply
6. Return the reply

**Response — 200 OK:**

```json
{
  "success": true,
  "reply": "Bitcoin is currently trading around $63,200 with an RSI of 54, suggesting neutral momentum. The nearest support sits near $61,800...",
  "timestamp": "2025-09-24T14:32:11.284Z"
}
```

**Errors:**

| Status | Body | Cause |
|--------|------|-------|
| 400 | `{success:false, error:"message is required"}` | Empty message |
| 400 | `{success:false, error:"session_id is required"}` | Missing session |
| 502 | `{success:false, error:"AI service unavailable"}` | Gemini timeout / error |
| 503 | `{success:false, error:"metrics unavailable"}` | CoinGecko + cache both down |

**Graceful degradation:** If Gemini is unavailable, the API returns a
fallback message containing raw metrics instead of a 5xx error.

---

### `GET /chat/history`

Retrieve the full message history for a chat session.

**Query params:**

| Param | Required | Description |
|-------|----------|-------------|
| `session_id` | ✅ | Chat session identifier |

**Response — 200 OK:**

```json
{
  "success": true,
  "messages": [
    {
      "role": "user",
      "content": "How is BTC looking today?",
      "timestamp": "2025-09-24T14:32:08.112Z"
    },
    {
      "role": "assistant",
      "content": "Bitcoin is currently trading around $63,200...",
      "timestamp": "2025-09-24T14:32:11.284Z"
    }
  ]
}
```

**Errors:**

| Status | Body | Cause |
|--------|------|-------|
| 400 | `{success:false, error:"session_id is required"}` | Missing query param |

> ℹ️ If the `session_id` does not exist, the endpoint returns
> `{success: true, messages: []}` — an empty thread, not a 404.

---

### `POST /reset`

Clear the chat history for a session.

**Body:**

```json
{
  "session_id": "user-6512a...-1727188331"
}
```

**Response — 200 OK:**

```json
{ "success": true }
```

**Side effect:** Deletes the `chat_sessions` document with the matching
`session_id`. If the document does not exist, returns `200` anyway
(idempotent).

---

## ⚙️ System

### `GET /`

Health check endpoint. Used by UptimeRobot for uptime monitoring.

**Response — 200 OK:**

```json
{
  "status": "ok",
  "uptime": 384729,
  "timestamp": "2025-09-24T14:32:11.284Z"
}
```

- `uptime` — process uptime in seconds
- No authentication required

---

### `GET /metrics`

Return current BTC metrics without involving the AI layer.

**Response — 200 OK:**

```json
{
  "success": true,
  "metrics": {
    "symbol": "BTC",
    "price": 63214.52,
    "rsi_14": 54.3,
    "trend": "sideways",
    "support": 61800.0,
    "resistance": 64850.0,
    "volatility_30d": 0.042,
    "updated_at": "2025-09-24T14:30:00.000Z",
    "cache_age_seconds": 131
  }
}
```

**Errors:**

| Status | Body | Cause |
|--------|------|-------|
| 503 | `{success:false, error:"metrics unavailable"}` | CoinGecko unreachable |

> **Cache behavior:** `cache_age_seconds` lets clients know how fresh the
> data is. Stale-while-revalidate pattern: if `cache_age_seconds` > 300,
> the response is returned immediately and a background refresh is
> triggered.

---

## ❌ Error Handling

### Response Shape

All errors follow:

```json
{
  "success": false,
  "error": "Descriptive but non-sensitive message"
}
```

### Principles

| Principle | Implementation |
|-----------|---------------|
| **No stack traces in responses** | `try/except` wraps all handlers |
| **No user enumeration** | Login error is identical for unknown email and wrong password |
| **No sensitive fields in responses** | `password_hash` is never serialized (via `_public_user` helper) |
| **Explicit status codes** | Handlers return 4xx for client errors, 5xx only for server faults |
| **Idempotent where reasonable** | `/reset` returns 200 even if nothing to delete |

---

## 📊 Status Codes

| Code | Meaning | When used |
|------|---------|-----------|
| **200** | OK | Successful GET/POST/PUT |
| **201** | Created | Successful signup |
| **400** | Bad Request | Missing/invalid fields |
| **401** | Unauthorized | Missing/invalid/expired token |
| **403** | Forbidden | CORS rejection (frontend only) |
| **409** | Conflict | Duplicate email on signup |
| **429** | Too Many Requests | *Not currently returned* ⚠️ |
| **500** | Internal Server Error | Unhandled exception |
| **502** | Bad Gateway | Upstream (Gemini) failure |
| **503** | Service Unavailable | MongoDB or CoinGecko down |

---

## 🧪 Examples

### Full signup → chat → logout flow

**1. Sign up:**

```bash
curl -X POST https://deep-seek-chat-bot-python.onrender.com/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"demo@example.com","password":"demo1234","name":"Demo"}'
```

**2. Capture token from response, then send a chat message:**

```bash
curl -X POST https://deep-seek-chat-bot-python.onrender.com/chat \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What is BTC RSI right now?",
    "session_id": "user-demo-1727188331"
  }'
```

**3. Retrieve history:**

```bash
curl "https://deep-seek-chat-bot-python.onrender.com/chat/history?session_id=user-demo-1727188331"
```

**4. Logout:**

```bash
curl -X POST https://deep-seek-chat-bot-python.onrender.com/auth/logout \
  -H "Content-Type: application/json" \
  -d '{"token":"<token-from-step-1>"}'
```

---

## 🚧 Planned Changes

| Change | Reason | Priority |
|--------|--------|----------|
| Consistent response envelope | Some endpoints inline payload, others nest under `data` | P1 |
| Rate limiting headers | `X-RateLimit-Limit`, `X-RateLimit-Remaining` | P0 |
| Token TTL + refresh tokens | Currently sessions never expire | P0 |
| CORS whitelist | Currently wildcard | P0 |
| Pagination on `/chat/history` | Currently returns full thread | P2 |
| Streaming `/chat` responses | Currently blocking (2–5s) | P3 |

---

## 📚 Related Documents

- [README](./README.md) — project overview
- [Architecture](./architecture.md) — system design
- [Engineering Notes](./engineering-notes.md) — trade-offs and roadmap

---

**Last updated:** September 2025  
**Author:** [Ongun Akay](https://ongunakay.com)
```