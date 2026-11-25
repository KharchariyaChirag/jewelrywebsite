# AGENTS.md — Workspace Constitution (Imitation Jewelry B2B + B2C Marketplace)

> **Always-On Rules for Google Antigravity IDE Agents.**
> Read this file and `docs/PROGRESS.md` before planning or writing any code.

## 1. Project Identity & Phase 1 Scope
We are building a **Hybrid B2B + B2C Imitation Jewelry Marketplace** MVP.
- **Core Goal:** Complete an end-to-end live commerce cycle: Supplier Registration & Catalog Upload → SEO Public Storefront → Hybrid Single (B2C) & Tiered Bulk (B2B) Checkout → Platform Payment Gateway (Razorpay) → Hybrid Fulfillment (**Platform Courier** OR **Self-Ship by Supplier**).
- **Strict Scope Guardrail:** Do **NOT** build Phase 2+ features (no live WebSocket chat, no RFQ bidding marketplace, no Meilisearch/Elasticsearch, no microservices, no multi-currency FX, no AI pipelines, no mobile apps). Keep Phase 1 simple, fast, and shippable.

## 2. Locked Technology Stack
- **Monorepo Layout:**
  - `/backend` — Node.js 22 LTS + Express + TypeScript + Prisma ORM + MariaDB / MySQL + Zod + Razorpay SDK
  - `/frontend` — Nuxt 4 (Vue 3 + TypeScript + Tailwind CSS + Pinia + `@nuxtjs/sitemap`)
  - `/docs` — Single source of truth for requirements, architecture, tasks, and progress
- **Database & Connection:** MariaDB (Prisma `provider = "mysql"`). Local development connects directly to the developer's pre-installed MariaDB instance via `DATABASE_URL` in `backend/.env`. Production uses server-installed MariaDB.
- **Media Storage:** Cloudflare R2 / S3-compatible presigned uploads (with local `/uploads` dev fallback so development never stalls without cloud keys).
- **Search:** MariaDB / MySQL indexed queries + `LIKE` / `FULLTEXT` search (no external search engine in Phase 1; default collation is case-insensitive, do not use Postgres `mode: 'insensitive'`).
- **Shipping:** Strategy pattern (`ShippingProvider`) supporting `PLATFORM_COURIER` (Shiprocket adapter with built-in sandbox/mock mode) and `SELF_SHIP` (supplier flat/free shipping rules + manual courier/tracking entry).

## 3. Non-Negotiable Architecture Laws
1. **Multi-Supplier Order Splitting:** Every checkout creates **one parent `Order`** (tied to one Razorpay payment) and **one or more `SubOrder` records** (one per supplier in the cart). Shipping mode, shipping fee, fulfillment status, and tracking ALWAYS live on `SubOrder`, never on `Order`.
2. **Server-Side Price Truth:** Never trust cart prices, tier discounts, or shipping totals sent from the frontend. The backend MUST recalculate unit prices from `Product.retailPrice` and `ProductPriceTier`, GST, and shipping rates inside the checkout transaction.
3. **Tenant / Supplier Data Isolation:** Every `/api/v1/supplier/*` endpoint MUST filter queries and mutations by `req.user.supplierId`. A supplier must NEVER view, edit, or ship another supplier's products or sub-orders.
4. **Money Precision:** Store all currency amounts in MariaDB / MySQL as `Decimal(12, 2)` (INR). Convert to paise (`Math.round(amount * 100)`) only at the Razorpay SDK boundary.
5. **Payment Security:** Verify Razorpay payment signatures and webhook HMAC-SHA256 signatures using `crypto.timingSafeEqual`. Webhook handling MUST be idempotent.
6. **SSR & SEO First:** All public storefront pages (`/`, `/categories/[slug]`, `/products/[slug]`, `/suppliers/[slug]`, `/search`) MUST fetch data via Nuxt `useAsyncData` / `useFetch` so full HTML, canonical URLs, `useSeoMeta`, and JSON-LD `Product` schema render on the server.
7. **Validation & Error Standard:** Validate every API request body/query/params with **Zod**. Return consistent JSON errors: `{ error: { code: string, message: string, details?: unknown } }`.

## 4. Coding & Repository Hygiene
- **Scope Lock:** Only modify files inside the directories specified in the active task from `docs/PHASE_1_TASKS.md`. Do not refactor unrelated modules.
- **Type Safety:** Strict TypeScript (`"strict": true`) in both `/backend` and `/frontend`. Never use `any` without an explicit comment justifying why.
- **Database Changes:** Update `backend/prisma/schema.prisma`, generate a named Prisma migration (`npx prisma migrate dev --name <name>`), and update `docs/ARCHITECTURE.md` if the schema changes.
- **Secrets:** Never commit `.env` files or real Razorpay/Shiprocket keys. Always keep `.env.example` synchronized with every new environment variable.

## 5. Mandatory Verification Gate (Before Marking Any Task Done)
Before claiming a task is complete, agents MUST execute and pass:
1. Backend typecheck & build: `cd backend && npm run typecheck && npm test`
2. Frontend typecheck: `cd frontend && npm run typecheck`
3. Browser verification (for UI tasks): Open the running Nuxt app in Antigravity Browser, verify zero console/hydration errors, and confirm API network calls return `2xx`.
4. Documentation sync: Check off completed items in `docs/PHASE_1_TASKS.md` and append a concise entry to `docs/PROGRESS.md`.