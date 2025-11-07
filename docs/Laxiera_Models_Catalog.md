# Laxiera POS – Domain Models Catalog
**Version:** 2025-11-07 • **Owner:** Laxiera Technologies LLC • **Contact:** @laxiera-platform  
**Audience:** Backend, Android, Web Dashboard, Terminal, and Product teams.  
**Status:** Canonical reference for domain objects, lifecycles, events, and storage across all Laxiera surfaces.

> This is the **single source-of-truth** models catalog. It’s organized first by **merchant type** (what each vertical needs),
> then by **product surface** (where each model lives and how), and finally by **cross-cutting contracts** (IDs, money, time, sync, PCI/PII, auditing, events).

---

## 0) Contents
- [1) Merchant-Type Bundles (What each vertical needs)](#1-merchant-type-bundles-what-each-vertical-needs)
  - [1.1 Restaurants (Dine-In / Bar / QSR)](#11-restaurants-dine-in--bar--qsr)
  - [1.2 Retail](#12-retail)
  - [1.3 Wholesale / Distributor](#13-wholesale--distributor)
- [2) Platform Catalogs (Where models live)](#2-platform-catalogs-where-models-live)
  - [2.1 Backend (Authoritative)](#21-backend-authoritative)
  - [2.2 Android POS App (On-Device)](#22-android-pos-app-on-device)
  - [2.3 Dashboard App (Admin/Back-Office)](#23-dashboard-app-adminback-office)
  - [2.4 Standalone Android Terminal](#24-standalone-android-terminal)
  - [2.5 Customer App (Loyalty/Gift/Ordering)](#25-customer-app-loyaltygiftordering)
- [3) Payments, Settlements & Funds Flow](#3-payments-settlements--funds-flow)
- [4) Data Contracts, Sync & Offline](#4-data-contracts-sync--offline)
- [5) Security, PCI, PII & Compliance](#5-security-pci-pii--compliance)
- [6) Observability: Audit, Analytics, Webhooks](#6-observability-audit-analytics-webhooks)
- [7) Naming, IDs, Time & Money Conventions](#7-naming-ids-time--money-conventions)
- [8) Appendices: Field-Level Reference, Events, ERD, Indexing, Fixtures](#8-appendices-field-level-reference-events-erd-indexing-fixtures)

---

## 1) Merchant-Type Bundles (What each vertical needs)

> Each vertical lists **required** models, their **key fields**, **relationships**, and **lifecycle**.  
> “Server” = authoritative backend; “POS cache” = on-device subset; “Dashboard write” = back-office UI.

### 1.1 Restaurants (Dine-In / Bar / QSR)

#### Dining & Host
- **FloorPlan**  
  - Key fields: `id (ulid)`, `locationId`, `name`, `version`, `gridSize`, `rotation`, `updatedAt`  
  - Rel: 1:* Tables; 1:* ServerSections  
  - Lifecycle: draft → published (`publishedAt`, `publishedBy`)  
  - Storage: **Server**, **POS cache** (active plan), **Dashboard write**  
- **Table**  
  - Fields: `id`, `floorPlanId`, `label (unique per plan)`, `shape (round/rect)`, `size`, `position{x,y}`, `rotation`, `capacity`, `status (FREE/BOOKED/OCCUPIED)`, `occupiedSinceSec?`, `server?`, `sectionId?`  
  - Rel: 0–1 Order (current active), * Seat  
  - Storage: **Server**, **POS cache**  
- **Seat**  
  - Fields: `id`, `tableId`, `index`, `enabled`, `guestName?`  
  - Storage: **Server**, **POS cache**  
- **ServerSection**  
  - Fields: `id`, `floorPlanId`, `name`, `color`, `tableIds[]`  
  - Lifecycle: nightly reassign supported  
- **Reservation**  
  - Fields: `id`, `locationId`, `customerId?`, `name`, `partySize`, `slotStart`, `slotEnd`, `tablePreferences[]`, `status (PENDING/CONFIRMED/SEATED/CANCELLED/NO_SHOW)`, `notes`  
  - Events: `reservation.confirmed`, `reservation.seated`  
  - Storage: **Server**, **POS cache (windowed)**  
- **WaitlistEntry**  
  - Fields: `id`, `locationId`, `name`, `partySize`, `phone?`, `quotedMin`, `quotedMax`, `status (WAITING/SEATED/CANCELLED/LEFT)`  

#### Menu & Pricing
- **MenuCategory, MenuItem, Variant**  
  - Key: hierarchical tree; `MenuItem.route` for KDS routing; `Variant.sku?`  
- **MenuOptionGroup, MenuOption, Combo**  
  - Constraints: min/max select; required groups; priced options; combo entitlements  
- **Price (effective)**  
  - Fields: `id`, `entityType (ITEM/VARIANT/OPTION/COMBO)`, `entityId`, `currency`, `amount`, `startAt`, `endAt?`, `tier? (e.g., size)`, `channel (POS/ONLINE)`  
  - Server computes *effective* price windows; POS caches only selected effective prices.  
- **TaxRate, TaxGroup**  
  - `TaxGroup.rules[]` with `jurisdiction`, `startAt`, inclusivity, thresholds.

#### Orders & KDS
- **Order**  
  - Fields: `id`, `locationId`, `tableId?`, `customerId?`, `openedBy`, `startedAtSec`, `status (OPEN/COMPLETED/VOIDED)`, `subtotalCents`, `discountCents`, `taxCents`, `tipCents`, `totalCents`, `closedAtSec?`  
- **OrderItem**  
  - Fields: `id`, `orderId`, `catalogRef (item/variant)`, `qty`, `unitPriceCents`, `extendedPriceCents`, `course?`, `seatIndex?`, `firedAt?`, `voided?`  
- **OrderItemModifier**  
  - Fields: `id`, `orderItemId`, `optionRef`, `qty`, `priceDeltaCents`  
- **Course, RoutingRule**  
  - `Course.order`: appetizer/entree/dessert; `RoutingRule`: maps catalog routes to KDS stations.  
- **KDSTicket, KDSStation**  
  - Ticket = flattened production list; `status (QUEUED/IN_PROGRESS/READY/EXPO)`.

#### Payments & Drawer
- **PaymentIntent (ids only)**, **Payment (refs/status)**, **Refund**  
  - Processor-tokens only; see [§3](#3-payments-settlements--funds-flow).  
- **Tip, TipDistribution**  
- **CashDrawerSession, CashMovement**  
  - Enforce open/close with counted amounts, drops, safe pulls.

---

### 1.2 Retail

- **InventoryItem** (`sku`, `barcode[]`, `uom`, `costCents`, `isWeighed`)  
- **StockLevel** (`locationId`, `onHand`, `reserved`, `reorderPoint`)  
- **Supplier, PurchaseOrder, GoodsReceipt**  
- **StockMovement** (`RECEIVE/ADJUST/TRANSFER/SALE/RETURN`), **WasteLog**, **CostUpdate**  
- **Order, OrderItem** (no dining ties), **Discount**, **Promotion**  
- **PaymentIntent/Payment/Refund**, **ReceiptTemplate**, **Printer**

---

### 1.3 Wholesale / Distributor

- **Customer, CustomerGroup**, **PriceList**, **ContractTerm**  
- **Invoice, InvoiceItem** (AR), **Statement**  
- **ACH intents** via processor; **Subscription/Plan** optional  
- **Settlement, Payout, Reconciliation** (server-only finance)

---

## 2) Platform Catalogs (Where models live)

### 2.1 Backend (Authoritative)

**Identity & Settings**
- **Merchant** (`id`, legal, DBA, branding, TIN, `defaultCurrency`)  
- **Location** (`merchantId`, time zone, address, `businessHours[]`)  
- **Device/Terminal** (`pairingCode`, `lastHeartbeat`, `roles[]`)  
- **User/Role/Permission** (RBAC), **ApiKey**, **FeatureFlag**

**Payments & Finance (authoritative)**
- **ProcessorAccount** (Stripe/Adyen/Moov), **BankAccount** (tokenized), **SettlementConfig**  
- **FeePlan, FeeRule, SurchargeRule, ServiceCharge**  
- **PaymentIntent**, **Payment**, **Refund**, **Dispute/Chargeback**  
- **Settlement**, **Payout**, **Reconciliation**

**Catalog & Dining (authoritative)**
- Full **Menu** tree, **Price**, **TaxGroup**  
- **FloorPlan**, **Table**, **Seat**, **ServerSection**, **Reservation**, **WaitlistEntry**  
- **RoutingRule**, **KDSStation**, **KDSTicket**

**Inventory & Supply**
- **Supplier**, **PurchaseOrder**, **GoodsReceipt**  
- **InventoryItem**, **StockLevel**, **StockMovement**, **RecipeBOM**, **WasteLog**

**Ops, Audit & Analytics**
- **WebhookEvent**, **AuditLog**, **AnalyticsEvent**, **ErrorLog**, **ReportSnapshot**

> **Persistence:** Postgres (row-level security by tenant), Redis (caches/locks), S3 (binary), CDC to warehouse.

---

### 2.2 Android POS App (On-Device)

**Principles**
- Offline-first Room DB; **no secrets**, **no PAN**; only processor **tokens/ids**.  
- Sync slices **per-Location**; delta by `updatedAt`.

**Core Entities (Room)**
- `Session`, `UserLite`, `Device`, `Terminal`  
- `FloorPlan`, `Table`, `Seat`, `ServerSection`, `Reservation`, `WaitlistEntry`  
- `MenuCategory`, `MenuItem`, `Variant`, `OptionGroup`, `Option`, `Combo`  
- `PriceEffective`, `TaxEffective` (flattened, deduped)  
- `Order`, `OrderItem`, `OrderItemModifier`, `Split`, `Note`, `TableAssignment`  
- `PaymentRef`, `PaymentIntentRef`, `TipRef`  
- `Printer`, `ReceiptTemplate`  
- Optional: `InventoryItemLite`, `StockLevelLite`, `Shift`, `Timesheet`

**Local Compute**
- Totals, tax, discounts using effective rules; server validates on submit.  
- KDS tablet mode: `KDSTicket` subset.

---

### 2.3 Dashboard App (Admin/Back-Office)

- Next.js/React; TS types generated from contracts; RBAC enforced per route.  
- Bulk editors (catalog, price, tax), layout builders (floor plan), finance views (settlements/payouts), staff/time, inventory, webhooks.

---

### 2.4 Standalone Android Terminal

- Minimal Room: `Device`, `Terminal`, `Session`  
- Payment-only: `PaymentIntentRef`, `PaymentRef`, `TipRef`  
- Optional `OrderStub` for receipt association  
- Pairing/attestation flows; receipt printing.

---

### 2.5 Customer App (Loyalty/Gift/Ordering)

- `Customer`, `LoyaltyAccount`, `LoyaltyTransaction`, `GiftCard`  
- Menu browse, order place, pay (processor intents), track; coupons & promos.

---

## 3) Payments, Settlements & Funds Flow

**States**
- `PaymentIntent`: `CREATED → REQUIRES_ACTION? → PROCESSING → SUCCEEDED | FAILED | CANCELED`  
- `Payment`: mirrors processor webhook truth; contains `amount`, `tip`, `last4`, `brand`, `authCode`, `status`  
- `Refund`: `REQUESTED → PROCESSING → SUCCEEDED | FAILED`  
- `Settlement`: batch from processor; `Payout`: bank transfer; `Reconciliation`: matching engine  
- `Chargeback`: dispute lifecycle with evidences

**Events (outbound webhooks)**
- `payment.intent.created/updated`, `payment.succeeded/failed/refunded`, `payout.created/paid`, `order.closed`, `drawer.closed`

**Secrets**
- Store only **refs/ids**; never PAN/track/CVV; follow SAQ A boundaries.

---

## 4) Data Contracts, Sync & Offline

**Contracts**
- JSON Schema / OpenAPI + Protobuf for streaming.  
- Versioned endpoints; additive changes default; deprecations have 90+ days.

**Sync**
- **Delta** by `updatedAt` watermark; **tombstones** with `deletedAt`.  
- **Scope**: per `merchantId` + `locationId`.  
- **Conflict**: POS tentative totals; server authoritative on commit.  
- **Batch**: pagination with stable cursors; exponential backoff; resume tokens.

**IDs**
- `ULID`/`UUIDv7` with prefixes: `tbl_`, `ord_`, `pm_`, `stl_`, `pty_`, `loc_`.

---

## 5) Security, PCI, PII & Compliance

- PCI: SAQ A boundary; tokenization only; never store raw card data.  
- PII: at-rest AES-GCM; on-device SQLCipher; TLS 1.2+; DP masking in logs.  
- RBAC: Roles → Permissions → Enforced on all surfaces.  
- Data retention: configurable per tenant (invoices, receipts, logs).  
- Key rotation and device attestation for terminals.

---

## 6) Observability: Audit, Analytics, Webhooks

- **AuditLog**: `id`, `actor`, `action`, `target`, `ip`, `ua`, `before`, `after`, `ts`.  
- **AnalyticsEvent**: `name`, `props`, `userId`, `deviceId`, `ts`.  
- **WebhookEvent**: `type`, `payload`, `attempts`, `status`, `nextRetryAt`.  
- Idempotency keys on mutating APIs; DLQ for failed webhooks.

---

## 7) Naming, IDs, Time & Money Conventions

- Time on wire = RFC 3339 UTC; POS UI uses epoch seconds for timers.  
- Money = integer minor units (cents); taxes computed per line then rounded per merchant policy.  
- Enums upper snake on wire; sealed classes in Kotlin when valuable.  
- Slugs kebab-case; SQL table names snake_case.

---

## 8) Appendices: Field-Level Reference, Events, ERD, Indexing, Fixtures

### A) Field-Level Reference (selected highlights)

#### Order (server)
| Field | Type | Notes |
|---|---|---|
| id | string (ulid) | `ord_...` |
| locationId | string | Tenant scope |
| tableId? | string | Null for counter/retail |
| customerId? | string | Loyalty link |
| startedAtSec | int | Epoch seconds |
| status | enum | OPEN/COMPLETED/VOIDED |
| subtotalCents | int | Sum of lines (pre-discount) |
| discountCents | int | Applied discounts |
| taxCents | int | Computed by rules |
| tipCents | int | Gratuity |
| totalCents | int | Final total |
| closedAtSec? | int | If completed/voided |

#### Payment (server)
| Field | Type | Notes |
|---|---|---|
| id | string | `pm_...` |
| orderId | string | Optional for standalone |
| processor | enum | stripe/adyen/moov |
| intentId | string | Processor id |
| amountCents | int | Authorized/charged |
| tipCents | int | Tip |
| brand | string | masked brand |
| last4 | string | masked last4 |
| authCode | string | if available |
| status | enum | SUCCEEDED/FAILED/CANCELED |
| failureCode? | string | if failed |
| createdAt | ts | |

#### Table (server/POS)
| Field | Type | Notes |
|---|---|---|
| id | string | `tbl_...` |
| floorPlanId | string | |
| label | string | UI display |
| shape | enum | ROUND/RECT |
| size | json | {radius} or {width,height,corner} |
| position | json | {x,y} |
| rotation | float | degrees |
| capacity | int | seats |
| status | enum | FREE/BOOKED/OCCUPIED |
| occupiedSinceSec? | int | for timer |
| server? | string | display name |

> Full field dictionaries for all models live in `/contracts/schema/*.json`.

---

### B) Event Catalog (examples)

- `order.opened`, `order.item.added`, `order.item.voided`, `order.closed`  
- `payment.intent.created`, `payment.succeeded`, `payment.refunded`  
- `inventory.stock.updated`, `kds.ticket.ready`, `webhook.failed`

---

### C) Textual ERD (high-level)

```
Merchant 1--* Location 1--* FloorPlan 1--* Table 1--* Seat
Location 1--* Reservation, WaitlistEntry
Location 1--* MenuCategory 1--* MenuItem 1--* Variant
MenuItem *--* OptionGroup 1--* Option
Variant 1--* Price
Order 1--* OrderItem 1--* OrderItemModifier
Order 1--* Payment
Payment *--1 Settlement *--1 Payout
```

---

### D) Indexing Suggestions

- `order(locationId, status, startedAtSec)`  
- `payment(intentId) UNIQUE`, `payment(orderId)`  
- `table(floorPlanId, label)`  
- `price(entityType, entityId, startAt DESC)`  
- `inventory(stockLevel: locationId, sku)`

---

### E) Testing Fixtures

- Seed catalogs per vertical (restaurant, retail, wholesale).  
- Synthetic orders with random tips and discounts; payment events with success/failure mix.  
- Floor plans with mixed tables (round/rect), varied capacities, reservations, and waitlist.

---

## Surface Mapping Matrix

| Model | Backend | Android POS | Dashboard | Standalone Terminal | Customer App |
|---|:--:|:--:|:--:|:--:|:--:|
| Merchant/Location | ✓ | read | ✓ | read | — |
| User/Role/Perm | ✓ | read | ✓ | read | — |
| FloorPlan/Table/Seat | ✓ | ✓ | ✓ | — | — |
| Reservation/Waitlist | ✓ | ✓ | ✓ | — | — |
| Menu/Options/Combos | ✓ | ✓ | ✓ | read | ✓ (browse) |
| Price/Tax (effective) | ✓ | ✓ | ✓ | read | read |
| Order/Lines/Mods | ✓ | ✓ | ✓ | stub | ✓ (own orders) |
| Payments/Refunds | ✓ | refs | ✓ | ✓ | refs |
| Inventory (core) | ✓ | opt | ✓ | — | — |
| Loyalty/Gift | ✓ | ✓ | ✓ | — | ✓ |

---

**Change Log**  
- 2025-11-07: Extreme detail pass; added lifecycles, states, Room vs Server split, events, indexing, and fixtures.