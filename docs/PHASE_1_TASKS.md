# Phase 1 Implementation Checklist (End-to-End MVP)

Check off each task (`[x]`) **only after** passing typecheck, tests, and browser verification.

---

## Sprint 1: Monorepo Scaffold, Database & Auth Foundation
- [ ] **T1.1** Create monorepo structure (`/backend`, `/frontend`, `backend/.env.example` configured for pre-installed MariaDB).
- [ ] **T1.2** Initialize `/backend` (Node.js + Express + TypeScript + Zod + Prisma + Vitest + CORS/Helmet/CookieParser) and `/api/v1/health`.
- [ ] **T1.3** Write complete `backend/prisma/schema.prisma` matching `docs/ARCHITECTURE.md`, run initial migration against local MariaDB via `backend/.env`, and create `prisma/seed.ts` (8 imitation jewelry categories + 1 Admin user + 1 active Demo Supplier with 4 sample products).
- [ ] **T1.4** Implement `modules/auth` (Buyer register, Supplier register, Login, Logout, `GET /me`, bcrypt hashing, `HttpOnly` JWT cookies, RBAC middleware).
- [ ] **T1.5** Initialize `/frontend` (Nuxt 4 + Vue 3 + TypeScript + Tailwind CSS + Pinia + `@nuxtjs/sitemap`), configure `useApi` composable + `auth.store.ts`, and build `/auth/login`, `/auth/register`, and `/auth/register-supplier` pages.

**Sprint 1 Verification:** Buyer and Supplier can register/login, refresh the page with session intact, and `/api/v1/health` returns `200 OK` with DB connected.

---

## Sprint 2: Supplier Portal & Hybrid B2B/B2C Jewelry Catalog
- [ ] **T2.1** Implement `modules/uploads` (Cloudflare R2 / S3 upload with automatic local `/uploads` fallback in development, validating JPEG/PNG/WebP and max 5 MB).
- [ ] **T2.2** Implement `modules/suppliers` (`GET/PUT /supplier/profile` including pickup pincode, `shippingCapability`, `selfShipFlatFee`, `selfShipFreeAbove`).
- [ ] **T2.3** Implement `utils/pricing.util.ts` + Vitest unit tests verifying retail pricing, MOQ enforcement, B2B tier slab matching, and GST calculation.
- [ ] **T2.4** Implement `modules/products` supplier endpoints (CRUD products with jewelry attributes, multi-image ordering, variants, and `ProductPriceTier` slabs + automatic `lowestWholesalePrice` calculation).
- [ ] **T2.5** Build Supplier Portal UI (`/supplier/settings`, `/supplier/products`, `/supplier/products/new`, `/supplier/products/[id]/edit`) with an interactive **Wholesale Price Tier Builder** and **Image Uploader**.

**Sprint 2 Verification:** Supplier logs in, sets shipping rules, uploads a jewelry product with 3 images, 2 color variants, retail price ₹499, and bulk tiers (6+ pcs ₹299, 24+ pcs ₹219). Data persists accurately.

---

## Sprint 3: Public SEO Storefront & Product Discovery (Nuxt SSR)
- [ ] **T3.1** Implement `modules/products` & `modules/suppliers` public read APIs (`GET /public/categories`, `GET /public/products` with keyword search + jewelry attribute filters + `bulkOnly`/`singleAllowed` toggle + pagination, `GET /public/products/:slug`, `GET /public/suppliers/:slug`, `GET /public/sitemap-data`).
- [ ] **T3.2** Build Public Layout (`layouts/default.vue`) with search bar, category nav, cart badge, and mobile-responsive header.
- [ ] **T3.3** Build SSR Homepage (`/`), Search/Filter page (`/search`), Category page (`/categories/[slug]`), and Public Supplier Storefront (`/suppliers/[slug]`).
- [ ] **T3.4** Build SSR Product Detail Page (`/products/[slug]`) featuring:
  - Image gallery + variant selector + jewelry specs table
  - Visual **Single vs. Bulk Tier Pricing Table**
  - Live quantity calculator with wholesale upsell prompt
  - Pincode shipping estimator & "Add to Cart" button
  - Full `useSeoMeta()` + JSON-LD `Product`/`AggregateOffer` structured data
- [ ] **T3.5** Wire `@nuxtjs/sitemap` dynamic route fetching and verify SSR HTML via `curl` / View Source.

**Sprint 3 Verification:** Opening `/products/[slug]` renders full product details, pricing tiers, and JSON-LD in raw server HTML; filtering by Plating + Stone Type + Price Range works accurately.

---

## Sprint 4: Hybrid Cart, Multi-Supplier Checkout & Razorpay Integration
- [ ] **T4.1** Implement `modules/shipping` (`IShippingProvider`, `ShiprocketProvider` with mock fallback, `SelfShipProvider`, and `POST /api/v1/shipping/quote`).
- [ ] **T4.2** Implement `POST /api/v1/cart/calculate` so the frontend cart always displays server-verified tier prices, GST, and MOQ validation grouped by supplier.
- [ ] **T4.3** Implement `modules/orders` (`POST /api/v1/orders/checkout`) creating atomic `Order` + per-supplier `SubOrder` + `OrderItem` snapshots + Razorpay Order ID.
- [ ] **T4.4** Implement `modules/payments` (`POST /api/v1/payments/verify` + idempotent `POST /api/v1/webhooks/razorpay` with HMAC verification and atomic stock deduction inside a Prisma transaction).
- [ ] **T4.5** Build `cart.store.ts`, `/cart` (grouped by supplier with live tier upsell nudges), and `/checkout` (address + optional B2B GSTIN + per-supplier Platform Courier vs. Self-Ship picker + Razorpay Checkout modal + Order Success page).

**Sprint 4 Verification:** Buyer adds 1 pc from Supplier A (B2C price) and 12 pcs from Supplier B (B2B tier price), picks Platform Courier for A and Self-Ship for B, completes Razorpay Test Mode payment, and sees 1 Order split into 2 PAID SubOrders with decremented stock.

---

## Sprint 5: Dual Fulfillment (Platform Courier + Self-Ship) & Buyer Tracking
- [ ] **T5.1** Implement Supplier Order APIs (`GET /supplier/orders`, `POST /supplier/orders/:id/ship-platform`, `POST /supplier/orders/:id/ship-self`, `PATCH /supplier/orders/:id/status`).
- [ ] **T5.2** Implement Buyer Order APIs (`GET /buyer/orders`, `GET /buyer/orders/:id`, `POST /buyer/orders/:subOrderId/confirm-delivery`).
- [ ] **T5.3** Build Supplier Order Fulfillment UI (`/supplier/orders`) clearly separating Platform Courier orders ("Generate AWB & Print Label" button) and Self-Ship orders ("Enter Courier & Tracking/LR Number" modal).
- [ ] **T5.4** Build Buyer Order History & Tracking UI (`/account/orders`, `/account/orders/[id]`) displaying per-supplier shipment status, AWB/LR number, courier link, and GST invoice summary view (printable).

**Sprint 5 Verification:** Supplier A generates a Shiprocket AWB/label; Supplier B submits DTDC tracking details via Self-Ship; Buyer immediately sees both tracking updates on their order page.

---

## Sprint 6: Minimal Admin Portal, Hardening & Production Readiness
- [ ] **T6.1** Implement `modules/admin` (Dashboard KPIs, Supplier Approval/Suspension, Product Moderation, SubOrder Payout Ledger + Mark Settled with UTR).
- [ ] **T6.2** Build Admin Portal UI (`/admin`, `/admin/suppliers`, `/admin/orders`).
- [ ] **T6.3** Add security & operational polish: Express rate limiting on auth/checkout, input sanitization, supplier tenant isolation integration tests, and production `Dockerfile`s + `DEPLOYMENT.md` (with server-installed MariaDB setup) quick-start guide.
- [ ] **T6.4** Run full End-to-End Walkthrough in Antigravity Browser (Supplier signup → Admin approve → Product upload → Buyer B2C + B2B checkout → Razorpay test pay → Platform Ship + Self-Ship → Admin settle).

**Sprint 6 Definition of Done:** Complete Phase 1 cycle runs error-free end-to-end; all automated tests pass; ready to deploy to staging/production Ubuntu VPS.