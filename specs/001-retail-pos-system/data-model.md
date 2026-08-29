# Data Model: Retail Point of Sale (POS) System

**Feature**: 001-retail-pos-system | **Date**: 2026-08-28

All monetary amounts are integer minor units (`amountMinor`) paired with an ISO 4217
`currency` code — never floating point (Constitution Principle I). All timestamps are
UTC with the originating local offset retained. The server stores an append-only event
ledger; current-state tables are derived caches. The terminal (IndexedDB) mirrors the
subset it needs.

---

## Entities

### Product

A sellable item in the catalog.

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key, client-assignable for offline creation |
| name | string | Required, 1–200 chars |
| sku | string | Required, unique among active products (case-insensitive) |
| barcode | string? | Optional, unique among active products when present |
| categoryId | UUID? | FK → Category, nullable (uncategorized allowed) |
| priceMinor | int | ≥ 0; in tax-inclusive mode this is the tax-inclusive shelf price |
| costMinor | int? | ≥ 0, optional (margin reporting) |
| taxRuleId | UUID | FK → TaxRule in effect for this product |
| reorderThreshold | int | ≥ 0, default 0 (0 = no low-stock flag) |
| isActive | bool | Soft-delete flag; inactive products hidden from checkout |
| createdAt / updatedAt | timestamp | UTC, immutable audit fields |

**Validation**: duplicate SKU/barcode among active products rejected (FR-002); price
edits never alter historical sales (FR-004 — sale lines snapshot the price).

### Category

A node in a nested tree organizing products.

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key |
| name | string | Required, 1–100 chars |
| parentId | UUID? | FK → Category (self); null = root; max depth 5 enforced |
| sortOrder | int | Display ordering among siblings |

**Rules**: deleting a category with products or children is rejected; products are
reassigned first. Tree queries roll up sales by category (FR-043).

### TaxRule

A versioned tax rule — data, never code branches (Principle VII).

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key |
| jurisdiction | string | Default "STORE"; future multi-jurisdiction support |
| name | string | Display name (e.g., "VAT standard", "Sales tax 8.25%") |
| rateBps | int | Basis points, 0–100000 (0%–1000%) |
| mode | enum | `INCLUSIVE` \| `EXCLUSIVE` |
| effectiveFrom / effectiveTo | timestamp | Date range; overlapping ranges for same jurisdiction rejected |
| isDefault | bool | One default rule per jurisdiction at a time |

**Rules**: the rule (and mode) effective at sale time is stamped on the sale and its
lines; historical sales always reprice under their stamped rule (FR-006).

### StockMovement (append-only)

Every stock change, traceable to its source (FR-031, FR-036).

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key, client-generated (idempotency) |
| productId | UUID | FK → Product |
| delta | int | Positive (in) or negative (out); never zero |
| source | enum | `SALE` \| `REFUND` \| `ADJUSTMENT` \| `INITIAL` |
| reason | string? | Required for ADJUSTMENT (damage/theft/count correction/supplier delivery); rejected otherwise empty (FR-033) |
| onHandBefore / onHandAfter | int | Snapshot for audit; server rejects ADJUSTMENT where after < 0 |
| actorId | UUID | FK → User |
| terminalId | UUID | Originating terminal |
| occurredAt | timestamp | UTC + local offset |
| correlationId | UUID | Links to the sale/refund event, or self for adjustments |

**Derivation**: on-hand = sum of deltas per product. Negative on-hand from concurrent
sales is permitted only via SALE source (see research §8) and raises an operational alert.

### Sale (append-only, immutable once complete)

A completed financial transaction.

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key, client-generated idempotency key |
| receiptNumber | string | Unique, human-readable, assigned at completion (e.g., `R-2026-000123`) |
| status | enum | `COMPLETED` \| `VOIDED` (pre-tender void only, FR-014) |
| cashierId | UUID | FK → User (signed-in cashier) |
| terminalId | UUID | Originating terminal |
| customerId | UUID? | FK → Customer, nullable (anonymous sale) |
| subtotalMinor | int | Sum of line totals before cart discounts |
| discountTotalMinor | int | Sum of all applied discounts |
| taxTotalMinor | int | Sum of line taxes |
| totalMinor | int | subtotal − discounts + tax (exclusive) or tax-extracted total (inclusive) |
| currency | ISO 4217 | Store currency |
| taxMode | enum | `INCLUSIVE` \| `EXCLUSIVE` — mode stamped at sale time |
| completedAt | timestamp | UTC + local offset |

**Invariant** (CI-enforced): `sum(line.lineTotalMinor) − cartDiscounts + tax = totalMinor`
and `sum(tenders) − changeGiven = totalMinor`.

### SaleLine

A line on a sale; snapshots pricing at time of sale.

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key |
| saleId | UUID | FK → Sale |
| productId | UUID | FK → Product (historical reference; product may later deactivate) |
| productName | string | Snapshot of name at sale time |
| unitPriceMinor | int | Price at sale time (inclusive of tax if mode = INCLUSIVE) |
| quantity | int | ≥ 1 |
| lineDiscountMinor | int | ≥ 0; line-level discount applied |
| taxRuleSnapshot | JSON | Rate, mode, name at sale time |
| taxMinor | int | Tax computed for this line |
| lineTotalMinor | int | unitPrice × qty − lineDiscount (pre-tax in exclusive mode) |
| returnedQuantity | int | Units already returned against this line (starts 0) |

**Rules**: `returnedQuantity ≤ quantity` enforced on every refund (FR-026a).

### Tender

A payment record on a sale.

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key |
| saleId | UUID | FK → Sale |
| method | enum | `CASH` \| `CARD` |
| amountMinor | int | ≥ 0 |
| cardToken | string? | Processor token (CARD only) — never PAN/CVV |
| cardLastFour | string? | Last four digits (CARD only) |
| approvalCode | string? | Terminal approval reference (CARD only) |
| idempotencyKey | UUID | Client-generated; retried tender returns original result |

**Rules**: split tender supported — multiple Tender rows per sale; change applies to
cash portion only (FR-010a, US-2 scenario 3).

### Refund (append-only, compensating entry)

A return against an original sale; original sales are never mutated (FR-030).

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key, client-generated idempotency key |
| originalSaleId | UUID | FK → Sale |
| refundNumber | string | Unique (e.g., `RF-2026-000045`) |
| method | enum | `ORIGINAL` \| `CASH` \| `STORE_CREDIT` |
| refundTotalMinor | int | Computed from prices actually paid (FR-025) |
| approverId | UUID? | FK → User (manager) — required when outside return window or above threshold (FR-028) |
| cashierId | UUID | FK → User (processing cashier) |
| reason | string | Required |
| createdAt | timestamp | UTC + local offset |

### RefundLine

A returned line within a refund.

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key |
| refundId | UUID | FK → Refund |
| saleLineId | UUID | FK → SaleLine |
| quantity | int | ≥ 1; `saleLine.returnedQuantity + quantity ≤ saleLine.quantity` enforced atomically (FR-026a) |
| amountMinor | int | Pro-rata share of price actually paid including discounts |

### Discount

A promotion definition.

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key |
| name | string | Display name |
| type | enum | `PERCENTAGE` \| `FIXED_AMOUNT` \| `SALE_PRICE` |
| value | int | Basis points (PERCENTAGE) or minor units (FIXED_AMOUNT/SALE_PRICE) |
| scope | enum | `STORE` \| `CATEGORY` \| `PRODUCT` |
| scopeRefId | UUID? | Category or Product id when scoped |
| requiresApproval | bool | Manager auth required at application (FR-022) |
| validFrom / validTo | timestamp? | Validity window; rejected outside (FR-021) |
| isActive | bool | Deactivated discounts unavailable at checkout |

### Customer

A person record; PII separable from the financial ledger (FR-040).

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key |
| name | string | Required |
| phone | string? | Searchable |
| email | string? | Searchable; format-validated |
| isAnonymized | bool | Erasure flag — PII replaced, sales preserved (FR-040) |
| createdAt | timestamp | UTC |

### User

A staff account; individual identity, no shared accounts (FR-048).

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key |
| username | string | Unique, required |
| displayName | string | Required |
| role | enum | `ADMIN` \| `MANAGER` \| `CASHIER` |
| passwordHash | string? | argon2id; required for ADMIN/MANAGER |
| pinHash | string? | argon2id; required for CASHIER (min 6 digits) |
| isActive | bool | Deactivated users cannot log in; attribution preserved (FR-049) |
| createdAt | timestamp | UTC |

### AuditEntry (append-only, hash-chained)

Tamper-evident record of every sensitive operation (Principle III).

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | Primary key |
| seq | bigint | Monotonic sequence — hash chain order |
| prevHash | string | Hash of prior entry — chain integrity |
| entryHash | string | Hash of this entry's canonical content |
| actorId | UUID | FK → User |
| terminalId | UUID | Originating terminal |
| action | enum | `PRICE_OVERRIDE` \| `DISCOUNT_THRESHOLD` \| `VOID` \| `REFUND` \| `NO_SALE_DRAWER` \| `STOCK_ADJUSTMENT` \| `USER_ADMIN` \| `PERMISSION_CHANGE` \| `AUTH_OVERRIDE` |
| beforeValue / afterValue | JSON | Snapshot of changed state |
| correlationId | UUID | Links to sale/refund/adjustment |
| occurredAt | timestamp | UTC + local offset |

**Rules**: no UPDATE/DELETE grants on this table to the application role; verification
job re-computes the chain and alerts on break.

### SyncLog (server)

Tracks per-terminal sync state.

| Field | Type | Rules |
|-------|------|-------|
| terminalId | UUID | Primary key |
| lastEventSeq | bigint | Last server sequence acknowledged by terminal |
| lastSyncedAt | timestamp | UTC |
| backlogDepth | int | Reported queue depth (health panel, Principle VIII) |

---

## State Transitions

### Sale

```text
CART (client, mutable)
  └─ tender confirmed ──→ COMPLETED (immutable; stock decremented atomically)
  └─ voided pre-tender ──→ VOIDED (no financial/stock effect; audited)
COMPLETED ──→ (refund flow) ──→ remains COMPLETED + linked Refund records
```

### Refund

```text
DRAFT (lines selected, eligibility validated)
  └─ manager approval if required ──→ PROCESSED (immutable; stock restored atomically)
  └─ cancelled ──→ no effect
```

### Product

```text
ACTIVE ──deactivate──→ INACTIVE (hidden from checkout; history intact)
INACTIVE ──reactivate──→ ACTIVE
```

---

## Terminal-Side Stores (IndexedDB)

| Store | Contents | Notes |
|-------|----------|-------|
| catalog | Product + Category + TaxRule snapshots | Read-only except during sync pull; encrypted at rest |
| events | Append-only queue: sale/refund/void/adjustment/audit events | One IndexedDB transaction = event + stock delta + cart clear (atomic) |
| stockDeltas | Local per-product deltas since last sync | Merged into server view on push |
| cart | Active cart(s) | Survives process kill (Principle II) |
| session | Cached auth verifier + current user | Enables offline PIN login |
| receipts | Receipt payloads awaiting print/email | Printer failure queues here (Principle VI) |

---

## Relationship Summary

```text
Category 1──n Product
TaxRule 1──n Product
Product 1──n StockMovement
Product 1──n SaleLine (via productId, historical)
Sale 1──n SaleLine
Sale 1──n Tender
Sale 1──n Refund
Refund 1──n RefundLine
RefundLine n──1 SaleLine
Customer 1──n Sale
User 1──n Sale (cashier), Refund (cashier/approver), StockMovement, AuditEntry
Discount n──1 Category | Product (scope)
```
