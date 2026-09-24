# 🧠 Engineering Notes — Crypto Bot

> Trade-offs, decisions and roadmap behind Crypto Bot.
> This document is part of the [Crypto Bot Case Study](./README.md).

These are the notes I would want a senior engineer to read before reviewing
the system. They explain **why** things are the way they are, **what I would
do differently**, and **where the sharp edges are**.

---

## 📌 Table of Contents

1. [Design Philosophy](#-design-philosophy)
2. [Key Decisions & Trade-offs](#-key-decisions--trade-offs)
3. [What Went Well](#-what-went-well)
4. [What I Would Do Differently](#-what-i-would-do-differently)
5. [Known Limitations](#-known-limitations)
6. [Roadmap](#-roadmap)
7. [Observability & Operations](#-observability--operations)
8. [Lessons Learned](#-lessons-learned)

---

## 🎯 Design Philosophy

Three principles shaped every decision in this project:

### 1. **Compute deterministically, explain probabilistically**

LLMs hallucinate numbers. Technical indicators don't. Crypto Bot computes
RSI, trend, support and resistance **in Python** using deterministic
formulas, then asks Gemini only to **explain those numbers in natural
language**. Gemini is never asked to compute, estimate or recall values.

This separation is the single most important design decision in the
project. It means:

- Metrics are reproducible and auditable
- Model swaps don't change the numbers, only the tone of the explanation
- The AI layer can fail gracefully without losing data integrity

### 2. **Separate concerns even when it feels redundant**

The codebase is split into `databases/` (API client), `store/` (state),
`components/` (UI atoms) and `pages/` (route-level views). On a small
project this feels like over-engineering. It isn't.

The moment you need to swap a model, change an auth strategy or reuse the
chat logic in another view, the boundaries pay for themselves. I've seen
codebases where `brain.js` didn't exist and every component imported
`axios` directly. That code is unmaintainable six months later.

### 3. **Ship working software, then harden**

This is a production-deployed MVP, not a thesis project. Authentication
works. Chat works. Data persists. Monitoring is live. Some hardening —
rate limiting, token TTL, CORS whitelist — is deliberately deferred to
later phases, documented openly, and prioritized. Shipping beats
perfection.

---

## ⚖️ Key Decisions & Trade-offs

### Session tokens vs JWT

| | Session tokens (chosen) | JWT |
|---|---|---|
| Revocation | Instant (delete from DB) | Requires denylist |
| Statelessness | No (DB lookup per request) | Yes |
| Complexity | Low | Medium |
| Federated auth ready | No | Yes |

**Chosen because:** The app has a single backend, no cross-service auth
requirements, and logout needs to be immediate. DB lookup cost is
negligible at current scale.

**Would reconsider if:** Traffic grows beyond ~100 req/s, or if a second
backend service needs to validate tokens independently.

---

### Zustand vs Redux Toolkit

| | Zustand (chosen) | Redux Toolkit |
|---|---|---|
| Boilerplate | Minimal | Heavy |
| Learning curve | Flat | Moderate |
| DevTools | Good enough | Excellent |
| Fit for 2 slices | Perfect | Overkill |

**Chosen because:** The app has exactly two orthogonal state slices
(`accountStore`, `tradeStore`). Redux Toolkit would have added ~5x the
code for the same behavior.

**Would reconsider if:** State grows beyond ~5 slices, or if time-travel
debugging becomes a real need.

---

### Flask vs FastAPI

| | Flask (chosen) | FastAPI |
|---|---|---|
| Async native | No | Yes |
| Type hints | Optional | First-class |
| Ecosystem maturity | Very mature | Mature |
| Fit for this workload | Excellent | Also fine |

**Chosen because:** Only one endpoint (`/chat`) is I/O-heavy, and it's a
single outbound AI call. FastAPI's async advantage would show up with
concurrent external calls. Velocity won this trade-off.

**Would reconsider if:** We add streaming responses, WebSocket chat, or
concurrent multi-provider AI calls.

---

### MongoDB vs PostgreSQL

| | MongoDB (chosen) | PostgreSQL |
|---|---|---|
| Document model | Native | JSONB workaround |
| Transactions | Multi-doc (v4+) | Full ACID |
| Schema migration | Schemaless | Explicit |
| Fit for chat history | Excellent | Acceptable |

**Chosen because:** Chat history is naturally a nested document array.
Schema evolution during prototyping was frictionless. Atlas free tier
removed ops overhead entirely.

**Would reconsider if:** The app adds user-to-user features, financial
data, or any workflow requiring cross-collection transactions. This is
a known architectural ceiling.

---

### Gemini 3.5 Flash Lite vs larger models

**Chosen because:** Cost-per-request matters for a chat app. Flash Lite
delivers sufficient quality for "explain these RSI numbers" tasks at a
fraction of the cost of Gemini Pro or GPT-4 class models.

**Mitigation for model lock-in:** All AI calls flow through a single
`brain.js` client. Swapping to Claude, GPT or a local Llama is a
one-file change, not a rewrite.

---

### Stale-while-revalidate for CoinGecko

| Strategy | Latency | Freshness | CoinGecko load |
|---|---|---|---|
| No cache | High | Perfect | High (rate-limited) |
| TTL cache | High on miss | Up to TTL old | Low |
| **SWR (chosen)** | Low always | Up to TTL old | Low |

**Chosen because:** Users never wait. If metrics are fresh, they get
fresh metrics. If stale, they get stale metrics immediately and a
background refresh happens. The chat UX never depends on CoinGecko's
response time.

**Trade-off accepted:** A user might see 5-minute-old data. For BTC
analysis over a 90-day window, this is irrelevant.

---

## ✅ What Went Well

1. **Deterministic AI boundary** — Gemini never touches numbers. This
   eliminated an entire class of hallucination bugs before they could
   happen.

2. **Cache strategy** — The 5-minute stale-while-revalidate pattern
   turned a CoinGecko rate-limit problem into a non-issue. Zero user
   complaints about slow metrics.

3. **State separation** — Zustand stores for account and trade session
   made feature additions trivial. Adding account settings was a
   one-afternoon task.

4. **Single HTTP boundary (`brain.js`)** — When the auth flow needed to
   change from "token in body" to "Bearer header," it was one file.

5. **Deployment velocity** — Vercel + Render auto-deploy means
   `git push` is the entire release process. No CI pipeline needed at
   this scale.

6. **Graceful degradation** — If Gemini is down, the API returns raw
   metrics with a fallback message. The user still gets value.

---

## 🔧 What I Would Do Differently

If I started over tomorrow, I would change four things:

### 1. TypeScript from day one

JavaScript was faster to prototype, but the cost is now visible in
`brain.js` and the Zustand stores. Types would have caught three bugs
during development that I only found in manual testing.

### 2. Refresh tokens from day one

Adding token TTL after the fact means touching every auth-protected
endpoint. Doing it at signup time would have been free.

### 3. Consistent response envelope

Some endpoints nest under `data`, some inline the payload. This is a
documented inconsistency in `api-contract.md`. A single
`ok(payload)` / `fail(message)` helper would have prevented it.

### 4. Structured logging from the start

`print()` statements work in development but are useless in production.
A `structlog` or `logging` setup with JSON output would have made the
Render logs actually searchable from day one.

---

## ⚠️ Known Limitations

| # | Limitation | Severity | Impact | Mitigation Path |
|---|-----------|----------|--------|-----------------|
| 1 | No token TTL | 🔴 High | Sessions valid indefinitely | TTL index + refresh tokens |
| 2 | CORS wildcard | 🔴 High | Defense-in-depth reduced | Whitelist Vercel origin |
| 3 | No rate limiting | 🔴 High | Abuse possible | `flask-limiter` |
| 4 | No error tracking | 🟡 Medium | Silent failures possible | Sentry integration |
| 5 | Render cold start (30–60s) | 🟡 Medium | First request after idle is slow | Paid tier / Cloud Run |
| 6 | Base64 photo uploads | 🟡 Medium | 16 MB document limit | S3 / GridFS migration |
| 7 | Single Gunicorn worker | 🟡 Medium | No parallel handling | Increase workers (paid) |
| 8 | No test coverage | 🟡 Medium | Regression risk | pytest + Jest |
| 9 | Gemini rate limits | 🟢 Low | Throttling under load | Retry + exponential backoff |
| 10 | No streaming chat | 🟢 Low | 2–5s perceived latency | SSE / WebSocket |

These are documented openly because a senior engineer's job is to know
the ceiling of their own system.

---

## 🗺️ Roadmap

### P0 — Production Hardening *(next 2 weeks)*

- [ ] **Token TTL index + refresh tokens** — MongoDB TTL index on
      `sessions.created_at`, plus a `/auth/refresh` endpoint
- [ ] **CORS whitelist** — Replace `*` with Vercel production origin
- [ ] **Rate limiting** — `flask-limiter` with per-IP and per-token
      limits on `/chat` and `/auth/*`
- [ ] **Error tracking** — Sentry on both frontend and backend

### P1 — User-Visible Features *(next 1–2 months)*

- [ ] **Chat history hydration on F5** — Currently the thread is lost
      on refresh because `tradeStore` doesn't call `/chat/history`
      on mount
- [ ] **Multi-symbol support** — BTC / ETH / SOL with a symbol selector
- [ ] **Portfolio tracking** — Users log holdings, get contextual
      analysis on their positions
- [ ] **Price alerts** — Email or in-app notification when RSI crosses
      thresholds

### P2 — Technical Debt *(next 3–6 months)*

- [ ] **TypeScript migration** — Frontend first, then type hints on
      backend endpoints
- [ ] **Test coverage** — Jest for `brain.js` and Zustand stores,
      pytest for Flask routes
- [ ] **CI/CD pipeline** — GitHub Actions running tests before deploy
- [ ] **Dockerization** — For local dev parity and future K8s path
- [ ] **Photo uploads to S3** — Replace base64 in MongoDB

### P3 — Scaling *(when traffic justifies it)*

- [ ] **Redis cache** — Shared metrics + session cache across workers
- [ ] **PostgreSQL migration** — For transactional user data
- [ ] **WebSocket chat** — Streaming responses via SSE or WS
- [ ] **CDN for static assets** — CloudFront / Cloudflare in front of
      Vercel if needed

---

## 📊 Observability & Operations

### Current State

| Area | Tool | Coverage |
|------|------|----------|
| Uptime | UptimeRobot (HEAD `/` every 5 min) | ✅ Production |
| Logs | Render built-in log viewer | 🟡 Basic |
| Errors | None ❌ | 🔴 Silent failures possible |
| Metrics | None ❌ | 🔴 No latency/throughput visibility |
| Traces | None ❌ | 🔴 No request tracing |

### What I Log Today

- Auth events (signup, login, logout) — email redacted
- AI call latency (ms) — useful for detecting Gemini degradation
- CoinGecko fetch outcomes — success/failure/cache-hit
- Unhandled exceptions — stack trace to stderr only

### What I Want to Log

- Structured JSON with `request_id` correlation
- User-id tagged events (for per-user debugging)
- Endpoint latency histograms
- Cache hit ratios

This is a P1 priority once Sentry is integrated.

---

## 🎓 Lessons Learned

### 1. **AI is not a calculator**

The most common mistake in AI-augmented apps is asking the model to
compute. It will compute — incorrectly — and confidently. Move every
deterministic calculation out of the model and into code. Let the model
do what it's actually good at: explaining.

### 2. **Cache is not optional with external APIs**

CoinGecko's rate limits are real. Without caching, the app would break
under any meaningful load. The stale-while-revalidate pattern is
surprisingly easy to implement and delivers 90% of the benefit of a
proper cache with 10% of the complexity.

### 3. **Ship, then harden — but write it down**

Deferring hardening is fine. Deferring it silently is not. Every
limitation in this document was written down before it was shipped. This
makes them planned work, not forgotten bugs.

### 4. **Deployment friction kills momentum**

Vercel + Render auto-deploy means a bug fix goes from "idea" to
"production" in under two minutes. This changes how you work. You
experiment more, you iterate faster, you learn quicker.

### 5. **Boundaries age well**

The `brain.js` boundary has survived two model changes, one auth
refactor and a complete rewrite of the chat UI. Components that touched
`axios` directly did not survive half of that.

---

## 🔗 Related Documents

- [README](./README.md) — project overview
- [Architecture](./architecture.md) — system design and diagrams
- [API Contract](./api-contract.md) — endpoint specification

---

**Last updated:** September 2025  
**Author:** [Ongun Akay](https://ongunakay.com)
```