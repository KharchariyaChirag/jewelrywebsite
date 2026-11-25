# Technical Architecture — Phase 1 MVP

## 1. High-Level System Architecture

```text
                        [ Googlebot / B2C & B2B Buyers / Suppliers ]
                                             │
                                           HTTPS
                                             ▼
                  ┌────────────────────────────────────────────────────┐
                  │         Nuxt 4 Frontend (Vue 3 + TS + SSR)         │
                  │  - Public SEO Storefront (SSR + JSON-LD + Sitemap) │
                  │  - Buyer Cart, Razorpay Checkout & Order Tracking  │
                  │  - Supplier Dashboard (Catalog + Dual Shipping)    │
                  │  - Minimal Admin Panel                             │
                  └──────────────────────────┬─────────────────────────┘
                                             │ REST JSON (/api/v1/*)
                                             ▼
                  ┌────────────────────────────────────────────────────┐
                  │     Node.js 22 + Express + TypeScript REST API     │
                  │  ┌─────────┐ ┌─────────┐ ┌───────────┐ ┌────────┐  │
                  │  │  Auth   │ │ Catalog │ │ Checkout  │ │Shipping│  │
                  │  │  & RBAC │ │ & Tiers │ │ & Orders  │ │Engine  │  │
                  │  └─────────┘ └─────────┘ └─────┬─────┘ └───┬────┘  │
                  │                                │           │       │
                  │  [ Prisma ORM ]                │           │       │
                  └────────┬───────────────────────┼───────────┼───────┘
                           │                       │           │
                           ▼                       ▼           ▼
                  ┌────────────────┐      ┌────────────┐  ┌──────────────────┐
                  │ MariaDB / MySQL│      │  Razorpay  │  │ Shiprocket API   │
                  │ (Primary DB)   │      │ (Payments  │  │ (Platform Courier│
                  └────────────────┘      │ & Webhooks)│  │  + Mock Adapter) │
                                          └────────────┘  └──────────────────┘
                           │
                  ┌────────┴────────┐
                  │ Cloudflare R2 / │
                  │ Local Uploads   │
                  └─────────────────┘
```

---

## 2. Monorepo Directory Structure

```text
jewelry-marketplace/
├── AGENTS.md
├── docs/
│   ├── PRD.md
│   ├── ARCHITECTURE.md
│   ├── PHASE_1_TASKS.md
│   ├── ANTIGRAVITY_PROMPTS.md
│   └── PROGRESS.md
├── backend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── .env.example                    # Template with DATABASE_URL for MariaDB
│   ├── .env                            # Local developer connection to existing MariaDB
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── seed.ts                     # Default jewelry categories + demo admin/supplier
│   └── src/
│       ├── index.ts                    # Server bootstrap
│       ├── app.ts                      # Express app, middleware, route registration
│       ├── config/                     # Env validation (Zod), Prisma client, R2/Storage
│       ├── middleware/                 # auth.middleware.ts, role.middleware.ts, error.middleware.ts
│       ├── utils/                      # pricing.util.ts (B2B tier math), slug.util.ts, jwt.util.ts
│       ├── modules/
│       │   ├── auth/                   # register, login, refresh, me
│       │   ├── suppliers/              # profile, shipping settings, public storefront
│       │   ├── categories/             # jewelry categories CRUD + public list
│       │   ├── products/               # supplier CRUD, variants, tiers, public search & PDP
│       │   ├── uploads/                # presigned R2 or local dev image upload
│       │   ├── cart/                   # server-validated cart & price tier preview
│       │   ├── orders/                 # checkout, multi-supplier splitting, buyer/supplier views
│       │   ├── payments/               # Razorpay order creation, signature verify, webhook
│       │   ├── shipping/               # ShippingService + ShiprocketProvider + SelfShipProvider
│       │   └── admin/                  # supplier approval, product moderation, payout ledger
│       └── tests/                      # Vitest unit & supertest integration tests
└── frontend/
    ├── package.json
    ├── nuxt.config.ts
    ├── .env.example
    └── app/
        ├── app.vue
        ├── layouts/
        │   ├── default.vue             # Public SEO header, search bar, cart badge, footer
        │   ├── supplier.vue            # Supplier portal sidebar
        │   └── admin.vue               # Admin portal sidebar
        ├── pages/
        │   ├── index.vue               # SEO Homepage (featured categories + wholesale highlights)
        │   ├── search.vue              # Faceted jewelry search & filter page
        │   ├── categories/[slug].vue   # Category SEO landing page
        │   ├── products/[slug].vue     # Jewelry PDP with B2C/B2B tier table + calculator
        │   ├── suppliers/[slug].vue    # Public supplier storefront
        │   ├── cart.vue                # Supplier-grouped hybrid cart
        │   ├── checkout.vue            # Address, per-supplier shipping picker & Razorpay pay
        │   ├── auth/                   # login.vue, register.vue, register-supplier.vue
        │   ├── account/                # orders/index.vue, orders/[id].vue
        │   ├── supplier/               # dashboard, products/, orders/, settings.vue
        │   └── admin/                  # index.vue, suppliers.vue, orders.vue
        ├── components/
        │   ├── ProductCard.vue         # Displays retail + lowest wholesale badge ("From ₹180/pc")
        │   ├── PriceTierTable.vue      # Visual B2B slab breakdown
        │   ├── ShippingModePicker.vue  # Platform Courier vs Self-Ship selector
        │   └── ImageUploader.vue
        ├── composables/                # useApi.ts, useAuth.ts, useRazorpay.ts
        └── stores/                     # auth.store.ts, cart.store.ts
```

---

## 3. MariaDB / MySQL Database Schema (Prisma Data Model)

- **Database Engine:** MariaDB (pre-installed locally on developer PC; natively installed on production VPS). MariaDB is fully compatible with MySQL client protocol.
- **Datasource Configuration:** Prisma provider is set to `"mysql"`.
- **Connection String:** Configured directly via `backend/.env`:
  `DATABASE_URL="mysql://${DB_USER}:${DB_PASSWORD}@localhost:3306/${DB_NAME}"` (developer configures their own local MariaDB credentials).
- **MariaDB / MySQL Compatibility Rules:**
  - Primary keys use `cuid()` or `uuid()`. Monetary fields use `Decimal @db.Decimal(12, 2)`.
  - JSON columns use Prisma `Json` type (MySQL/MariaDB native `json`, NOT PostgreSQL `jsonb` or `@db.JsonB`).
  - No scalar list types (PostgreSQL `String[]` is not supported; use explicit relation tables like `ProductImage` or `Json`).
  - Indexing: Standard B-Tree indexes and `FULLTEXT` index for catalog search (PostgreSQL GIN indexes are not supported).
  - Collation: Default collation is case-insensitive (e.g. `utf8mb4_unicode_ci` or `utf8mb4_general_ci`); Prisma queries must NOT use PostgreSQL-only `mode: 'insensitive'`.

### Core Enums
- `UserRole`: `BUYER`, `SUPPLIER`, `ADMIN`
- `SupplierStatus`: `PENDING`, `ACTIVE`, `SUSPENDED`
- `ShippingCapability`: `PLATFORM_COURIER`, `SELF_SHIP`, `BOTH`
- `ShippingMode`: `PLATFORM_COURIER`, `SELF_SHIP`
- `ProductStatus`: `DRAFT`, `PUBLISHED`, `ARCHIVED`
- `OrderPaymentStatus`: `PENDING`, `PAID`, `FAILED`, `REFUNDED`
- `SubOrderStatus`: `PENDING_PAYMENT`, `CONFIRMED`, `READY_TO_SHIP`, `SHIPPED`, `DELIVERED`, `CANCELLED`
- `PayoutStatus`: `PENDING`, `SETTLED`

### Table Specifications

| Model | Key Fields | Relations & Indexes |
| :--- | :--- | :--- |
| **`User`** | `id`, `email` (unique), `passwordHash`, `fullName`, `phone`, `role` (`UserRole`), `createdAt`, `updatedAt` | 1:1 `SupplierProfile`, 1:N `Order`, 1:N `Address` |
| **`SupplierProfile`** | `id`, `userId` (unique), `businessName`, `slug` (unique), `description`, `gstin`, `phone`, `addressLine`, `city`, `state`, `pincode`, `status` (`SupplierStatus`), `shippingCapability` (`ShippingCapability`), `selfShipFlatFee` (Decimal), `selfShipFreeAbove` (Decimal?), `dispatchDays` (Int, default 2), `commissionRate` (Decimal, default 5.00), `bankDetailsJson` (Json?) | FK `userId → User`; Index `(status, slug)` |
| **`Category`** | `id`, `name`, `slug` (unique), `description`, `imageUrl`, `sortOrder`, `isActive` | 1:N `Product`; Index `(slug, isActive)` |
| **`Product`** | `id`, `supplierId`, `categoryId`, `title`, `slug` (unique), `description`, `sku`, `status` (`ProductStatus`), `minOrderQty` (Int, default 1), `retailPrice` (Decimal), `mrp` (Decimal?), `lowestWholesalePrice` (Decimal — denormalized for fast sorting/display), `gstPercent` (Decimal, default 3.00), `stockQty` (Int), `weightGrams` (Int, default 100), `baseMetal`, `plating`, `stoneType`, `occasion`, `setContents`, `shippingCapabilityOverride` (`ShippingCapability?`), `createdAt`, `updatedAt` | FK `supplierId → SupplierProfile`, `categoryId → Category`; 1:N `ProductImage`, `ProductVariant`, `ProductPriceTier`; Indexes: `(status, categoryId, lowestWholesalePrice)`, `(supplierId, status)`, `(plating, stoneType, occasion)` |
| **`ProductImage`** | `id`, `productId`, `url`, `altText`, `sortOrder`, `isPrimary` | FK `productId → Product` (onDelete Cascade) |
| **`ProductVariant`** | `id`, `productId`, `name` (e.g., "Ruby Red / 2.6"), `sku`, `priceOverride` (Decimal?), `stockQty` (Int), `isActive` | FK `productId → Product` (onDelete Cascade) |
| **`ProductPriceTier`** | `id`, `productId`, `minQty` (Int), `maxQty` (Int?), `unitPrice` (Decimal) | FK `productId → Product` (onDelete Cascade); Unique `(productId, minQty)` |
| **`Order`** | `id`, `orderNumber` (unique, e.g., `ORD-2026-00012`), `buyerId`, `shippingAddressJson` (Json), `buyerGstin` (String?), `buyerCompanyName` (String?), `subtotalAmount`, `taxAmount`, `shippingAmount`, `totalAmount`, `paymentStatus` (`OrderPaymentStatus`), `razorpayOrderId` (unique?), `razorpayPaymentId`?, `paidAt`?, `createdAt` | FK `buyerId → User`; 1:N `SubOrder`; Index `(buyerId, createdAt DESC)` |
| **`SubOrder`** | `id`, `subOrderNumber` (unique), `orderId`, `supplierId`, `status` (`SubOrderStatus`), `subtotalAmount`, `taxAmount`, `shippingAmount`, `totalAmount`, `platformCommissionAmount`, `supplierNetAmount`, `shippingMode` (`ShippingMode`), `courierName`?, `awbCode`?, `trackingNumber`?, `trackingUrl`?, `shippingLabelUrl`?, `shiprocketOrderId`?, `dispatchSlipUrl`?, `shippedAt`?, `deliveredAt`?, `payoutStatus` (`PayoutStatus`), `payoutReference`?, `settledAt`?, `createdAt` | FK `orderId → Order`, `supplierId → SupplierProfile`; 1:N `OrderItem`; Index `(supplierId, status, createdAt DESC)` |
| **`OrderItem`** | `id`, `subOrderId`, `productId`, `variantId`?, `titleSnapshot`, `variantNameSnapshot`?, `skuSnapshot`, `imageSnapshot`?, `quantity`, `unitPrice`, `gstPercent`, `taxAmount`, `lineTotal`, `isWholesaleTier` (Boolean) | FK `subOrderId → SubOrder`, `productId → Product` |
| **`WebhookEvent`** | `id`, `provider` (`RAZORPAY` / `SHIPROCKET`), `externalEventId` (unique), `eventType`, `payloadJson`, `processedAt` | Ensures 100% idempotent webhook processing |

---

## 4. Hybrid Pricing Algorithm (`pricing.util.ts`)

```text
Input: product (retailPrice, minOrderQty, wholesaleTiers[]), optional variant (priceOverride), quantity

1. Validate: quantity >= product.minOrderQty (else throw 400 MOQ_NOT_MET)
2. Sort wholesaleTiers by minQty DESC
3. Find first tier where quantity >= tier.minQty AND (tier.maxQty is null OR quantity <= tier.maxQty)
4. If tier matched:
     unitPrice = tier.unitPrice
     isWholesaleTier = true
   Else:
     unitPrice = variant?.priceOverride ?? product.retailPrice
     isWholesaleTier = false
5. lineSubtotal = unitPrice * quantity
   taxAmount    = round2(lineSubtotal * (product.gstPercent / 100))
   lineTotal    = lineSubtotal + taxAmount
```

---

## 5. Dual Shipping Strategy Architecture (`modules/shipping/`)

Use a clean Strategy Pattern behind `IShippingProvider`:

1. **`GET /api/v1/shipping/quote`** (called during checkout per supplier group):
   - Computes total parcel weight (`sum(item.weightGrams * item.quantity)`).
   - If supplier allows `PLATFORM_COURIER`: calls `ShiprocketProvider.estimateRate(pickupPincode, deliveryPincode, weightKg)`.
     - *Sandbox/Dev Fallback:* If `SHIPROCKET_EMAIL` is unset, returns deterministic realistic Indian courier slab rates (e.g., ₹65 up to 500g + ₹35 per extra 500g, ETA 3–5 days) tagged `isMock: true`.
   - If supplier allows `SELF_SHIP`: calls `SelfShipProvider.estimateRate(supplier, subtotal)` → returns `0` if `subtotal >= selfShipFreeAbove`, else `selfShipFlatFee`.
2. **Fulfillment Dispatch:**
   - **`POST /api/v1/supplier/orders/:subOrderId/ship-platform`**: Calls `ShiprocketProvider.createOrderAndAssignAwb(subOrder)` → stores `awbCode`, `courierName`, `labelUrl`, transitions `SubOrder` to `READY_TO_SHIP`.
   - **`POST /api/v1/supplier/orders/:subOrderId/ship-self`**: Accepts `{ courierName, trackingNumber, trackingUrl?, dispatchSlipUrl? }` → validates with Zod, transitions `SubOrder` to `SHIPPED`, sets `shippedAt`.

---

## 6. REST API Specification (Phase 1 Endpoints)

| Domain | Endpoints | Auth |
| :--- | :--- | :--- |
| **Auth** | `POST /api/v1/auth/register`, `POST /api/v1/auth/register-supplier`, `POST /api/v1/auth/login`, `POST /api/v1/auth/logout`, `GET /api/v1/auth/me` | Public / Cookie JWT |
| **Public Storefront** | `GET /api/v1/public/categories`, `GET /api/v1/public/products` (with filters + pagination), `GET /api/v1/public/products/:slug`, `GET /api/v1/public/suppliers/:slug`, `GET /api/v1/public/sitemap-data` | Public |
| **Supplier Portal** | `GET/PUT /api/v1/supplier/profile`, `GET/POST /api/v1/supplier/products`, `GET/PUT/DELETE /api/v1/supplier/products/:id`, `POST /api/v1/uploads/image` | `SUPPLIER` |
| **Checkout & Payments** | `POST /api/v1/cart/calculate`, `POST /api/v1/shipping/quote`, `POST /api/v1/orders/checkout`, `POST /api/v1/payments/verify`, `POST /api/v1/webhooks/razorpay` | `BUYER` / Webhook |
| **Orders & Shipping** | `GET /api/v1/buyer/orders`, `GET /api/v1/buyer/orders/:id`, `POST /api/v1/buyer/orders/:subOrderId/confirm-delivery`, `GET /api/v1/supplier/orders`, `POST /api/v1/supplier/orders/:subOrderId/ship-platform`, `POST /api/v1/supplier/orders/:subOrderId/ship-self`, `PATCH /api/v1/supplier/orders/:subOrderId/status` | `BUYER` / `SUPPLIER` |
| **Admin** | `GET /api/v1/admin/dashboard`, `GET /api/v1/admin/suppliers`, `PATCH /api/v1/admin/suppliers/:id/status`, `GET /api/v1/admin/orders`, `PATCH /api/v1/admin/sub-orders/:id/settle` | `ADMIN` |

---

## 7. SEO Implementation Strategy (Nuxt 4)
1. **Clean Semantic URLs:** `/categories/bridal-necklace-sets`, `/products/kundan-gold-plated-bridal-choker-set-sku104`, `/suppliers/shree-krishna-imitation-jewels`.
2. **Dynamic Metadata:** Every category, product, and supplier page calls `useSeoMeta()` with keyword-rich titles (e.g., *"{Title} — Buy Single or Wholesale Bulk | {BrandName}"*), price range, and primary image.
3. **Structured Data (JSON-LD):** Inject `Product` schema with `AggregateOffer` (`lowPrice = lowestWholesalePrice`, `highPrice = retailPrice`, `priceCurrency = "INR"`, `availability = InStock`) so Google displays rich price snippets.
4. **Performance:** Serve compressed images with explicit `width`/`height` attributes and `loading="lazy"` below the fold; use SSR on public pages and CSR (`ssr: false` route rules) on `/supplier/**` and `/admin/**` dashboards to save server CPU.