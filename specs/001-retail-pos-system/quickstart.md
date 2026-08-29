# Quickstart: Retail Point of Sale (POS) System

**Feature**: 001-retail-pos-system | **Date**: 2026-08-28

Runnable validation scenarios proving the feature works end-to-end. These are the
demonstration paths for review and acceptance — implementation details live in
`tasks.md`.

**Prerequisites**: Node.js 20+, Docker (for PostgreSQL), a modern browser
(Chrome/Edge 120+ or Safari 17+). No physical POS hardware is required — all scenarios
run against the `FakePeripheralSet`.

---

## Setup

```bash
# 1. Start PostgreSQL
docker compose up -d

# 2. Install dependencies and seed demo data (catalog, tax rules, users)
npm run setup        # installs workspaces, runs migrations, seeds demo store

# 3. Run the full CI suite (unit + contract + integration + E2E with fakes)
npm run verify

# 4. Start the app for manual validation
npm run dev          # backend on :3001, frontend on :5173
```

Demo accounts (seeded): `admin` / `admin-pass` (ADMIN), `manager` / `manager-pass`
(MANAGER), cashier PIN `123456` (CASHIER).

---

## Scenario 1 — Cash sale with change (P1 core)

**Validates**: US-1, FR-007–FR-012, FR-052g, SC-001, SC-003.

1. Sign in with cashier PIN `123456` on the quick-switch screen.
2. Attempt a cash sale before opening a shift: the system blocks it and prompts to open one (FR-052g).
3. Open a shift declaring a $200.00 opening float.
4. Scan a known barcode (fake scanner: type SKU + Enter in the search field).
5. Verify the line renders in < 1 s with name, price, line total.
6. Set quantity to 2; verify totals update instantly. Confirm fractional quantities are rejected.
7. Tender $30.00 cash on a $25.00 cart; verify change due $5.00.
8. Confirm tender; verify receipt (number, lines, tax, tender, change) prints via fake
   printer.
9. Check the product's stock: decremented by exactly 2.

**Pass**: all nine checks hold; `npm run verify` money-invariant suite passes.

## Scenario 2 — Offline sale and sync (Constitution Principle II)

**Validates**: FR-017, FR-057, SC-011.

1. Open DevTools → Network → Offline (or stop the backend).
2. Complete a full cash sale — every step must work with zero network calls.
3. Verify the health panel shows "Offline — N events queued" (non-modal).
4. Restore connectivity; verify the queue drains and the sale appears in server-side
   sales history.
5. Kill the browser mid-sale (step 2), reopen: cart and queue survive.

**Pass**: no lost sales, no duplicates after sync, backlog indicator accurate.

## Scenario 3 — Return with refund and stock restore (P1)

**Validates**: US-3, FR-024–FR-030, SC-006.

1. Look up the Scenario-1 sale by receipt number.
2. Return 1 of the 2 units; verify refund amount uses the price actually paid.
3. Complete refund to original tender; verify refund receipt and stock +1.
4. Attempt to return 2 more units of the same line: rejected (only 1 remains
   returnable).

**Pass**: refund record linked to original sale; duplicate over-return blocked.

## Scenario 4 — Discount with manager approval (P2)

**Validates**: US-6, FR-018–FR-023.

1. As manager, create a 10% category discount with approval threshold.
2. As cashier, apply it at checkout; manager override prompt appears.
3. Authenticate as manager (not just a confirm click); discount applies.
4. Verify receipt itemizes the discount; verify tax recomputed.

**Pass**: unapproved application blocked; audit entry `DISCOUNT_THRESHOLD` recorded.

## Scenario 5 — 500-event sync burst (Constitution Principle II)

**Validates**: research §3 target.

```bash
npm run test:integration -- --grep "500 queued events"
```

**Pass**: 500 queued events drain in ≤ 10 batches, < 60 s, zero duplicates (idempotent
replay verified by re-pushing the same batch).

## Scenario 6 — Role enforcement (P3)

**Validates**: US-9, FR-047–FR-049, SC-010.

1. As cashier, navigate directly to `/admin/users` (typed URL): server returns 403 with
   a clear message — not a hidden menu.
2. As admin, create a manager account; sign in; verify manager capabilities.
3. Deactivate the account; verify login refused and historical attribution intact.

**Pass**: deny-by-default confirmed server-side on every restricted endpoint.

## Scenario 7 — Hardware failure mid-sale (Constitution Principle VI)

**Validates**: FR-059a, peripheral contracts.

1. Script the fake printer: `printer.failNext("OUT_OF_PAPER")`.
2. Complete a sale; tender must succeed despite print failure.
3. Verify receipt queued; fix printer; reprint from the receipt queue.

**Pass**: tender never blocked; receipt recoverable without restarting.

## Scenario 8 — Shift close with drawer variance (Constitution Principle I)

**Validates**: FR-052a–e, SC-013.

1. Open a shift with a $200.00 declared float.
2. Complete three cash sales totalling $85.50 and one card sale of $40.00.
3. Record a cash drop of $100.00 to the safe with reason "mid-day drop".
4. Close the shift entering counted cash of $185.00.
5. Verify expected cash = $200.00 + $85.50 − $100.00 = $185.50, and variance = −$0.50.
6. Open the Z-report: verify opening float, cash sales, card sales, drops, expected,
   counted, and variance are all present and the card sale is excluded from expected cash.
7. Drop the report caches and re-run: figures must be identical (ledger-derived).

**Pass**: the drawer invariant holds to the cent; variance is surfaced, not hidden.

```bash
npm run test:integration -- --grep "drawer invariant"
```

## Scenario 9 — Park a cart and resume on another terminal (Principle VIII)

**Validates**: FR-014a–c.

1. On Terminal A, scan three items, then park the cart; note the reference (e.g. `P-042`).
2. Verify the cart list shows the reference, item count, total, cashier, and age.
3. On Terminal B, retrieve `P-042`; verify all three lines and any discounts are intact.
4. While Terminal B holds it, attempt to resume `P-042` on Terminal A: expect a clear
   "already in use on Terminal B" message, not a silent duplicate.
5. Restart Terminal A's browser; verify parked carts still appear.
6. Complete the sale on Terminal B; verify stock decrements once, not twice.

**Pass**: exclusive resume enforced; no financial or stock effect until tender.

## Scenario 10 — Manager price override (Principle III)

**Validates**: FR-023a–b.

1. As cashier, add an item priced $50.00 and request a price override to $35.00.
2. Verify manager authentication is required and a reason is mandatory.
3. Authenticate as manager with reason "damaged packaging"; verify the line shows as
   price-overridden, distinct from a discount, and the receipt reflects it.
4. Complete the sale, then refund that line: verify the refund is $35.00, not $50.00.
5. Check the audit log: a `PRICE_OVERRIDE` entry exists with before ($50.00) and after
   ($35.00) values and the approving manager's identity.

**Pass**: override audited with before/after; refund uses the overridden price.

---

## Latency Budgets (CI-enforced)

```bash
npm run test:perf
```

- Scan-to-line-rendered p95 < 100 ms
- Tender-to-receipt-issued p95 < 500 ms (local, fake peripherals)

## References

- Entities and invariants: [data-model.md](./data-model.md)
- API and port contracts: [contracts/](./contracts/)
- Technical decisions: [research.md](./research.md)
- Requirements: [spec.md](./spec.md)
