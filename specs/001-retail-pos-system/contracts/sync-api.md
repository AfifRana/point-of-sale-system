# Sync API Contract

**Feature**: 001-retail-pos-system | **Version**: 1.0

REST API between POS terminals and the server. Base URL: `/api/v1`. All bodies are JSON,
validated by shared Zod schemas. All money fields are integer minor units with ISO 4217
currency. All timestamps are ISO 8601 UTC.

**Idempotency**: every mutating request carries a client-generated `idempotencyKey`
(UUID). A retried request with a known key returns the original result with
`X-Idempotent-Replay: true` — never a duplicate effect (Constitution Principle I).

**Authorization**: `Authorization: Bearer <sessionToken>`. Roles: ADMIN, MANAGER,
CASHIER. Server-side deny-by-default; each endpoint lists its minimum role.

---

## Authentication

### POST /auth/login

Password login (admin/manager).

```json
{ "username": "string", "password": "string" }
```

→ `200 { "sessionToken": "string", "user": { "id": "uuid", "role": "ADMIN|MANAGER" } }`
→ `401` invalid credentials · `429` rate-limited (5 failures → 15 min lockout)

### POST /auth/pin

PIN login (cashier, shared terminal quick-switch).

```json
{ "pin": "6-8 digits" }
```

→ `200 { "sessionToken": "string", "user": { "id": "uuid", "role": "CASHIER" } }`
→ `401` · `429` (same lockout policy)

### POST /auth/override

Manager override for sensitive operations. Authenticates the **manager**, not the
cashier (FR-052).

```json
{ "managerUsernameOrPin": "string", "action": "DISCOUNT_THRESHOLD|REFUND|VOID|STOCK_OVERRIDE" }
```

→ `200 { "overrideToken": "string", "expiresInSeconds": 120 }`

---

## Sync (terminal ↔ server)

### POST /sync/push

Push a batch of queued local events. Batch size ≤ 50 events. Server ingests each event
in its own transaction; per-event failures are reported without aborting the batch.

**Role**: CASHIER (terminal session)

```json
{
  "terminalId": "uuid",
  "lastEventSeq": 12345,
  "events": [
    {
      "idempotencyKey": "uuid",
      "type": "SALE_COMPLETED | REFUND_PROCESSED | SALE_VOIDED | STOCK_ADJUSTMENT | SHIFT_OPENED | SHIFT_CLOSED | CASH_MOVEMENT | AUDIT",
      "occurredAt": "iso8601-utc",
      "localOffsetMinutes": 420,
      "payload": { "...": "type-specific, matches data-model.md entities" }
    }
  ]
}
```

→ `200`:

```json
{
  "results": [
    { "idempotencyKey": "uuid", "status": "ACCEPTED", "eventSeq": 12346 },
    { "idempotencyKey": "uuid", "status": "REPLAY", "eventSeq": 12340 },
    { "idempotencyKey": "uuid", "status": "REJECTED", "error": { "code": "string", "message": "string" } }
  ],
  "serverEventSeq": 12390
}
```

**Error codes**: `DUPLICATE_REFUND` (FR-026a), `INVALID_ADJUSTMENT` (FR-033),
`STALE_TAX_RULE`, `UNKNOWN_PRODUCT`, `SCHEMA_VIOLATION`, `NO_OPEN_SHIFT` (FR-052g),
`SHIFT_ALREADY_CLOSED`, `FRACTIONAL_QUANTITY` (FR-008 — quantities must be integers ≥ 1).

**Shift event payloads**: `SHIFT_OPENED` carries `{ shiftId, terminalId, openedById,
openingFloatMinor }`. `SHIFT_CLOSED` carries `{ shiftId, closedById, countedCashMinor }` —
the server derives `expectedCashMinor` and `varianceMinor` from the ledger and returns them,
never trusting a client-supplied expected value. `CASH_MOVEMENT` carries
`{ shiftId, type, amountMinor, reason }`.

**Contract test requirements**: duplicate push of the same batch returns REPLAY for all;
500-event backlog drains in ≤ 10 batches; a mid-batch server crash resumes without
duplication; a cash `SALE_COMPLETED` without an open shift is REJECTED with `NO_OPEN_SHIFT`.

### GET /sync/pull?sinceSeq={n}

Pull catalog/config changes since a sequence number. **Role**: CASHIER.

→ `200`:

```json
{
  "serverEventSeq": 12390,
  "changes": {
    "products": [ { "...": "Product entity, full snapshot per change" } ],
    "categories": [ { "...": "Category entity" } ],
    "taxRules": [ { "...": "TaxRule entity" } ],
    "discounts": [ { "...": "Discount entity" } ],
    "users": [ { "id": "uuid", "role": "...", "isActive": true } ],
    "storeSettings": [ { "...": "StoreSettings entity — retention, thresholds, policies" } ]
  }
}
```

**Conflict rule**: catalog/config is server-authoritative; the terminal snapshot is
replaced wholesale per entity (research §2). Financial events never flow down this
endpoint.

---

## Shifts & Cash Drawer

### GET /shifts/current?terminalId={uuid}

**Role**: CASHIER. → the open shift for that terminal, or `204` if none (checkout blocks
cash sales until one is opened, FR-052g).

### GET /shifts/{shiftId}/report

**Role**: MANAGER (own shift: CASHIER). → Z-report derived from the event ledger:

```json
{
  "shiftId": "uuid",
  "openedAt": "iso8601-utc", "closedAt": "iso8601-utc",
  "openingFloatMinor": 20000,
  "cashSalesMinor": 145030, "cardSalesMinor": 302500,
  "cashRefundsMinor": 1200, "dropsMinor": 100000, "payoutsMinor": 0,
  "expectedCashMinor": 63830, "countedCashMinor": 63500, "varianceMinor": -330,
  "transactionCount": 47
}
```

**Contract test**: `expectedCashMinor` MUST equal `openingFloat + cashSales − cashRefunds
− drops + payouts` (drawer invariant, SC-013), and MUST be recomputable from the ledger
alone with caches dropped.

### POST /shifts/{shiftId}/reopen

**Role**: MANAGER (requires `overrideToken`). Creates a new Shift linked via
`reopenedFromId`; the original closed shift is never mutated (FR-052f).

---

## Parked Carts

### POST /parked-carts

**Role**: CASHIER. Park an in-progress cart; returns its retrieval `reference`.

```json
{ "id": "uuid", "lines": [ { "productId": "uuid", "quantity": 2, "...": "discounts, overrides" } ], "customerId": null }
```

→ `201 { "reference": "P-042" }`

### GET /parked-carts

**Role**: CASHIER. → list of `PARKED` carts with reference, item count, total, parking
cashier, and age (FR-014b).

### POST /parked-carts/{reference}/resume

**Role**: CASHIER. Acquires the exclusive resume lock for the calling terminal.

→ `200 { "...": "cart lines and metadata", "resumeLockExpiresAt": "iso8601-utc" }`
→ `409 { "error": { "code": "CART_IN_USE", "message": "Cart P-042 is open on Terminal 2" } }` (FR-014b)

**Contract test**: two concurrent resume requests for the same reference — exactly one
succeeds, the other receives `409 CART_IN_USE`. An expired lock lease is reclaimable so a
crashed terminal cannot strand a cart.

---

## Customer Data Rights

### POST /customers/{id}/anonymize

**Role**: MANAGER. Removes PII, preserves anonymized financial records, emits a
`PII_ANONYMIZED` audit entry (FR-040).

### GET /customers/{id}/export

**Role**: MANAGER. → machine-readable file of stored details and purchase history;
emits a `CUSTOMER_EXPORT` audit entry (FR-040c).

---

## Catalog & Operations (online paths)

### POST /catalog/import — bulk CSV onboarding commit

**Role**: MANAGER. (Parsing/validation happens client-side; this commits validated rows.)

```json
{ "products": [ { "...": "Product entity + initialStock" } ] }
```

→ `200 { "imported": 850, "rejected": 12, "rejections": [ { "row": 41, "reason": "DUPLICATE_SKU" } ] }`

### GET /reports/daily-summary?date={YYYY-MM-DD}

**Role**: MANAGER. → gross, discounts, net, tax, refunds, transactionCount — derived
from the event ledger (research §11). Response includes `lastSyncedAt` watermark.

### GET /reports/sales-by-product?from={date}&to={date} · GET /reports/sales-by-category?from&to

**Role**: MANAGER. Ranked lists with units, revenue, refunds; category subtotals roll
up the tree.

### GET /reports/low-stock

**Role**: MANAGER. Products at/below reorder threshold with onHand, threshold,
suggestedReorder.

### GET /reports/drawer-variance?from={date}&to={date}

**Role**: MANAGER. Per-shift expected vs. counted cash with variance, for the
drawer-variance alerting threshold required by Constitution Principle VIII.

### GET /reports/export?report={id}&format=csv

**Role**: MANAGER. CSV generated from the same SQL views as on-screen reports (FR-045).

### GET /sales?from&to&cashierId&method&status · GET /sales/{receiptNumber}

**Role**: MANAGER (history), CASHIER (own-sales lookup for returns).

### GET /health/terminal/{terminalId}

**Role**: CASHIER. → `{ "backlogDepth": 0, "lastSyncedAt": "...", "clockSkewMs": 12 }`
(Principle VIII observability).

---

## Versioning

Breaking changes follow expand → migrate → contract across ≥ 1 release (Quality Gates).
Terminals may run mixed versions; the server accepts both old and new request shapes
during migration windows.
