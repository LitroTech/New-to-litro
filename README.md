# Litro Architecture (Phase 1)

This repository now captures the architecture baseline for **Litro** based on the full product brief.

## 1) Product Scope Summary

Litro is a free Filipino small-store management app for sari-sari stores, boutiques, milk tea shops, and carinderias, with:
- Owner app (mobile/web)
- Facebook Messenger bot for staff and low-end devices
- In-app chatbot using the same bot brain as Messenger

Core constraints:
- Online-first, server as source of truth, cache as safety net
- Multi-device safe (no silent data loss)
- Basic/Pro/Ultra behavior and limits
- Very simple UX: no overwhelming screens

## 2) Recommended Tech Stack

- **Frontend (owner + staff app):** React Native (Expo) + React web compatibility layer
- **Backend API:** TypeScript + NestJS (or Fastify) in modular monolith form
- **Database:** PostgreSQL (primary relational store)
- **Cache/queues:** Redis (rate limits, ephemeral carts, jobs)
- **Object storage:** S3-compatible bucket for product/expense photos
- **Auth/session:** Store-scoped access code + staff profile tokens (no email/password in Basic)
- **Messaging integration:** Meta Messenger webhook + Graph API
- **AI provider abstraction:** OpenAI-compatible gateway with strict token/cost controls
- **Observability:** OpenTelemetry + structured logs + Sentry
- **Hosting:** Single cloud region first (PH-near) with managed Postgres/Redis

Why this stack:
- Fast delivery for a small team
- Strong schema guarantees for transactional POS/credit flows
- Easy path from MVP to scale without early microservice overhead

## 3) Service Boundaries (Modular Monolith)

Single backend, clear domain modules:

1. **Identity & Access Module**
   - Store creation
   - Join via QR/access code
   - Staff identities (app identity and Messenger PSID identity)
   - Owner PIN validation (void, sensitive actions)

2. **Catalog & Inventory Module**
   - Product CRUD
   - Stock mode per product (numerical vs descriptive)
   - Auto-deduct on checkout for numerical stock
   - Color-coded stock status logic

3. **Checkout & Transactions Module**
   - Cart lifecycle
   - Payment labels: Cash/GCash/Card/Credit
   - Submit/void flow (void remains visible as struck-through state)
   - Camera proof metadata support

4. **Credit Ledger Module**
   - Credit customer management
   - Credit sales from checkout
   - FIFO settlement for payments
   - Credits list sorting and paid-state behavior

5. **Money & Expenses Module**
   - Daily sales totals
   - Open/close cash drawer
   - Owner visibility toggles for staff-facing dashboard fields
   - Expense logs with optional photo

6. **Alerts & Coaching Module**
   - Day-1 urgent stock alerts
   - Behavioral insights after threshold (30 days or 100 tx)
   - Milestone celebrations + one-time nudges policy

7. **Bot Brain Module (Shared by App Chat + Messenger)**
   - Pattern-matching parser (primary path)
   - Cart intent interpreter (line-by-line accumulation)
   - AI escalation only for correction intents ("mali", "hindi", "wrong")
   - Pattern learning pipeline from approved AI corrections

8. **Messaging Adapter Module**
   - Messenger webhook ingestion
   - PSID mapping to staff identity
   - Outbound send/reply handling

9. **Reporting & Limits Module**
   - Tier gates (Basic/Pro/Ultra)
   - AI question quota (3/10/unlimited)
   - Usage metering and billing-ready events

## 4) Core Data Model (Initial)

Key entities and purpose:

- `stores` — store identity, language, tier, settings
- `store_access_codes` — current active join code + rotation history
- `staff_identities` — app-based identities (name-only onboarding)
- `messenger_identities` — PSID-linked identities
- `identity_links` — owner-approved merge between app + Messenger identities
- `products` — item catalog, price, photo, stock_mode, stock value/label
- `inventory_events` — stock adjustments and audit trail
- `carts` / `cart_items` — in-progress checkout state (app + bot)
- `transactions` / `transaction_items` — immutable submitted sales records
- `void_events` — owner-PIN authorized void actions
- `customers` — credit customer records (name required, phone optional)
- `credit_entries` — credit-origin transactions and outstanding balances
- `credit_payments` — payment records applied FIFO to credit entries
- `expenses` — owner expense logs
- `cash_drawer_sessions` — open/close drawer periods and amounts
- `alerts` — generated low-stock/business insights
- `coaching_events` — one-time nudge/milestone delivery guardrails
- `bot_patterns` — deterministic parser rules
- `bot_messages` — inbound/outbound chat audit
- `ai_usage_events` — token/cost/quota tracking

Design rules:
- Submitted transactions are immutable; corrections are represented as compensating records
- All critical financial operations are idempotent via request keys
- Every write is store-scoped for multi-tenant safety

## 5) One Backend for Messenger + App Chat

Shared flow:
1. Message enters from either channel (`APP_CHAT` or `MESSENGER`)
2. Bot Brain normalizes text and resolves actor identity
3. Pattern engine attempts deterministic parse
4. If correction intent detected, run one AI correction call (max 1)
5. Updated cart summary returned
6. Only explicit **Submit** creates transaction

Important guarantees:
- Same parser/business rules regardless of surface
- Same cart and checkout services used by UI and Messenger
- Messenger PSID is permanent identity key; app identity remains separate unless linked by owner
- Context payload for AI is capped (~500 tokens): store, top products, today summary, open credits, language

## 6) Critical Business Rules Captured in Architecture

- Stock mode is per product, not per store
- Credit is triggered only when payment method is Credit
- FIFO payment allocation for credit settlements
- App avoids the term “utang” in responses, but accepts it in user input
- Home screen keeps only three visible primary actions
- Access-code regeneration invalidates previous join access immediately

## 7) Delivery Plan (Architecture to Build)

1. Build modular monolith + database schema + migrations
2. Implement owner/staff onboarding and access-code lifecycle
3. Implement product + stock modes + checkout + void
4. Implement credit ledger + FIFO settlement
5. Implement Messenger adapter + shared bot brain
6. Add AI correction path + quota and token caps
7. Add alerts/coaching and dashboard permissions
8. Harden with idempotency, audit logs, observability, backups

## 8) Non-Goals for Phase 1

- Hardware auto-pairing workflows
- Supplier affiliates, embedded finance, micro-insurance, ad network
- Full white-label capability

These remain aligned to later roadmap phases.
