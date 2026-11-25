# Step-by-Step Google Antigravity IDE Prompts (Phase 1 MVP)

> **How to use this playbook:**
> 1. Save `AGENTS.md` in your project root and put `PRD.md`, `ARCHITECTURE.md`, `PHASE_1_TASKS.md`, `ANTIGRAVITY_PROMPTS.md`, and `PROGRESS.md` inside `/docs/`.
> 2. Open the folder in **Google Antigravity IDE**.
> 3. Run **one prompt at a time** in order. Start a **fresh Antigravity session** for each sprint to keep context clean and fast.
> 4. Always review the **Implementation Plan Artifact** before approving code execution.

---

## Prompt 0 — Workspace Setup & Environment Bootstrap (Sprint 1: T1.1–T1.3)
```text
/plan Read AGENTS.md, docs/PRD.md, docs/ARCHITECTURE.md, and docs/PHASE_1_TASKS.md.
We are starting Sprint 1 (Tasks T1.1, T1.2, T1.3).
Note: MariaDB is already installed locally on the developer machine and will also be installed directly on the production server (no Docker DB container required).

Please create an Implementation Plan to:
1. Initialize the monorepo structure and backend environment config (backend/.env.example). Connection to the existing local MariaDB instance will be configured via DATABASE_URL in backend/.env.
2. Scaffold /backend with Node.js, Express, TypeScript (strict), Zod, Prisma ORM, Helmet, CORS, cookie-parser, Vitest, and tsx dev runner. Include GET /api/v1/health checking DB connectivity.
3. Write the complete backend/prisma/schema.prisma matching docs/ARCHITECTURE.md (datasource provider = "mysql", all enums, tables, Decimal(12,2) money fields, ProductImage relation table, and indexes; note MariaDB/MySQL does not support mode: 'insensitive' or scalar list arrays).
4. Write backend/prisma/seed.ts seeding 8 imitation jewelry categories (Necklace Sets, Earrings & Jhumkas, Bangles & Bracelets, Bridal Sets, Rings, Mangalsutra, Anklets, Hair Accessories), 1 Admin account, and 1 ACTIVE demo supplier with 4 realistic jewelry products (each with wholesale price tiers).
5. Provide backend/.env.example with DATABASE_URL="mysql://root:password@localhost:3306/jewelry_db".

Do NOT build auth endpoints or frontend yet. After I approve the plan, verify connectivity to the existing local MariaDB instance using the credentials in backend/.env, run Prisma migrate + seed, and verify `npm run typecheck` and GET /api/v1/health.
```

---

## Prompt 1 — Backend Auth & Role-Based Access Control (Sprint 1: T1.4)
```text
Read AGENTS.md, docs/ARCHITECTURE.md, and docs/PROGRESS.md.
Implement Task T1.4 strictly inside /backend.

Requirements:
1. Build backend/src/modules/auth/ with Zod validation:
   - POST /api/v1/auth/register (Buyer registration)
   - POST /api/v1/auth/register-supplier (Creates User with role SUPPLIER + SupplierProfile in PENDING status; include businessName, slug, phone, city, state, pincode, gstin?, shippingCapability)
   - POST /api/v1/auth/login (Sets HttpOnly, SameSite=Lax JWT cookie)
   - POST /api/v1/auth/logout
   - GET /api/v1/auth/me (Returns authenticated user + supplierProfile if applicable)
2. Build auth.middleware.ts and role.middleware.ts (requireAuth, requireRole, requireActiveSupplier).
3. Write Vitest integration tests covering buyer registration, supplier registration, login cookie flow, and 401/403 role protection.
4. Run `npm run typecheck` and `npm test`, then update docs/PHASE_1_TASKS.md and docs/PROGRESS.md.
```

---

## Prompt 2 — Nuxt 4 Frontend Scaffold & Auth Pages (Sprint 1: T1.5)
```text
Read AGENTS.md, docs/ARCHITECTURE.md, and docs/PROGRESS.md.
Implement Task T1.5 strictly inside /frontend.

Requirements:
1. Initialize Nuxt 4 (Vue 3, TypeScript strict, Tailwind CSS, Pinia, @nuxtjs/sitemap) inside /frontend with runtimeConfig pointing to the backend API (http://localhost:4000).
2. Create composables/useApi.ts (cookie-forwarding SSR-safe fetch wrapper) and stores/auth.store.ts.
3. Build clean, modern jewelry-brand layouts:
   - layouts/default.vue (Public header, search bar, links, auth/account menu, cart badge, footer)
   - layouts/supplier.vue (Sidebar for Products, Orders, Shipping Settings)
   - layouts/admin.vue (Sidebar for Overview, Suppliers, Orders)
4. Build /auth/login, /auth/register (Buyer), and /auth/register-supplier pages with validation and role-based redirects.
5. Add middleware protecting /supplier/** (SUPPLIER only) and /admin/** (ADMIN only).
6. Run `npm run typecheck`, start both servers, and use /browser to verify registering a buyer and logging in as the seeded demo supplier with zero console errors. Update docs/PHASE_1_TASKS.md and docs/PROGRESS.md.
```

---

## Prompt 3 — Backend Media Uploads, Pricing Engine & Supplier Catalog APIs (Sprint 2: T2.1–T2.4)
```text
/plan Read AGENTS.md, docs/PRD.md, docs/ARCHITECTURE.md, and docs/PROGRESS.md.
Implement Sprint 2 backend tasks (T2.1, T2.2, T2.3, T2.4) strictly inside /backend.

Requirements:
1. modules/uploads: POST /api/v1/uploads/image accepting multipart image upload (JPEG/PNG/WebP, max 5MB). Upload to Cloudflare R2 if R2 env vars exist, otherwise save to backend/uploads/ and serve statically at /uploads/*.
2. modules/suppliers: GET/PUT /api/v1/supplier/profile for updating business profile, pickup address/pincode, shippingCapability (PLATFORM_COURIER | SELF_SHIP | BOTH), selfShipFlatFee, selfShipFreeAbove, and dispatchDays.
3. utils/pricing.util.ts: Pure function calculating exact unitPrice, isWholesaleTier, lineSubtotal, GST taxAmount, and lineTotal given product, optional variant, and quantity. Enforce minOrderQty and validate non-overlapping ascending wholesale tiers. Write thorough Vitest unit tests for edge cases.
4. modules/products (supplier endpoints): GET/POST/PUT/DELETE /api/v1/supplier/products managing jewelry attributes (baseMetal, plating, stoneType, occasion, weightGrams, setContents), images, variants, and ProductPriceTier slabs. Automatically compute and store `lowestWholesalePrice` on save. Enforce strict `supplierId === req.user.supplierId` tenant isolation.
5. Write Vitest tests proving Supplier A cannot edit or delete Supplier B's products. Run `npm run typecheck && npm test` and update docs.
```

---

## Prompt 4 — Supplier Portal UI: Catalog & Shipping Settings (Sprint 2: T2.5)
```text
Read AGENTS.md, docs/PRD.md, and docs/PROGRESS.md.
Implement Task T2.5 strictly inside /frontend.

Requirements:
1. Build /supplier/settings allowing suppliers to configure their pickup pincode, address, GSTIN, and Dual Shipping Settings (Platform Courier, Self-Ship, or Both, plus Self-Ship flat fee & free-shipping threshold).
2. Build /supplier/products listing the supplier's jewelry catalog with stock, retail price, lowest wholesale price, status badge, and Edit/Archive actions.
3. Build /supplier/products/new and /supplier/products/[id]/edit featuring:
   - Jewelry spec selectors (Category, Base Metal, Plating, Stone Type, Occasion, Weight in grams, Set Contents)
   - Multi-image uploader component with primary image toggle
   - Dynamic Wholesale Price Tier Builder (add/remove quantity slabs with live preview)
   - Optional Variants section (Color/Size, SKU, stock, price override)
   - "Bulk Only" helper toggle (sets minOrderQty > 1)
4. Run `npm run typecheck`, then use /browser to log in as the demo supplier, create a new Kundan Necklace Set with 2 wholesale tiers, and confirm it saves and reloads cleanly. Update docs.
```

---

## Prompt 5 — Public SEO Storefront, Filters & Jewelry PDP (Sprint 3: T3.1–T3.5)
```text
/plan Read AGENTS.md, docs/PRD.md, docs/ARCHITECTURE.md, and docs/PROGRESS.md.
Implement Sprint 3 (Tasks T3.1–T3.5) across /backend (public read routes) and /frontend (SSR public pages).

Requirements:
1. Backend public endpoints (NO auth required, ONLY return PUBLISHED products from ACTIVE suppliers):
   - GET /api/v1/public/categories
   - GET /api/v1/public/products (supports q, categorySlug, plating, stoneType, baseMetal, occasion, orderType=single|bulk, minPrice, maxPrice, sort=newest|price_asc|price_desc, page, limit)
   - GET /api/v1/public/products/:slug
   - GET /api/v1/public/suppliers/:slug (supplier storefront + their published products)
   - GET /api/v1/public/sitemap-data
2. Frontend SSR pages (MUST use useAsyncData / useFetch for full SSR HTML):
   - / (Homepage: hero tailored to B2B wholesale + B2C single-piece buyers, category grid, featured jewelry grid)
   - /search and /categories/[slug] (Clean faceted sidebar filters + ProductCard showing retail price AND "Wholesale from ₹X/pc" badge)
   - /products/[slug] (Image gallery, variant picker, jewelry specs table, visual PriceTierTable, interactive quantity input with instant tier price + savings upsell banner, pincode shipping check, Add to Cart button, full useSeoMeta() + JSON-LD Product/AggregateOffer script)
   - /suppliers/[slug] (Public manufacturer storefront with city, shipping options badge, and catalog)
3. Configure @nuxtjs/sitemap to dynamically pull URLs from /api/v1/public/sitemap-data.
4. Verify SSR HTML contains product titles, prices, and JSON-LD, test filters in /browser, and update docs.
```

---

## Prompt 6 — Dual Shipping Engine, Checkout & Razorpay Backend (Sprint 4: T4.1–T4.4)
```text
/plan Read AGENTS.md, docs/PRD.md, docs/ARCHITECTURE.md, and docs/PROGRESS.md.
Implement Sprint 4 backend tasks (T4.1–T4.4) strictly inside /backend.

Requirements:
1. modules/shipping:
   - Define IShippingProvider interface.
   - Implement ShiprocketProvider (real Shiprocket API integration for rate estimation + create order/AWB, with automatic clean Mock Mode when SHIPROCKET_EMAIL is not configured in .env).
   - Implement SelfShipProvider (computes supplier's flat fee or free shipping above selfShipFreeAbove).
   - Expose POST /api/v1/shipping/quote accepting buyer pincode and supplier-grouped items.
2. modules/cart: POST /api/v1/cart/calculate returning server-verified per-item unitPrice, isWholesaleTier, GST, MOQ errors, and supplier groupings.
3. modules/orders: POST /api/v1/orders/checkout:
   - Recalculates ALL prices, wholesale tiers, GST, and selected shipping fees server-side.
   - Validates stock and per-supplier shippingMode compatibility.
   - Creates 1 Order + N SubOrders (calculating platformCommissionAmount and supplierNetAmount) + OrderItem snapshots.
   - Creates a Razorpay Order via Razorpay SDK (or mock test order ID if RAZORPAY_KEY_ID is set to test/mock) and returns checkout payload.
4. modules/payments:
   - POST /api/v1/payments/verify (verifies Razorpay HMAC signature with timingSafeEqual, transitions Order to PAID, SubOrders to CONFIRMED, decrements product/variant stock atomically inside prisma.$transaction).
   - POST /api/v1/webhooks/razorpay (uses raw request body for HMAC check + WebhookEvent table for idempotency).
5. Write Vitest tests covering multi-supplier order splitting, MOQ rejection, server-side price tampering prevention, and idempotent payment confirmation. Run tests and update docs.
```

---

## Prompt 7 — Hybrid Cart, Checkout & Razorpay Frontend (Sprint 4: T4.5)
```text
Read AGENTS.md, docs/PRD.md, and docs/PROGRESS.md.
Implement Task T4.5 strictly inside /frontend.

Requirements:
1. Build stores/cart.store.ts persisting cart items in localStorage and syncing totals via POST /api/v1/cart/calculate.
2. Build /cart page grouping items by Supplier, showing quantity steppers (respecting minOrderQty), live wholesale tier badges, upsell prompts ("Add X more pcs to unlock ₹Y/pc"), and GST breakdown.
3. Build /checkout page:
   - Shipping address form (Name, Phone, Address, City, State, 6-digit Pincode) + optional "Buying for Business? Add GSTIN & Company Name" toggle.
   - Per-supplier ShippingModePicker calling POST /api/v1/shipping/quote when pincode is entered, allowing selection between Platform Courier and Supplier Self-Ship whenever the supplier supports BOTH.
   - Razorpay Checkout integration (load checkout.js cleanly via composable useRazorpay.ts; on success call POST /api/v1/payments/verify and redirect to /account/orders/[id]?success=true).
4. Verify in /browser by adding single-piece and bulk items to cart, entering a pincode, toggling shipping modes, and completing a test checkout flow. Update docs.
```

---

## Prompt 8 — Dual Fulfillment (Platform Courier + Self-Ship) & Buyer Tracking (Sprint 5: T5.1–T5.4)
```text
/plan Read AGENTS.md, docs/PRD.md, docs/ARCHITECTURE.md, and docs/PROGRESS.md.
Implement Sprint 5 (Tasks T5.1–T5.4) across /backend and /frontend.

Requirements:
1. Backend Supplier Order Endpoints (strictly scoped to req.user.supplierId):
   - GET /api/v1/supplier/orders (list SubOrders with items, buyer shipping address, shippingMode, status)
   - POST /api/v1/supplier/orders/:subOrderId/ship-platform (only for PLATFORM_COURIER orders; calls ShiprocketProvider.createShipment, saves awbCode, courierName, shippingLabelUrl, sets status READY_TO_SHIP)
   - POST /api/v1/supplier/orders/:subOrderId/ship-self (only for SELF_SHIP orders; accepts courierName, trackingNumber, trackingUrl?, dispatchSlipUrl?, sets status SHIPPED + shippedAt)
   - PATCH /api/v1/supplier/orders/:subOrderId/status (allows valid transitions e.g. READY_TO_SHIP -> SHIPPED -> DELIVERED)
2. Backend Buyer Order Endpoints (strictly scoped to req.user.id):
   - GET /api/v1/buyer/orders & GET /api/v1/buyer/orders/:id (shows Order + all SubOrders with live shipping/tracking details)
   - POST /api/v1/buyer/orders/:subOrderId/confirm-delivery (marks SHIPPED -> DELIVERED)
3. Frontend Supplier UI (/supplier/orders):
   - Tabbed view (New Paid Orders, Ready/Shipped, Delivered).
   - Clear badge indicating PLATFORM COURIER vs SELF-SHIP.
   - Action button: "Generate Shipping Label" for Platform Courier, or "Enter Courier & Tracking/LR" modal for Self-Ship.
4. Frontend Buyer UI (/account/orders and /account/orders/[id]):
   - Clean timeline showing each supplier's sub-order status, AWB/LR number, tracking link, "Confirm Delivery" button, and a printable GST Order Summary.
5. Verify both Platform Courier and Self-Ship flows end-to-end in /browser. Update docs.
```

---

## Prompt 9 — Minimal Admin Portal & Payout Ledger (Sprint 6: T6.1–T6.2)
```text
Read AGENTS.md, docs/PRD.md, and docs/PROGRESS.md.
Implement Tasks T6.1 and T6.2 across /backend and /frontend.

Requirements:
1. Backend Admin Endpoints (strictly protected by ADMIN role):
   - GET /api/v1/admin/dashboard (Total GMV, Total Orders, Active/Pending Suppliers, Published Products, Pending Payout Total)
   - GET /api/v1/admin/suppliers & PATCH /api/v1/admin/suppliers/:id/status (Approve PENDING -> ACTIVE, or SUSPEND)
   - PATCH /api/v1/admin/products/:id/status (Unpublish/Archive any product)
   - GET /api/v1/admin/orders (All SubOrders with payment status, shippingMode, platformCommissionAmount, supplierNetAmount, payoutStatus)
   - PATCH /api/v1/admin/sub-orders/:id/settle (Record payoutReference UTR and mark payoutStatus = SETTLED)
2. Frontend Admin Pages (/admin, /admin/suppliers, /admin/orders):
   - KPI overview cards
   - 1-click Supplier Approve/Suspend table
   - SubOrder fulfillment & settlement ledger with "Mark Settled (UTR)" modal
3. Verify in /browser using the seeded Admin credentials. Update docs.
```

---

## Prompt 10 — Security Audit, Deployment Prep & Full End-to-End Sign-Off (Sprint 6: T6.3–T6.4)
```text
Read AGENTS.md, docs/PRD.md, docs/ARCHITECTURE.md, and docs/PHASE_1_TASKS.md.
Run Tasks T6.3 and T6.4:

1. Perform a security & bug audit across /backend and /frontend checking:
   - Every /supplier/* route enforces req.user.supplierId isolation
   - Checkout recalculates all prices, wholesale tiers, GST, and shipping server-side
   - Razorpay webhook verifies HMAC signature on raw body and is idempotent
   - Rate limiting is active on /auth/* and /orders/checkout
   - No secrets are hardcoded and .env.example files are complete
2. Create production-ready Dockerfiles for /backend and /frontend plus a concise docs/DEPLOYMENT.md explaining how to deploy on an Ubuntu VPS with server-installed MariaDB, Nginx/Caddy + SSL + Razorpay & Shiprocket live keys.
3. Run the entire automated test suite (`backend` Vitest + `frontend` typecheck) and execute a complete end-to-end test in /browser covering: New Supplier Register -> Admin Approve -> Product Upload -> Buyer B2C + B2B Cart -> Checkout with Platform Courier & Self-Ship -> Payment -> Dispatch -> Buyer Tracking -> Admin Settle.
4. Update docs/PHASE_1_TASKS.md and docs/PROGRESS.md with final Phase 1 launch status.
```