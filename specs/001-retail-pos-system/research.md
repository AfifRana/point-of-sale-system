# Research: Retail Point of Sale (POS) System

**Feature**: 001-retail-pos-system | **Date**: 2026-08-28

This document records the technical decisions made during planning, resolving every
NEEDS CLARIFICATION item and the constitution TODOs. Each decision follows the format:
Decision → Rationale → Alternatives considered.

---

## 1. Technology Stack

**Decision**: TypeScript 5.x (strict) end-to-end; React 18 SPA client; Fastify server;
PostgreSQL 16 via Prisma; Zod schemas in a shared package; Vite build; Vitest + Playwright
testing.

**Rationale**: The constitution's hardest constraints — integer money (I), offline-first
(II), and hardware ports (VI) — all demand that identical domain logic run on the terminal
and the server. A single language with a shared package is the only way to guarantee one
money/pricing implementation instead of two that can drift. TypeScript strict mode catches
money-type confusion (cents vs. units) at compile time. Fastify + Prisma + Zod is a
boring, well-trodden combination with first-class JSON-schema validation that doubles as
contract documentation.

**Alternatives considered**:
- *Python (FastAPI) backend + React frontend*: two languages means the money module is
  implemented twice (or one side calls the other over HTTP — unacceptable offline).
  Rejected on Principle I risk.
- *C#/.NET + Blazor*: strong typing and good money handling, but Blazor WASM offline
  story adds payload weight on low-end tablets; team familiarity assumed lower.
- *Go backend + TS frontend*: same dual-implementation problem as Python.
- *Deno/Bun runtime*: promising but smaller ecosystem for POS-adjacent libraries
  (printing, WebUSB) — not worth the risk for v1.

---

## 2. Offline Architecture: Local-First with Event Queue

**Decision**: Each terminal keeps its working state in IndexedDB: a read-only catalog
snapshot, the active cart, a durable append-only queue of local events (sales, refunds,
voids, adjustments, audit entries), and local stock deltas. The checkout path performs
zero network calls. A background sync engine pushes queued events to the server in
batches and pulls catalog/config updates, with exponential backoff.

**Rationale**: Constitution Principle II requires the sale path to complete with zero
network calls and to survive process kill and reboot. IndexedDB is durable across
restarts, supports transactions (atomic event append + stock delta), and is available in
all target browsers. An append-only local event queue mirrors the server ledger design
(Principle III), so sync is a relay of immutable facts rather than state reconciliation —
financial events can never conflict.

**Alternatives considered**:
- *Service Worker API caching only (no local DB)*: caching makes reads fast but writes
  still need a queue; IndexedDB is required regardless, so SW is used only for the app
  shell (PWA install), not as the data layer.
- *localStorage for the queue*: synchronous, 5 MB cap, no transactions. Rejected.
- *OPFS (Origin Private File System)*: faster for large blobs but lower-level and less
  uniformly supported than IndexedDB for structured event records.
- *CRDT libraries (Automerge/Yjs)*: CRDTs shine for concurrent *mutable* state. Our
  financial events are immutable and append-only — a plain sequence + server sequence
  number is simpler and deterministic. Rejected as over-engineering (YAGNI, per Quality
  Gates).

---

## 3. "What happens when 500 queued events sync at once?"

**Decision**: The sync engine pushes events in batches (default 50 events per request),
sequentially per terminal, with per-event idempotency keys. The server ingests each batch
inside one database transaction per event (not per batch) so a mid-batch failure resumes
without duplication. Target: 500 events complete in under 60 seconds. Sync backlog depth
and last-successful-sync age are surfaced on the terminal health panel (Principle VIII).

**Rationale**: Batching amortizes HTTP overhead while keeping transactions small enough
to avoid lock contention on the ledger. Per-event idempotency (client-generated UUID +
operation type) makes retries safe: a retried event returns the original result
(Principle I). The 60-second target is derived from the constitution's alerting
requirement for sync backlog age — operators must see the backlog drain, not wonder.

**Alternatives considered**:
- *Single request with all 500 events*: one failure retries everything; large JSON
  payloads risk timeouts on poor store Wi-Fi. Rejected.
- *WebSocket stream*: adds connection-state complexity for a burst that happens rarely;
  plain HTTPS batches with backoff are simpler and debuggable with curl. Rejected for v1.

---

## 4. Money Representation

**Decision**: All monetary amounts are integers in minor units (e.g., cents), typed as
`Money = { amountMinor: number; currency: ISO4217Code }` in the shared package. All
arithmetic goes through `shared/domain/money.ts` functions with explicit rounding
(half-up to minor unit) applied at exactly three documented points: line total, tax per
line, and tender change. No `float`/`double` anywhere in a money field; lint rule
enforces it.

**Rationale**: Principle I is non-negotiable. JavaScript `number` is exact for integers
up to 2^53 — far beyond any retail amount — so integer cents are safe without BigInt
complexity. Centralizing rounding in one module makes the money-invariant test suite
(`sum(lines) + tax − discounts = total`; `sum(tenders) − change = total`) meaningful.

**Alternatives considered**:
- *BigInt*: unbounded, but JSON serialization requires string conversion everywhere and
  adds friction with no benefit at retail scale. Deferred.
- *Dinero.js / currency.js*: good libraries, but a 40-line typed module we fully control
  is easier to prove correct and audit than a dependency. Rejected (YAGNI).
- *Decimal columns in PostgreSQL + strings on the wire*: pushes rounding to the database,
  violating "single-point rounding" — rejected.

---

## 5. Tax Engine: Configurable Inclusive/Exclusive Mode

**Decision**: Tax is data: a `TaxRule` table with rate (basis points), mode
(inclusive/exclusive), and effective date ranges, scoped to the store. The mode in
effect at sale time is stamped on the sale record. Inclusive mode extracts tax from the
taxable base (`tax = base − round(base / (1 + rate))`); exclusive mode adds it
(`tax = round(base × rate)`). Both formulas live in `shared/domain/pricing.ts` with
table-driven tests covering both modes, mixed rates in one cart, and zero-rated items.

**Rationale**: Clarification 2026-08-28 made the mode a store setting. Constitution
Principle VII requires tax rules as versioned data with historical repricing — stamping
the mode (and rule version) on each sale guarantees a 2027 refund recomputes under 2026
rules. Basis points (integer) avoid float rates.

**Alternatives considered**:
- *Hardcoded single mode*: violates Principle VII and the clarification. Rejected.
- *Per-product mode override*: no requirement; adds combinatorial test surface. Deferred.

---

## 6. FISCAL_JURISDICTIONS TODO — Resolution

**Decision**: v1 ships one configurable tax regime per store (rate + mode + effective
dates). The `TaxRule` data model is jurisdiction-shaped (a `jurisdiction` field, default
"STORE"), so adding states/countries later is data entry plus conformance tests, not
re-architecture. Concrete jurisdiction enumeration is deferred until a multi-jurisdiction
deployment is scheduled.

**Rationale**: The store is single-location (spec assumption). Modeling the jurisdiction
field now costs nothing and honors Principle VII's "configuration, never code branches."
This is a scoped deferral, not a deviation — the constitution's requirement is that
jurisdictional behavior be data-driven, which v1 satisfies.

**Alternatives considered**:
- *Enumerating US states now*: speculative work with no v1 consumer (YAGNI). Rejected.
- *Ignoring jurisdiction modeling*: would force a schema migration later. Rejected.

---

## 7. PAYMENT_PROVIDERS TODO — Resolution

**Decision**: Card capture is terminal-integrated (clarification 2026-08-28). The POS
defines a `CardTerminalPort` interface (shared/ports): `charge(amount, idempotencyKey)`,
`refund(token, amount, idempotencyKey)`, `status()`. v1 ships (a) a deterministic fake
for CI and demos and (b) one real adapter targeting a browser-capable terminal protocol
(e.g., a payment terminal's local HTTP/WebSocket API). The POS stores only the outcome:
approved amount, processor token, last four, and approval code. No PAN/CVV ever enters
the system (PCI SAQ scope reduction by design).

**Rationale**: The constitution's security section mandates PCI-validated capture. The
port pattern (Principle VI) means the specific terminal vendor is a swappable adapter —
choosing one now does not lock the architecture. PSP reconciliation APIs are out of
scope; the port contract is the integration surface future providers implement.

**Alternatives considered**:
- *Hosted-payment-page redirect*: breaks offline checkout and adds a network dependency
  on the sale path. Rejected (Principle II).
- *Manual card entry with tokenizing JS SDK*: keeps SAQ scope but invites keying errors
  and slower checkout; the clarification chose terminal-integrated. Rejected.

---

## 8. Multi-Terminal Stock Coordination (1–5 stations)

**Decision**: Stock on-hand is a *derived* value: `initial + sum(sale deltas) + sum(refund
deltas) + sum(adjustments)`. Each terminal tracks local deltas optimistically and shows
"local view" stock; the server is authoritative. Oversell protection: at tender time the
terminal checks local stock and warns/blocks per policy (FR-013); the server never
rejects a completed sale (financial events are immutable) but flags negative stock as an
operational alert for manager review. Low-stock flags recompute from the event stream.

**Rationale**: With 1–5 terminals, pessimistic locking would add coordination complexity
for a rare race (two lanes selling the last unit of the same SKU within seconds). The
append-only ledger (Principle III) forbids rejecting committed financial events, so the
honest design is: warn locally, alert globally, adjust afterwards if needed. This is
deterministic and documented — the constitution's conflict-resolution requirement.

**Alternatives considered**:
- *Server-side reservation at cart-add time*: requires network on the sale path.
  Rejected (Principle II).
- *CRDT counters for stock*: stock is derived from immutable events, not merged mutable
  state; a plain sum is simpler and exactly reconcilable. Rejected.

---

## 9. Authentication Model

**Decision**: Admin/manager sign in with username + password (argon2id hashing,
server-side sessions with refresh). Cashiers sign in with a personal PIN (also
argon2id-hashed, minimum 6 digits) on the shared-terminal quick-switch screen. Sessions
are per-terminal with the signed-in user stamped on every event. Manager overrides
re-authenticate the *manager* (PIN or password), never just a prompt. Offline
authentication: the terminal caches a salted verifier allowing PIN login while
disconnected; all offline events are attributed and auditable.

**Rationale**: Clarification 2026-08-28 chose the dual model. The constitution requires
individual identity (no shared accounts) and server-side deny-by-default authorization —
the terminal enforces the same policy locally when offline (Principle II + Security).
PIN quick-switch keeps counter speed (Principle V): under 3 seconds to change operator.

**Alternatives considered**:
- *JWT access tokens only*: revocation on deactivation is awkward offline; server-side
  sessions + terminal session cache are simpler to invalidate. Rejected.
- *Badge/NFC hardware login*: extra hardware cost; not requested. Deferred.

---

## 10. Receipt Printing & Peripherals

**Decision**: All peripherals sit behind ports in `shared/ports`: `ReceiptPrinterPort`,
`BarcodeScannerPort` (keyboard-wedge default), `CashDrawerPort`,`CardTerminalPort`,
`CustomerDisplayPort`. The frontend ships a `FakePeripheralSet` (deterministic, CI-safe)
and real adapters: printer via ESC/POS over WebUSB/WebSerial (with browser print
fallback), drawer kick via printer interface, scanner via keyboard wedge (zero code —
scanners type + Enter). Printer failure never blocks tender: the receipt is queued in
IndexedDB and reprintable; peripheral state (connected/paper-low/offline) shows on the
health panel.

**Rationale**: Principle VI requires replaceable hardware and CI-runnable checkout with
zero physical devices. Keyboard-wedge scanners need no adapter at all — they emulate
typing — which keeps the most critical peripheral dependency-free. ESC/POS is the
receipt-printer lingua franca.

**Alternatives considered**:
- *Server-side printing via node-printer*: couples printing to server reachability —
  dead network would kill receipts. Rejected (Principle II).
- *PDF-only receipts*: slower at the counter; thermal print is the retail norm. Kept as
  fallback only.

---

## 11. Reporting from the Event Ledger

**Decision**: All reports (daily summary, by product, by category, low-stock, dashboard)
are computed from the append-only event ledger via SQL views; current-state tables are
caches that are rebuildable. Reports over periods containing unsynced offline sales show
a "last synced" watermark; totals recompute when sync completes. CSV export uses the
same SQL views, so on-screen and exported numbers are identical by construction.

**Rationale**: Principle III mandates that reported totals be derivable purely from the
event log. Deriving both screen and CSV from one view makes FR-045 (export matches
screen) structurally true rather than tested-after-the-fact.

**Alternatives considered**:
- *Reporting off denormalized tables only*: violates the rebuildability requirement.
  Rejected.
- *Separate analytics database*: unjustified at 20 tx/hour scale (YAGNI). Rejected.

---

## 12. CSV Catalog Import

**Decision**: Onboarding import parses CSV client-side (streaming, 10k-row capable),
validates every row against the same Zod schemas used by the API (duplicate SKU/barcode
against local snapshot, price format, required fields), shows a preview of valid and
rejected rows with reasons, and commits only after explicit confirmation. Initial stock
loads as an `initial` stock movement per product.

**Rationale**: Clarification 2026-08-28 chose manual + CSV import. Reusing the Zod
schemas guarantees import validation equals API validation — one rulebook. The
confirm-before-commit step prevents a bad spreadsheet from corrupting the catalog.

**Alternatives considered**:
- *Server-side import endpoint*: simpler client, but blocks onboarding on connectivity
  and adds an upload path for large files. Rejected for v1.
- *Excel (.xlsx) parsing*: extra dependency; CSV from any spreadsheet tool suffices.
  Deferred.

---

## 13. Shift & Drawer Accountability (added 2026-08-29)

**Decision**: A `Shift` is a first-class append-only record scoped to one terminal:
opening float, cash movements (drops, payouts, no-sale opens), and a close carrying the
counted cash. `expectedCashMinor` and `varianceMinor` are **always derived server-side
from the event ledger** — the client never supplies an expected figure. Cash sales are
gated on an open shift; card-only sales are not. At most one open shift per terminal.
Reopen creates a new linked Shift and never mutates the closed one.

**Rationale**: Constitution Principle I requires the release suite to prove
`drawer expected = opening float + cash in − cash out`; that invariant is meaningless
unless the expected value is computed from immutable events rather than stored state.
Principle III forbids mutating historical financial records, which forces the
reopen-as-new-record design. Gating only *cash* sales keeps card-only lanes usable
without shift ceremony.

**Alternatives considered**:
- *Store-wide shifts instead of per-terminal*: cannot attribute a drawer variance to a
  specific drawer or cashier, defeating the purpose. Rejected.
- *Client-computed expected cash*: a terminal could under-report a shortfall; violates
  Principle I's "money is sacred" posture. Rejected.
- *Mutable shift record updated on close*: violates Principle III append-only rule.
  Rejected.
- *Gating all sales on an open shift*: unnecessary friction for card-only transactions.
  Rejected.

---

## 14. Parked Carts & Cross-Terminal Handoff (added 2026-08-29)

**Decision**: A `ParkedCart` is **explicitly not a ledger event** — it carries no
financial or stock effect until tender. It is persisted in IndexedDB and synced to the
server so any terminal can list and retrieve it. Exclusive resume uses a **lock lease**
(`resumeLockTerminalId` + `resumeLockExpiresAt`): a resuming terminal acquires the lease,
and an expired lease is reclaimable so a crashed terminal cannot strand a customer's
basket. Concurrent resume attempts receive `409 CART_IN_USE`.

**Rationale**: Principle VIII mandates a "stuck-terminal handoff of an in-progress cart"
recovery path. Keeping parked carts out of the financial ledger is essential — a parked
cart is a UI convenience, and admitting it to the append-only ledger would pollute
reports with non-transactions. The lease (rather than a permanent lock) is what makes the
recovery path actually recover: a hard lock held by a dead terminal would be the very
failure mode the principle exists to prevent.

**Alternatives considered**:
- *Local-only parking (no server sync)*: fails the cross-terminal handoff requirement —
  a broken terminal's cart would be unreachable. Rejected.
- *Permanent exclusive lock with manual admin release*: turns a crash into a support
  ticket. Rejected.
- *Treating parked carts as draft sales in the ledger*: contaminates the immutable
  financial record with non-financial state. Rejected.

---

## 15. Price Override vs. Discount (added 2026-08-29)

**Decision**: A manager price override is modeled **on the SaleLine**, distinct from
discounts: `priceOverriddenFromMinor` (original), `priceOverrideReason`, and
`priceOverrideApproverId`. `unitPriceMinor` holds the overridden price, so all downstream
computation — tax, line total, and refund amount — flows from it naturally. Every
override emits a `PRICE_OVERRIDE` audit entry with before/after values.

**Rationale**: The constitution already names price override an individually auditable
sensitive operation, and FR-050 required the audit entry before any feature produced one —
this closes that gap. Keeping override separate from discount matters for reporting:
a manager needs to distinguish "we ran a promotion" from "we discounted damaged goods,"
and conflating them would hide margin leakage.

**Alternatives considered**:
- *Modeling override as an ad-hoc discount*: loses the audit distinction and corrupts
  discount-effectiveness reporting. Rejected.
- *Cashier override within a percentage limit*: adds a second policy threshold and a
  second approval path for marginal benefit; the clarification chose manager-only.
  Rejected for v1.

---

## 16. Retention & Data Rights (added 2026-08-29)

**Decision**: Two independent retention settings live in `StoreSettings`:
`financialRetentionYears` (default 7) and `piiInactivityAnonymizeYears` (default 2). A
scheduled job anonymizes customer PII once inactivity exceeds the threshold, emitting a
`PII_ANONYMIZED` audit entry; the linked sales survive as anonymized financial records.
Customer export (`GET /customers/{id}/export`) satisfies portability requests and is
itself audited.

**Rationale**: Constitution's privacy clause requires an explicit retention schedule plus
export and erasure support, while simultaneously forbidding erasure from destroying
financial records. Two separate periods are the only way to honor both: tax law wants the
transaction kept, privacy law wants the person forgotten. The existing PII/ledger
separation is precisely what makes independent periods implementable.

**Alternatives considered**:
- *Single retention period for everything*: either deletes tax records too early or holds
  PII far too long. Rejected.
- *Hardcoded periods*: violates Principle VII's "configuration, never code branches" and
  blocks deployment to a different regulatory regime. Rejected.
- *Manual-only erasure*: fails the "explicit retention schedule" requirement — a schedule
  nobody enforces is not a schedule. Rejected.

---

## 17. Whole-Unit Quantities (added 2026-08-29)

**Decision**: `quantity` on SaleLine, RefundLine, and cart lines is an integer ≥ 1.
Fractional quantities, unit-of-measure metadata, and scale peripherals are out of scope
for v1. The sync API rejects fractional quantities with `FRACTIONAL_QUANTITY`.

**Rationale**: Constraining quantity to integers removes an entire class of rounding
interaction — `unitPrice × fractionalQty` with tax in two modes and pro-rata refunds
multiplies the money test matrix substantially, and Principle I demands that matrix be
fully covered. The spec describes general small retail with no weighed goods. Adding a
per-product unit of measure later is additive (new nullable column, new port for the
scale) and does not invalidate existing sales rows.

**Alternatives considered**:
- *Decimal quantity everywhere now*: pays the full rounding-complexity cost for a
  capability no current requirement needs (YAGNI, per Quality Gates). Rejected.
- *Per-product unit of measure with a ScalePort*: the right eventual design, but
  speculative for v1 and would expand the money-invariant suite before the core is
  proven. Deferred, with the data model kept additive-friendly.

---

## 18. Testing Strategy Summary
**Decision**: Four suites, all blocking in CI: (1) Vitest unit/table-driven domain tests
(money, pricing, tax both modes, discount stacking, tender/change, refund eligibility,
drawer expected-cash — including zero, negative, max-qty, mixed-rate, boundary cases);
(2) contract tests for the sync API and every peripheral port (fake vs. interface),
including `NO_OPEN_SHIFT` rejection and concurrent parked-cart resume returning
`409 CART_IN_USE`; (3) integration tests for offline→online sync, mid-transaction power
loss (IndexedDB transaction replay), duplicate-request replay, concurrent catalog edits,
and shift close/reopen immutability; (4) Playwright E2E checkout with the
FakePeripheralSet, plus latency-budget performance checks (scan-to-render p95 < 100 ms,
tender-to-receipt p95 < 500 ms).

The money-invariant suite asserts three invariants, not two: order totals, tender/change,
**and drawer expected-cash** (SC-013).

**Rationale**: Principle IV is non-negotiable; the suites map one-to-one onto its
requirements. Table-driven tests are the only affordable way to cover retail's
combinatorial pricing/tax space.

**Alternatives considered**: none — this is the constitution's mandated minimum.

---

## Open Items Deferred to `/speckit-tasks`

- Exact keyboard shortcut map (Principle V) — UX detail, belongs in task breakdown.
- Seed data contents for demos/tests — implementation detail.
- Deployment topology (Docker Compose vs. single host) — operational, resolved at
  implementation.
