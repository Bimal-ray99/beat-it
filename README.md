# Beat it: Production-Grade Sports & Turf Platform Backend

> TypeScript monorepo · Express · MongoDB · Redis · BullMQ · Firebase · Socket.IO

![TypeScript](https://img.shields.io/badge/TypeScript-5.4-blue)
![Node](https://img.shields.io/badge/Node.js-20_LTS-green)
![Express](https://img.shields.io/badge/Express-5.2-lightgrey)
![License](https://img.shields.io/badge/license-MIT-green)

---

## What This Is

Tuff is a production-grade backend for a sports and turf booking platform targeting tier-2 and tier-3 Indian cities. Players book turfs, form teams, play ranked matches, bet on outcomes, and compete on city leaderboards for seasonal rewards.

9 distributed systems patterns, 18 MongoDB models, 89+ REST endpoints, real-time match events via Socket.IO, a complete player economy (TC + SP dual currency), commit-reveal anti-cheat, an ML-backed dispute system with jury fallback, and a background worker cluster handling rating computation, bet settlement, notifications, SP decay, and season lifecycle.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Clients · Mobile · Web                           │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ HTTPS / WebSocket
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Nginx — Edge Layer                                │
│        TLS termination · 3-zone rate limiting · HTTP→HTTPS          │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
         ┌─────────────────┴──────────────────┐
         ▼                                    ▼
┌──────────────────┐                ┌──────────────────────┐
│   apps/api       │                │   apps/workers        │
│   Express REST   │  ←─ BullMQ ──→ │   7 background jobs  │
│   + Socket.IO    │  ←─ Redis  ──→ │   rating · bets      │
│                  │                │   leaderboard · notif │
│   16 modules     │                │   sp-decay · season   │
│   89+ endpoints  │                │   outbox poller       │
│   Middleware:    │                │   3 cron jobs         │
│   JWT · RBAC     │                └──────────────────────┘
│   Idempotency    │
│   Circuit Breaker│
│   Rate Limiting  │
└────────┬─────────┘
         │
┌────────▼──────────────────────────────────────────────────┐
│                      Data Layer                            │
│                                                            │
│  MongoDB (18 collections)          Redis                   │
│  ┌──────────────────────────────┐  ┌──────────────────┐   │
│  │ Transactional sessions       │  │ 9 key patterns   │   │
│  │ Geospatial indexes           │  │ Sorted sets      │   │
│  │ TTL indexes (90d notifs)     │  │ Pub/Sub          │   │
│  │ Unique idempotency keys      │  │ Distributed lock │   │
│  │ Outbox pattern docs          │  │ Circuit breaker  │   │
│  └──────────────────────────────┘  └──────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

---

## Why These Patterns

| Pattern | Problem It Solves | Where |
|---|---|---|
| **Commit-Reveal Anti-Cheat** | Captain submits score, loses, edits it before submission is final. Score manipulation on ranked matches is undetectable. | `match-results.routes.ts` |
| **Outbox Pattern** | Payment webhook writes TC credit to queue. API crashes between DB write and `queue.add()`. User paid but TC never credited. | `outbox.worker.ts` |
| **Circuit Breaker** | FCM goes down. Every push notification waits 30s to timeout. Threads pile up. Notification worker drags down bet settlement. | `circuit-breaker.ts` |
| **Saga Orchestrator** | Ranked match toss collects stakes from both teams. Step 2 (teamB debit) fails after step 1 (teamA debit) succeeded. Player loses TC without a match starting. | `saga.ts` |
| **Dead Letter Queue** | Rating worker crashes on a malformed job. Job retries 3×, permanently fails. No trace. Dev never knows rating update was dropped silently. | `dlq.ts` |
| **Idempotent Webhook** | Razorpay sends duplicate webhook on payment. Two TC credits land. Player double-loaded. | `payments.routes.ts` |
| **Time-Bucketed Leaderboard Keys** | Monthly leaderboard reset requires cron that flushes Redis sorted set. Cron races with live writes. Window boundary = data loss. | `leaderboard-sync.ts` |
| **Redis Pub/Sub WebSocket Fan-out** | Match live event written on Instance A. Player's WebSocket connected to Instance B. Event dropped — live feed silent. | `websocket.ts` |
| **Distributed Cron Lock** | Two worker instances both run nightly leaderboard snapshot at 02:00 UTC. Double `seasonRank` writes, duplicate `rank_up` notifications. | `cron.jobs.ts` |

---

## Technical Decisions

Selected decisions with the most interesting tradeoffs:

**Dual currency (TC + SP)** — Tuff Coins are purchased with INR and spent on bookings, bets, and entry fees. Sport Points are earned only through gameplay and drive leaderboard rank. Separating them prevents pay-to-win: rank is always earned, not bought. Tradeoff: two ledgers to maintain, two transaction types to reconcile.

**Commit-reveal for score submission** — Both captains commit `SHA-256(scoreA|scoreB|nonce)` before either reveals. Reveal only accepted if hash matches commit. Conflict if both reveal but scores differ. This prevents a losing captain from seeing the winning captain's score before submitting their own. Tradeoff: two round trips required per match result.

**ML confidence threshold + jury fallback** — Disputed matches first go to ML model. If confidence ≥ 55%, auto-resolve. Below threshold, escalate to jury (7 eligible players, 5/7 majority required). Jury eligibility requires RAS ≥ 60. This removes admin from routine disputes while keeping humans for ambiguous cases.

**Outbox + idempotency on the same event** — Outbox guarantees at-least-once delivery from MongoDB to the economy worker. Unique `idempotencyKey` (e.g., `razorpay:{paymentId}`) on the OutboxEvent collection handles duplicate deliveries. E11000 duplicate key error = silently skip. Neither alone is sufficient: outbox without idempotency = potential double-credit; idempotency without outbox = event lost on crash.

**MongoDB unique constraint over Redis NX for idempotency** — Redis NX requires check-then-set, which has a race window between check and write under concurrent requests. MongoDB unique index enforces the constraint atomically at the storage layer with no race. Survives Redis restarts. One fewer dependency.

**Separate worker process** — `apps/workers` runs as a distinct Node.js process from `apps/api`. API crash or memory pressure does not kill in-flight rating computation or bet settlement. Workers scale independently. Shared `packages/db` ensures model changes propagate without code duplication.

**Time-bucketed Redis keys over cron reset** — Monthly leaderboard lives at `lb:city:monthly:2026-06`, weekly at `lb:city:weekly:2026-W23`. New month = new key. Old keys become stale naturally. No cron flush that races with live writes at month boundary. Tradeoff: old keys accumulate in Redis until expiry; query logic must construct the correct key from current date.

---

## Numbers

| Dimension | Count |
|---|---|
| REST endpoints | 89+ |
| Socket.IO event flows | 3 |
| MongoDB models | 18 |
| Redis key patterns | 9 |
| Distributed systems patterns | 9 |
| BullMQ queues | 6 + DLQ |
| Background workers | 7 |
| Cron jobs | 3 |
| Middleware layers per request | 7 |
| API route modules | 16 |

---

## Player Economy

```
INR ──[Razorpay]──► TC (Tuff Coins)
                      │
              ┌───────┴───────────┐
              ▼                   ▼
         Turf Bookings       Ranked Bets
         Entry Fees          Match Stakes
              │                   │
              └───────┬───────────┘
                      ▼
              SP (Sport Points)  ← earned only through gameplay
                      │
              ┌───────┴───────────┐
              ▼                   ▼
         City Leaderboard    Season Rewards
         Rank (ZSET)         (TC payouts to top 10)
```

SP awards per match:
- Win: `+100 SP`
- Play: `+30 SP`
- Graceful loss: `+10 SP`
- Weekly decay: `-50 SP` (inactive 30+ days)

Season top 10 TC rewards: `1500 / 800 / 400 / 150×7`

---

## Module Overview

| Module | Prefix | Notable Design |
|---|---|---|
| Auth | `/api/v1/auth` | Firebase phone OTP → ID token → JWT RS256 exchange |
| Users | `/api/v1/users` | Player code, wallet, trophies, wrapped stats, nemesis |
| Matches | `/api/v1/matches` | Commit-reveal toss, squad lock, live events (1/4s rate limit) |
| Match Results | `/api/v1/matches/:id` | Commit-reveal score, ML dispatch, jury escalation |
| Leaderboard | `/api/v1/leaderboard` | Redis ZSET primary path, 4 time windows, sport/locality filters |
| Betting | `/api/v1/matches/:id/bets` | Winner/MVP bets, odds, 10% platform cut on settlement |
| Bookings | `/api/v1/bookings` | Partial refund rules (full ≥2h, 50% <2h), reschedule (4h notice) |
| Payments | `/api/v1/payments` | Razorpay TC purchase, outbox idempotency, TC redemption |
| Teams | `/api/v1/teams` | Invite by code, availability polls, challenge flow, season stats |
| Social | `/api/v1/social` | PlayPal graph, DM/group conversations, feed, discovery |
| Turfs | `/api/v1/turfs` | Geo search, slot availability, legends, follower list |
| Owner | `/api/v1/owner` | Booking management, blacklist, analytics, waitlist |
| Campaigns | `/api/v1/campaigns` | Tournaments, bracket progression, team registration |
| Trainers | `/api/v1/trainers` | Geo search, availability, session booking, rating |
| Jury | `/api/v1/jury` | Eligible jurors (RAS ≥ 60), anonymous dispute voting |
| Admin | `/api/v1/admin` | Economy stats, dispute management, DLQ replay, season scheduling |

---

## Real-Time Events

```
Client ──► match:join       ──► server joins socket to match:{matchId} room
Client ──► match:leave      ──► server removes socket from room
Client ──► match:push_event ──► broadcast to room as match:event

Redis Pub/Sub ──► ws:broadcast ──► io.to(room).emit(event, data)
                  (cross-instance fan-out for multi-pod deployments)
```

All WebSocket connections require JWT in handshake auth. On connect, server auto-joins user to personal channel `user:{userId}` for targeted notifications.

---

## Background Workers

| Worker | Queue | Key Logic |
|---|---|---|
| **rating.worker** | `rating-updates` | TrueSkill update (winner/loser squads), SPR recompute, SP awards, RAS update, DLQ on failure |
| **bet.worker** | `bet-settlement` | Settle winner/MVP bets, distribute payout pool (90% to winners, 10% platform), DLQ on failure |
| **leaderboard.worker** | `leaderboard-refresh` | Sync match players to 7 Redis sorted sets (season/alltime/monthly/weekly/sport/locality/team) |
| **season.worker** | `season-wrapped` | Reward top 10 on season end, reset seasonSp, send push notifications |
| **sp-decay.worker** | `sp-decay` | Weekly SP penalty for players inactive 30+ days, concurrency 1 |
| **notif.worker** | `notifications` | FCM push dispatch (match result, rank up, jury duty), concurrency 10 |
| **outbox.worker** | (poll, no queue) | Poll `OutboxEvent` every 5s, process TC credit/debit, max 3 retries |

---

## Stress Tests

5 k6 scenarios assert correctness — not just throughput. CI fails if thresholds breach.

| Scenario | VUs | Key Assertion |
|---|---|---|
| Booking thundering herd | 100 concurrent, same slot | Exactly 1 booking created, rest get `409 SLOT_UNAVAILABLE` |
| TC concurrent debit | 50 debits, balance=500 | Balance never goes negative (atomic `$inc`) |
| WebSocket storm | 200 connections + broadcast | p95 latency < 200ms, 0 message drops |
| Score submit flood | 50 concurrent reveal calls | Exactly 1 result processed per match |
| Leaderboard burst | 100 concurrent rank reads | p99 < 100ms (Redis ZRANGE path) |

---

## Observability

```
Pino JSON logs     → request ID on every line (API + workers share req_id)
prom-client        → GET /metrics → Prometheus scrape → Grafana dashboard
OpenTelemetry SDK  → OTLP HTTP → Jaeger traces (auto-instrumented: HTTP, Mongoose, Redis)
```

Docker Compose profiles:

```bash
# API + workers + Mongo + Redis only
docker compose up

# Add Prometheus + Grafana + Jaeger
docker compose --profile monitoring up
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 LTS |
| Language | TypeScript 5.4 (strict mode) |
| HTTP | Express 5.2 |
| Real-time | Socket.IO 4.8 |
| Database | MongoDB · Mongoose 9 |
| Cache | Redis 5 (node-redis) |
| Queues | BullMQ 5 |
| Auth | Firebase Admin 13 (phone OTP) · JWT RS256 |
| Payments | Razorpay (via Axios) |
| File Upload | Multer → Firebase Storage |
| Validation | Zod 4 |
| Logging | Pino 9 · pino-http |
| Metrics | prom-client → Prometheus → Grafana |
| Tracing | OpenTelemetry SDK → Jaeger |
| Notifications | Firebase Admin (FCM) |
| Tests | Vitest · supertest · k6 |
| Containers | Docker · Docker Compose |

---

## Quick Start

```bash
git clone https://github.com/bimalray/tuff-backend
cd tuff-backend
npm install

# Copy env
cp .env.example .env

# Start full local stack (API + workers + Mongo + Redis)
docker compose up --build

# API:     http://localhost:3000
# Health:  http://localhost:3000/api/v1/health
# Metrics: http://localhost:3000/metrics
```

### Environment Variables

| Variable | Purpose |
|---|---|
| `MONGO_URI` | MongoDB connection string |
| `REDIS_URL` | Redis connection URL |
| `JWT_SECRET` | JWT signing secret |
| `FIREBASE_SERVICE_ACCOUNT_JSON` | Firebase Admin SDK credentials (JSON string) |
| `FIREBASE_STORAGE_BUCKET` | Firebase Storage bucket name |
| `RAZORPAY_KEY_ID` | Razorpay API key |
| `RAZORPAY_KEY_SECRET` | Razorpay API secret |
| `RAZORPAY_WEBHOOK_SECRET` | Razorpay webhook HMAC secret |
| `SEASON_END_DATE` | ISO 8601 datetime for season end scheduling |
| `PORT` | API server port (default 3000) |

---

## Repository Structure

```
tuff-backend/
├── apps/
│   ├── api/               Express REST API + Socket.IO
│   │   └── src/
│   │       ├── modules/   16 route modules
│   │       ├── middleware/ auth · validate · rate-limit · hmac
│   │       └── shared/    saga · circuit-breaker · websocket
│   └── workers/           BullMQ consumers + cron scheduler
│       └── src/
│           ├── workers/   7 background workers
│           ├── jobs/      cron.jobs.ts
│           └── shared/    dlq · leaderboard-sync · economy-tc
├── packages/
│   ├── db/                Mongoose models + connection
│   ├── redis/             Redis client + pub/sub
│   ├── types/             Shared types · QUEUE_NAMES · constants
│   └── config/            Environment validation (Zod)
├── infra/
│   ├── nginx/             nginx.conf (rate limiting, TLS, proxy)
│   └── docker/            Docker Compose (app + monitoring profiles)
└── tests/
    ├── integration/        Vitest + supertest API tests
    └── stress/             k6 correctness scenarios
```

---

[LinkedIn](https://www.linkedin.com/in/bimal-ray)
