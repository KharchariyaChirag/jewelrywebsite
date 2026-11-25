# Product Requirements Document (PRD) — Phase 1 MVP
**Product:** Hybrid B2B & B2C Imitation Jewelry Marketplace  
**Target Market:** India Domestic Launch (Direct-from-Manufacturer Supply → Retail & Wholesale Buyers)  
**Primary Objective:** Ship a fast, SEO-driven, end-to-end working marketplace covering Supplier Onboarding & Cataloging, Hybrid Single/Bulk Ordering with Razorpay Payments, and Dual Shipping (Platform Courier + Self-Ship).

---

## 1. User Personas & Roles

| Role | Who They Are | Core Phase 1 Goal |
| :--- | :--- | :--- |
| **Buyer (B2C & B2B)** | Boutique owners, Instagram resellers, retail shops (bulk buyers), and individual shoppers (single-piece buyers). | Discover imitation jewelry via Google Search/filters, clearly see single vs. bulk tier prices, pay safely online, and track delivery. |
| **Supplier (Manufacturer / Wholesaler)** | Direct imitation jewelry manufacturers and factory wholesalers onboarded by the founder. | Register quickly, upload jewelry catalogs with variants & wholesale slabs, receive paid orders, and ship via Platform Courier or Self-Ship. |
| **Platform Admin** | Marketplace founder/operator. | Approve suppliers/products, oversee all orders & Razorpay payments, monitor fulfillment, and track supplier payouts. |

> **Design Simplification:** A single user account has a `role` (`BUYER`, `SUPPLIER`, `ADMIN`). Suppliers can also browse/buy if needed, and buyers do NOT need separate B2B vs B2C account types—**quantity determines pricing automatically** (with optional GSTIN input at checkout for B2B buyers requesting a tax invoice).

---

## 2. Core Functional Requirements (Phase 1)

### Module A: Authentication & Onboarding
- **`REQ-AUTH-01` Buyer Registration & Login:** Email + password registration and login (JWT access + refresh cookies). Optional phone number and company name/GSTIN during checkout or profile update.
- **`REQ-AUTH-02` Supplier Registration:** Business registration capturing Legal/Shop Name, Contact Person, Phone, City/State, Pickup Pincode, GSTIN (optional for small artisans, stored if provided), and Default Shipping Preference (`PLATFORM_COURIER`, `SELF_SHIP`, or `BOTH`).
- **`REQ-AUTH-03` Supplier Approval:** New suppliers start in `PENDING` status (or can be toggled `ACTIVE` by Admin with 1 click). Only `ACTIVE` suppliers have their products displayed publicly.

### Module B: Supplier Portal & Jewelry Catalog
- **`REQ-CAT-01` Imitation Jewelry Product Form:** Suppliers can create, edit, pause, or archive products with jewelry-specific structured fields:
  - Core: Title, Auto-generated SEO Slug, Category (e.g., *Necklace Sets, Earrings & Jhumkas, Bangles & Bracelets, Bridal Sets, Rings, Mangalsutra, Anklets*), Description, SKU, Stock Quantity.
  - Jewelry Attributes: `baseMetal` (Brass, Copper, Alloy, Silver-Lookalike), `plating` (Gold Plated, Rose Gold, Oxidised Silver, Rhodium, Matte Gold, Antique), `stoneType` (Kundan, AD/CZ, Pearl, Meenakari, Polki, None), `occasion` (Bridal, Festive, Party, Daily Wear), `weightGrams`, `setContents` (e.g., "1 Necklace, 1 Pair Earrings, 1 Maang Tikka").
  - Media: Multi-image upload (up to 8 images, drag/order, primary thumbnail selection).
- **`REQ-CAT-02` Product Variants (Lightweight):** Optional variants per product (e.g., Color: *Ruby Red / Emerald Green / Mint*, or Bangle Size: *2.4 / 2.6 / 2.8*) with per-variant SKU, stock, and optional price override.
- **`REQ-CAT-03` Hybrid B2C + B2B Tiered Pricing:**
  - `minOrderQty` (default `1` for B2C+B2B; suppliers can set e.g. `6` or `12` for **Bulk-Only** items).
  - `retailPrice` (base 1-unit price in INR) and `mrp` (optional strikethrough display price).
  - `wholesaleTiers`: Up to 5 quantity slabs (e.g., `6–23 pcs → ₹290/pc`, `24–99 pcs → ₹220/pc`, `100+ pcs → ₹175/pc`).
  - Backend enforces strictly ascending, non-overlapping quantity slabs with decreasing unit prices.
- **`REQ-CAT-04` Supplier Shipping Settings:** Supplier configures:
  - Allowed modes: `PLATFORM_COURIER`, `SELF_SHIP`, or `BOTH`.
  - For `SELF_SHIP`: Flat shipping charge (₹), optional free-shipping threshold (e.g., Free above ₹5,000), and estimated dispatch days.
  - Pickup address & pincode (mandatory for `PLATFORM_COURIER`).
  - Per-product parcel weight (grams) for accurate courier rate calculation.

### Module C: Public SEO Storefront & Discovery (Nuxt SSR)
- **`REQ-SEO-01` Server-Side Rendering & Metadata:** Homepage, Category pages (`/categories/[slug]`), Product Detail Pages (`/products/[slug]`), and Supplier Storefronts (`/suppliers/[slug]`) render complete SSR HTML with dynamic `<title>`, `<meta description>`, OpenGraph image, canonical URL, and `application/ld+json` (`Product`, `AggregateOffer`, `BreadcrumbList`).
- **`REQ-SEO-02` Auto-Generated XML Sitemap & Robots:** Dynamic `/sitemap.xml` listing all active categories, published products, and active supplier storefronts.
- **`REQ-PUB-01` Jewelry Browsing & Faceted Filtering:** Filter by Category, Order Type (`Single Piece Allowed` vs `Bulk Wholesale`), Price Range, Plating, Stone Type, Base Metal, Occasion, and Sort (`Newest`, `Price Low→High`, `Price High→Low`).
- **`REQ-PUB-02` Product Detail Page (PDP) Tier Calculator:**
  - Clearly displays single-piece price (if `minOrderQty == 1`) alongside a visual **Bulk Wholesale Price Table**.
  - Interactive quantity input shows real-time unit price, total price, GST breakdown, and upsell nudge (e.g., *"Add 3 more pcs to save ₹110/pc!"*).
  - Pincode shipping estimator showing available shipping option(s) and estimated charge.

### Module D: Hybrid Cart, Order Splitting & Razorpay Payments
- **`REQ-ORD-01` Smart Hybrid Cart:**
  - Supports items from multiple suppliers in a single cart.
  - Groups cart items visually **by Supplier** (essential so buyers understand per-supplier shipping options).
  - Enforces each product's `minOrderQty` and stock availability.
  - Automatically applies the correct B2B tier price as quantity changes.
- **`REQ-ORD-02` Checkout & Shipping Selection:**
  - Buyer enters Shipping Address (with 6-digit Indian pincode), Phone, and optional Business Name + GSTIN (for B2B GST invoice).
  - For each Supplier group in the cart:
    - If supplier supports `BOTH`, buyer selects either **Platform Courier** (live/estimated Shiprocket rate) or **Supplier Self-Ship** (supplier's flat/free rate).
    - If supplier supports only one mode, that mode is auto-selected and displayed transparently.
- **`REQ-ORD-03` Razorpay Payment Integration:**
  - Clicking **Pay Now** creates a `PENDING_PAYMENT` parent `Order` + supplier `SubOrders` and opens Razorpay Checkout (UPI, Cards, NetBanking, Wallets).
  - Payment confirmation is finalized via signature verification (`POST /api/v1/payments/verify`) AND backed by an idempotent Razorpay Webhook (`POST /api/v1/webhooks/razorpay`) so dropped browser tabs never lose paid orders.
  - On `PAID`, product/variant stock is decremented atomically and confirmation emails are sent to Buyer and Supplier(s).

### Module E: Dual Fulfillment & Shipping Workflow
- **`REQ-SHP-01` Platform-Managed Courier Flow (`PLATFORM_COURIER`):**
  1. Supplier sees paid `SubOrder` in `/supplier/orders`.
  2. Supplier clicks **"Confirm & Generate Shipping Label"**.
  3. Backend calls `ShiprocketProvider.createShipment(subOrder)` (or Mock Sandbox mode when testing locally), saves `awbCode`, `courierName`, `labelUrl`, and updates status to `READY_TO_SHIP`.
  4. Supplier prints label, hands parcel to pickup agent; status updates via Shiprocket webhook (or manual sync button) through `SHIPPED` → `DELIVERED`.
- **`REQ-SHP-02` Self-Ship by Supplier Flow (`SELF_SHIP`):**
  1. Supplier sees paid `SubOrder` marked `SELF_SHIP`.
  2. Supplier packs and dispatches via their own courier (DTDC, Shree Maruti, Tirupati, local bulk transport).
  3. Supplier clicks **"Mark Dispatched"** and enters: `courierName`, `trackingNumber` (or Transport LR Number), optional `trackingUrl`, and optional dispatch slip photo.
  4. Status moves to `SHIPPED`; Buyer receives tracking details and sees them on `/account/orders/[id]`. Supplier or Admin marks `DELIVERED` upon confirmation (Buyer can also click "Confirm Delivery").

### Module F: Minimal Admin Portal (Phase 1 Operations)
- **`REQ-ADM-01` Supplier & Product Management:** List suppliers, approve/suspend accounts, view/unpublish flagged products.
- **`REQ-ADM-02` Order & Payment Ledger:** View all Orders & SubOrders, payment statuses, shipping mode, and commission calculation (`platformCommissionAmount` stored per `SubOrder` so you know exact payout due to each manufacturer).
- **`REQ-ADM-03` Manual Payout Recording:** Mark a delivered `SubOrder` as `SETTLED` with bank transfer UTR reference (no automated bank-split APIs needed in Phase 1).

---

## 3. Explicitly Out of Scope for Phase 1 (Defer to Phase 2+)
To launch rapidly, do **NOT** implement these in Phase 1:
- Real-time buyer-supplier chat / WebSockets (use clear product specs + email/support contact in Phase 1).
- Custom Request for Quotation (RFQ) bidding engine.
- International multi-currency shipping & customs duties (India INR first).
- Cash on Delivery (COD) — *Prepaid Razorpay only in Phase 1 to eliminate fake B2C/B2B RTO losses.*
- Automated Razorpay Route split settlements (track ledger & settle weekly via NEFT/IMPS in Phase 1).
- Coupon engine, loyalty points, reviews/ratings, or AI image search.