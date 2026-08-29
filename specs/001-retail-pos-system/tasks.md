---
description: "Task list for Retail Point of Sale (POS) System implementation"
---

# Tasks: Retail Point of Sale (POS) System

**Input**: Design documents from `/specs/001-retail-pos-system/`

**Prerequisites**: [plan.md](./plan.md) (required), [spec.md](./spec.md) (required for user stories), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/)

**Tests**: TDD is MANDATORY for this feature — Constitution Principle IV (Test-First) is NON-NEGOTIABLE. Every implementation task is preceded by its failing-first test task. Tests must be written, run, and observed to FAIL before the corresponding implementation begins.

**Organization**: Tasks are grouped by user story (US1–US9) to enable independent implementation and testing of each story, in priority order P1 → P2 → P3.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

Web application structure per [plan.md](./plan.md): `backend/src/`, `frontend/src/`, `shared/`.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Monorepo initialization, tooling, and quality guardrails

- [ ] T001 Create monorepo workspace structure per plan.md: `backend/`, `frontend/`, `shared/` with root `package.json` workspaces
- [ ] T002 Initialize TypeScript 5.x strict config in `tsconfig.base.json` with `strict: true`, `noUncheckedIndexedAccess: true`, project references for all three workspaces
- [ ] T003 [P] Initialize `shared/package.json` with Zod dependency and build script
- [ ] T004 [P] Initialize `backend/package.json` with Fastify, Prisma, Zod, Vitest dependencies
- [ ] T005 [P] Initialize `frontend/package.json` with React 18, Vite, `idb`, Vitest, Playwright dependencies
- [ ] T006 [P] Configure ESLint + Prettier in `eslint.config.js` and `.prettierrc` with a custom rule banning `float`/`number` in money-typed fields (Principle I enforcement)
- [ ] T007 [P] Create `.gitignore` covering `node_modules/`, `dist/`, `build/`, `.env*`, `coverage/`, `playwright-report/`, `*.log`
- [ ] T008 [P] Create `.dockerignore` covering `node_modules/`, `.git/`, `dist/`, `coverage/`, `.env*`
- [ ] T009 [P] Create `docker-compose.yml` provisioning PostgreSQL 16 with a named volume for local development
- [ ] T010 Create `.env.example` documenting all required environment variables (database URL, session secret, IndexedDB encryption key derivation salt) with no real secrets committed

**Checkpoint**: Workspaces install, lint, and typecheck cleanly; `docker compose up -d` starts PostgreSQL

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### 2a. Shared Package — Money, Schemas, Ports (Principle I & VI keystone)

- [ ] T011 Write failing table-driven money tests in `shared/tests/money.test.ts` covering integer minor units, addition, multiplication by quantity, half-up rounding, zero, negative, max-quantity, and currency-mismatch rejection
- [ ] T012 Implement `Money` type and arithmetic in `shared/domain/money.ts` — `{ amountMinor: number; currency: ISO4217Code }`, explicit half-up rounding, no floating point (makes T011 pass)
- [ ] T013 Write failing table-driven tax tests in `shared/tests/tax.test.ts` covering INCLUSIVE extraction (`tax = base − round(base / (1 + rate))`), EXCLUSIVE addition (`tax = round(base × rate)`), zero-rated items, mixed rates in one cart, and basis-point precision
- [ ] T014 Implement tax computation in `shared/domain/tax.ts` with `TaxRule` application in both modes (makes T013 pass)
- [ ] T015 Write failing table-driven discount tests in `shared/tests/pricing.test.ts` covering PERCENTAGE, FIXED_AMOUNT, SALE_PRICE types, line-before-cart stacking order, floor-at-zero with over-application warning, and validity-window rejection
- [ ] T016 Implement discount stacking and cart pricing in `shared/domain/pricing.ts` (makes T015 pass)
- [ ] T017 Write failing table-driven tender tests in `shared/tests/tender.test.ts` covering cash change computation, split tender remaining balance, over-tender applying change to cash only, and card-never-overcharged invariant
- [ ] T018 Implement tender and change computation in `shared/domain/tender.ts` (makes T017 pass)
- [ ] T019 Write failing table-driven refund tests in `shared/tests/refund.test.ts` covering pro-rata refund of discounted price actually paid, return-window eligibility, over-return rejection, and already-returned-quantity tracking
- [ ] T020 Implement refund eligibility and amount computation in `shared/domain/refund.ts` (makes T019 pass)
- [ ] T021 Write failing money-invariant suite in `shared/tests/invariants.test.ts` asserting `sum(lines) − cartDiscounts + tax = total` and `sum(tenders) − change = total` across generated carts (property-based)
- [ ] T022 [P] Define Zod schemas for all entities in `shared/types/entities.ts` per data-model.md (Product, Category, TaxRule, StockMovement, Sale, SaleLine, Tender, Refund, RefundLine, Discount, Customer, User, AuditEntry)
- [ ] T023 [P] Define Zod schemas for sync API requests/responses in `shared/types/sync.ts` per contracts/sync-api.md
- [ ] T024 [P] Define peripheral port interfaces in `shared/ports/peripherals.ts` per contracts/peripheral-ports.md (ReceiptPrinterPort, BarcodeScannerPort, CashDrawerPort, CardTerminalPort, CustomerDisplayPort, PeripheralStatus)

### 2b. Backend — Schema, Ledger, Server Skeleton

- [ ] T025 Create Prisma schema in `backend/prisma/schema.prisma` with all entities from data-model.md, integer money columns, and append-only tables (Sale, SaleLine, Tender, Refund, RefundLine, StockMovement, AuditEntry)
- [ ] T026 Generate initial migration in `backend/prisma/migrations/` and verify up/down both succeed (Quality Gates: migration verification)
- [ ] T027 Add database grants migration in `backend/prisma/migrations/` revoking UPDATE and DELETE on append-only tables from the application role (Principle III enforcement at the database level)
- [ ] T028 Write failing audit hash-chain tests in `backend/tests/unit/audit-chain.test.ts` verifying `prevHash` linkage, tamper detection on modified rows, and chain verification across gaps
- [ ] T029 Implement hash-chained audit service in `backend/src/services/audit.ts` with `append(entry)` and `verifyChain()` (makes T028 pass)
- [ ] T030 Implement Fastify server skeleton in `backend/src/api/server.ts` with Zod schema validation, structured logging (correlation IDs, no PII/PAN), and error handling middleware
- [ ] T031 [P] Implement configuration loader in `backend/src/config.ts` reading environment variables with fail-fast validation and no hardcoded secrets
- [ ] T032 [P] Implement Prisma client wrapper in `backend/src/models/db.ts` with transaction helper for atomic multi-table writes (Principle I atomicity)
- [ ] T033 Write failing idempotency tests in `backend/tests/unit/idempotency.test.ts` verifying that a replayed key returns the original result with no duplicate effect
- [ ] T034 Implement idempotency store and middleware in `backend/src/services/idempotency.ts` (makes T033 pass)

### 2c. Frontend — Shell, Local Stores, Peripheral Fakes

- [ ] T035 Create Vite + React app shell in `frontend/src/app/main.tsx` and `frontend/src/app/App.tsx` with routing for Checkout, Products, Inventory, Customers, Reports, Admin
- [ ] T036 Configure PWA service worker in `frontend/vite.config.ts` for app-shell caching and offline install (shell only — not the data layer, per research §2)
- [ ] T037 Write failing IndexedDB store tests in `frontend/tests/unit/stores.test.ts` verifying atomic event append + stock delta in one transaction, and survival across simulated reload
- [ ] T038 Implement IndexedDB schema and repositories in `frontend/src/stores/db.ts` with stores: `catalog`, `events`, `stockDeltas`, `cart`, `session`, `receipts` (makes T037 pass)
- [ ] T039 [P] Implement WebCrypto encryption-at-rest wrapper in `frontend/src/stores/crypto.ts` for local data (Security section: terminals are physically accessible)
- [ ] T040 [P] Implement `FakePeripheralSet` in `frontend/src/services/peripherals/fakes.ts` — deterministic fakes for all five ports with `reset()` and scripted failure injection per contracts/peripheral-ports.md
- [ ] T041 [P] Implement `KeyboardWedgeAdapter` in `frontend/src/services/peripherals/scanner.ts` (scanner emits keystrokes + Enter; barcodes treated as untrusted, length-capped input)
- [ ] T042 [P] Implement design tokens and base UI primitives in `frontend/src/components/ui/` with 44 px minimum touch targets, WCAG 2.1 AA contrast, and no color-only status encoding (Principle V)
- [ ] T043 Implement global keyboard shortcut manager in `frontend/src/app/shortcuts.ts` enabling mouse-free operation of the entire sale flow (Principle V)
- [ ] T044 Implement terminal health panel in `frontend/src/components/HealthPanel.tsx` showing sync backlog depth, last successful sync, peripheral status, and pending offline transaction count (Principle VIII)

### 2d. CI Pipeline — All Blocking Gates

- [ ] T045 Create CI workflow in `.github/workflows/ci.yml` running lint, typecheck, and unit tests on every push
- [ ] T046 [P] Add money-invariant gate job to `.github/workflows/ci.yml` running `shared/tests/invariants.test.ts` as a blocking check
- [ ] T047 [P] Add hardware-fake checkout gate job to `.github/workflows/ci.yml` running the full E2E suite with zero physical hardware attached (Principle VI)
- [ ] T048 [P] Add offline/sync scenario gate job to `.github/workflows/ci.yml` running the integration suite
- [ ] T049 [P] Add latency budget gate job to `.github/workflows/ci.yml` failing on scan-to-render p95 ≥ 100 ms or tender-to-receipt p95 ≥ 500 ms
- [ ] T050 [P] Add dependency and secret scanning job to `.github/workflows/ci.yml` blocking on known critical vulnerabilities in shipped paths
- [ ] T051 [P] Add migration up/down verification job to `.github/workflows/ci.yml`

**Checkpoint**: Foundation ready — shared domain logic proven by tests, database append-only, terminal stores durable, all CI gates active. User story implementation can now begin.

---

## Phase 3: User Story 1 - Cash Sale at the Counter (Priority: P1) 🎯 MVP

**Goal**: A cashier scans or searches products, adjusts quantities, takes cash payment with change, and receives a receipt — with stock decremented automatically and the whole flow working offline.

**Independent Test**: Scan two products, complete a cash sale with change due, verify receipt content, and confirm stock dropped by the sold quantities — with the network disabled.

### Tests for User Story 1 ⚠️ WRITE FIRST — MUST FAIL BEFORE IMPLEMENTATION

- [ ] T052 [P] [US1] Write failing cart state tests in `frontend/tests/unit/cart.test.ts` covering add-by-scan, quantity increment on duplicate scan, manual quantity edit, line removal, and real-time subtotal recomputation
- [ ] T053 [P] [US1] Write failing catalog search tests in `frontend/tests/unit/catalog-search.test.ts` covering partial name/SKU/barcode match, unknown-barcode not-found result, and inactive-product exclusion
- [ ] T054 [P] [US1] Write failing sale completion tests in `frontend/tests/unit/sale-complete.test.ts` verifying atomic IndexedDB write of sale event + stock delta + cart clear, and receipt number assignment
- [ ] T055 [P] [US1] Write failing contract test for `POST /sync/push` SALE_COMPLETED events in `backend/tests/contract/sync-push-sale.test.ts` per contracts/sync-api.md
- [ ] T056 [P] [US1] Write failing integration test for mid-transaction power loss in `frontend/tests/integration/power-loss.test.ts` — kill the store mid-write, reopen, verify no partial sale and cart survives
- [ ] T057 [P] [US1] Write failing E2E test for quickstart Scenario 1 (cash sale with change) in `frontend/tests/e2e/cash-sale.spec.ts` using FakePeripheralSet
- [ ] T058 [P] [US1] Write failing perf test in `frontend/tests/perf/scan-latency.spec.ts` asserting scan-to-line-rendered p95 < 100 ms and tender-to-receipt-issued p95 < 500 ms

### Implementation for User Story 1

- [ ] T059 [P] [US1] Implement catalog repository with local search index in `frontend/src/stores/catalog.ts` supporting sub-50 ms lookup over 10k SKUs (makes T053 pass)
- [ ] T060 [P] [US1] Implement cart store in `frontend/src/stores/cart.ts` using `shared/domain/pricing.ts` for all totals (makes T052 pass)
- [ ] T061 [US1] Implement sale completion service in `frontend/src/services/sale.ts` — atomic event append, stock delta, receipt number, idempotency key generation (makes T054, T056 pass)
- [ ] T062 [P] [US1] Implement checkout screen in `frontend/src/pages/Checkout.tsx` with search field wired to the scanner port, cart lines, and running totals
- [ ] T063 [P] [US1] Implement cart line component in `frontend/src/components/CartLine.tsx` with quantity edit and remove, 44 px targets
- [ ] T064 [US1] Implement cash tender dialog in `frontend/src/components/TenderCash.tsx` using `shared/domain/tender.ts` for change computation, keyboard-operable
- [ ] T065 [P] [US1] Implement receipt payload builder in `frontend/src/services/receipt.ts` producing the domain `ReceiptPayload` (store name, date/time, cashier, lines, discounts, tax, tenders, change, receipt number)
- [ ] T066 [P] [US1] Implement `EscPosWebUsbAdapter` and `BrowserPrintAdapter` in `frontend/src/services/peripherals/printer.ts` per contracts/peripheral-ports.md, with queue-on-failure semantics
- [ ] T067 [US1] Implement receipt queue and reprint in `frontend/src/services/receipt-queue.ts` — printer failure never blocks tender (Principle VI)
- [ ] T068 [US1] Implement `POST /sync/push` SALE_COMPLETED ingestion in `backend/src/api/sync.ts` with per-event transaction and idempotent replay (makes T055 pass)
- [ ] T069 [US1] Implement sale ingestion service in `backend/src/services/sale-ingest.ts` writing Sale, SaleLine, Tender, StockMovement, and AuditEntry atomically
- [ ] T070 [US1] Implement not-found and error messaging in `frontend/src/components/ErrorBanner.tsx` stating what happened, customer impact, and next action (Principle V)
- [ ] T071 [US1] Wire keyboard shortcuts for the full sale flow (scan → qty → tender → receipt) in `frontend/src/app/shortcuts.ts` and verify mouse-free completion

**Checkpoint**: User Story 1 fully functional — a complete offline cash sale with receipt and stock decrement. This is the MVP.

---

## Phase 4: User Story 2 - Card and Split Payment (Priority: P1)

**Goal**: Cashier accepts card payment or splits across cash and card, with each tender recorded separately and the card never over-charged.

**Independent Test**: Complete one full card sale and one split cash+card sale, verifying tender records and receipt accuracy for each.

### Tests for User Story 2 ⚠️ WRITE FIRST — MUST FAIL BEFORE IMPLEMENTATION

- [ ] T072 [P] [US2] Write failing split-tender tests in `frontend/tests/unit/split-tender.test.ts` covering remaining balance after first tender, completion when tenders sum to total, and over-tender applying change to cash only
- [ ] T073 [P] [US2] Write failing `CardTerminalPort` contract test in `frontend/tests/unit/card-terminal-port.test.ts` verifying approve, decline, timeout, and idempotent retry never double-charging
- [ ] T074 [P] [US2] Write failing E2E test in `frontend/tests/e2e/card-split-payment.spec.ts` covering card-only and split cash+card sales with FakeCardTerminal
- [ ] T075 [P] [US2] Write failing integration test for card terminal failure recovery in `frontend/tests/integration/card-failure.test.ts` — timeout leaves cart intact and no double charge on retry

### Implementation for User Story 2

- [ ] T076 [P] [US2] Implement `LocalHttpTerminalAdapter` in `frontend/src/services/peripherals/card-terminal.ts` per contracts/peripheral-ports.md, storing only token, last-four, and approval code (makes T073 pass)
- [ ] T077 [US2] Extend tender service in `frontend/src/services/tender.ts` to support multiple Tender records per sale with running balance (makes T072 pass)
- [ ] T078 [P] [US2] Implement card tender dialog in `frontend/src/components/TenderCard.tsx` with explicit degraded state for `TERMINAL_OFFLINE` (non-modal, never a bare spinner)
- [ ] T079 [P] [US2] Implement split-payment UI in `frontend/src/components/TenderSplit.tsx` showing remaining balance after each tender
- [ ] T080 [US2] Extend receipt builder in `frontend/src/services/receipt.ts` to itemize each tender method and amount separately (FR-010a)
- [ ] T081 [US2] Extend sale ingestion in `backend/src/services/sale-ingest.ts` to persist multiple tenders per sale with card token fields (never PAN/CVV)

**Checkpoint**: User Stories 1 AND 2 work independently — cash, card, and split payments all complete correctly.

---

## Phase 5: User Story 3 - Returns and Refunds (Priority: P1)

**Goal**: Cashier locates an original sale, selects returned items, processes the refund at the price actually paid, restores stock, and issues a refund receipt.

**Independent Test**: Complete a sale, refund one line item, verify the refund record, stock restoration, and refund receipt; attempt a duplicate return and confirm rejection.

### Tests for User Story 3 ⚠️ WRITE FIRST — MUST FAIL BEFORE IMPLEMENTATION

- [ ] T082 [P] [US3] Write failing sale-lookup tests in `backend/tests/unit/sale-lookup.test.ts` covering lookup by receipt number and by customer, returning lines, prices, discounts, and tenders
- [ ] T083 [P] [US3] Write failing refund contract test in `backend/tests/contract/sync-push-refund.test.ts` for REFUND_PROCESSED events, including `DUPLICATE_REFUND` error code
- [ ] T084 [P] [US3] Write failing over-return tests in `backend/tests/unit/refund-guard.test.ts` verifying `saleLine.returnedQuantity + quantity ≤ saleLine.quantity` enforced atomically under concurrent refunds
- [ ] T085 [P] [US3] Write failing E2E test for quickstart Scenario 3 in `frontend/tests/e2e/return-refund.spec.ts` including the duplicate-return rejection path
- [ ] T086 [P] [US3] Write failing integration test for duplicate-request replay in `backend/tests/integration/duplicate-replay.test.ts` — re-pushing the same refund event returns REPLAY with no second stock restore

### Implementation for User Story 3

- [ ] T087 [P] [US3] Implement `GET /sales/{receiptNumber}` and `GET /sales` filters in `backend/src/api/sales.ts` (makes T082 pass)
- [ ] T088 [P] [US3] Implement sale lookup UI in `frontend/src/pages/SaleLookup.tsx` searchable by receipt number
- [ ] T089 [US3] Implement refund draft service in `frontend/src/services/refund.ts` using `shared/domain/refund.ts` for pro-rata amounts at prices actually paid
- [ ] T090 [P] [US3] Implement return selection UI in `frontend/src/components/ReturnLines.tsx` showing returnable quantity per line
- [ ] T091 [US3] Implement refund ingestion in `backend/src/services/refund-ingest.ts` writing Refund, RefundLine, StockMovement (positive delta), Tender reversal, and AuditEntry atomically as a compensating entry — original sale never mutated (makes T083, T084 pass)
- [ ] T092 [US3] Implement return-window and threshold approval gate in `backend/src/services/refund-policy.ts` requiring manager authentication via `POST /auth/override` (FR-028)
- [ ] T093 [P] [US3] Implement refund receipt builder in `frontend/src/services/receipt.ts` linked to the original sale
- [ ] T094 [US3] Implement card refund path in `frontend/src/services/refund.ts` calling `CardTerminalPort.refund()` with idempotency key, defaulting to original tender method (FR-027)

**Checkpoint**: All P1 stories complete — the system can legally trade: sell, take any payment, and process returns.

---

## Phase 6: User Story 4 - Product and Category Management (Priority: P2)

**Goal**: Admin/manager maintains the product catalog and category tree — create, edit, deactivate products; organize nested categories.

**Independent Test**: Create a category, add a product with barcode and price, edit its price, deactivate it, and confirm it disappears from checkout search while historical sales remain intact.

### Tests for User Story 4 ⚠️ WRITE FIRST — MUST FAIL BEFORE IMPLEMENTATION

- [ ] T095 [P] [US4] Write failing product validation tests in `backend/tests/unit/product-validation.test.ts` covering required fields, duplicate SKU rejection, duplicate barcode rejection, and price-format validation
- [ ] T096 [P] [US4] Write failing category tree tests in `backend/tests/unit/category-tree.test.ts` covering nesting to depth 5, max-depth rejection, and deletion guard when products or children exist
- [ ] T097 [P] [US4] Write failing price-history tests in `backend/tests/unit/price-immutability.test.ts` verifying that editing a product price never alters completed sale lines
- [ ] T098 [P] [US4] Write failing integration test for concurrent catalog edits in `backend/tests/integration/concurrent-catalog.test.ts` — two terminals editing the same product resolves server-authoritative deterministically
- [ ] T099 [P] [US4] Write failing E2E test in `frontend/tests/e2e/product-management.spec.ts` covering create → edit price → deactivate → verify checkout exclusion

### Implementation for User Story 4

- [ ] T100 [P] [US4] Implement product CRUD endpoints in `backend/src/api/products.ts` with Zod validation and uniqueness enforcement (makes T095 pass)
- [ ] T101 [P] [US4] Implement category CRUD endpoints in `backend/src/api/categories.ts` with tree depth and deletion guards (makes T096 pass)
- [ ] T102 [P] [US4] Implement product service in `backend/src/services/product.ts` emitting catalog change events for `GET /sync/pull` (makes T098 pass)
- [ ] T103 [P] [US4] Implement product list page in `frontend/src/pages/Products.tsx` with filters (category, active status), search, and pagination for 10k SKUs
- [ ] T104 [P] [US4] Implement product form in `frontend/src/components/ProductForm.tsx` with field-level duplicate SKU/barcode errors
- [ ] T105 [P] [US4] Implement product detail view in `frontend/src/pages/ProductDetail.tsx` showing price, stock on hand, category, tax rate, and recent sales history
- [ ] T106 [P] [US4] Implement category tree manager in `frontend/src/components/CategoryTree.tsx` supporting nested create, rename, reorder, and reassignment
- [ ] T107 [US4] Implement `GET /sync/pull` catalog delta endpoint in `backend/src/api/sync.ts` returning products, categories, tax rules, discounts, and users since a sequence number
- [ ] T108 [US4] Implement sync pull consumer in `frontend/src/services/sync/pull.ts` replacing terminal catalog snapshot per entity (server-authoritative, research §2)

**Checkpoint**: Catalog is maintainable day-to-day; checkout reflects changes after sync.

---

## Phase 7: User Story 5 - Inventory Control (Priority: P2)

**Goal**: Staff record stock adjustments with reasons, view stock history, and see low-stock flags; oversell is guarded.

**Independent Test**: Record an adjustment of −2 with reason "damaged", verify the new on-hand count, the adjustment entry in history, and that a low-stock flag appears when a product drops below its threshold.

### Tests for User Story 5 ⚠️ WRITE FIRST — MUST FAIL BEFORE IMPLEMENTATION

- [ ] T109 [P] [US5] Write failing adjustment validation tests in `backend/tests/unit/adjustment-validation.test.ts` covering missing-reason rejection, negative-result rejection, and before/after snapshot correctness
- [ ] T110 [P] [US5] Write failing stock derivation tests in `backend/tests/unit/stock-derivation.test.ts` verifying on-hand equals the sum of all StockMovement deltas per product across sale, refund, adjustment, and initial sources
- [ ] T111 [P] [US5] Write failing low-stock tests in `backend/tests/unit/low-stock.test.ts` verifying the flag appears immediately when a sale brings stock to or below the reorder threshold
- [ ] T112 [P] [US5] Write failing oversell guard tests in `frontend/tests/unit/oversell.test.ts` covering insufficient-stock warning, manager-approved override with reason, and quantity capping per policy
- [ ] T113 [P] [US5] Write failing E2E test in `frontend/tests/e2e/inventory-adjustment.spec.ts` covering adjustment entry, history verification, and low-stock list appearance

### Implementation for User Story 5

- [ ] T114 [P] [US5] Implement stock adjustment endpoint in `backend/src/api/inventory.ts` writing StockMovement plus AuditEntry atomically (makes T109 pass)
- [ ] T115 [P] [US5] Implement stock derivation service in `backend/src/services/stock.ts` computing on-hand from the movement ledger with a rebuildable cache (makes T110, T111 pass)
- [ ] T116 [P] [US5] Implement inventory list page in `frontend/src/pages/Inventory.tsx` with a low-stock filter showing on-hand, threshold, and supplier
- [ ] T117 [P] [US5] Implement adjustment form in `frontend/src/components/AdjustmentForm.tsx` with a required reason selector (damage, theft, count correction, supplier delivery)
- [ ] T118 [P] [US5] Implement stock history view in `frontend/src/components/StockHistory.tsx` listing every change with source, actor, and timestamp
- [ ] T119 [US5] Implement oversell guard in `frontend/src/services/sale.ts` warning on insufficient stock and requiring manager override or capping quantity (makes T112 pass)
- [ ] T120 [US5] Implement negative-stock operational alert in `backend/src/services/stock.ts` flagging concurrent-sale oversells for manager review (research §8)

**Checkpoint**: Stock figures are trustworthy and every movement is traceable.

---

## Phase 8: User Story 6 - Discounts and Promotions (Priority: P2)

**Goal**: Manager defines discounts by scope and type; cashier applies them at checkout with correct stacking, tax recomputation, and approval gating.

**Independent Test**: Create a 10%-off category promotion, apply it at checkout, and verify line totals, cart total, and tax all reflect the discount, with the receipt itemizing it.

### Tests for User Story 6 ⚠️ WRITE FIRST — MUST FAIL BEFORE IMPLEMENTATION

- [ ] T121 [P] [US6] Write failing discount scope tests in `backend/tests/unit/discount-scope.test.ts` covering STORE, CATEGORY, and PRODUCT scoping and qualifying-item selection
- [ ] T122 [P] [US6] Write failing validity-window tests in `backend/tests/unit/discount-validity.test.ts` verifying rejection outside `validFrom`/`validTo` with the valid dates in the message
- [ ] T123 [P] [US6] Write failing approval-gate tests in `frontend/tests/unit/discount-approval.test.ts` verifying that a threshold discount prompts manager authentication before taking effect
- [ ] T124 [P] [US6] Write failing E2E test for quickstart Scenario 4 in `frontend/tests/e2e/discount-approval.spec.ts` including the audit entry assertion
- [ ] T125 [P] [US6] Write failing discounted-refund test in `frontend/tests/unit/discounted-refund.test.ts` verifying refunds use the discounted price paid, not shelf price

### Implementation for User Story 6

- [ ] T126 [P] [US6] Implement discount CRUD endpoints in `backend/src/api/discounts.ts` with scope and validity validation (makes T121, T122 pass)
- [ ] T127 [P] [US6] Implement discount management page in `frontend/src/pages/Discounts.tsx` for creating percentage, fixed-amount, and sale-price promotions
- [ ] T128 [US6] Implement discount application service in `frontend/src/services/discount.ts` using `shared/domain/pricing.ts` for line-before-cart stacking with visible order
- [ ] T129 [P] [US6] Implement discount picker UI in `frontend/src/components/DiscountPicker.tsx` showing qualifying items and computed savings
- [ ] T130 [US6] Implement manager override flow in `frontend/src/services/auth-override.ts` calling `POST /auth/override` and authenticating the manager, not the cashier (makes T123 pass)
- [ ] T131 [US6] Implement floor-at-zero warning in `frontend/src/services/discount.ts` surfacing over-application rather than silently discarding it (FR-020)
- [ ] T132 [US6] Extend receipt builder in `frontend/src/services/receipt.ts` to itemize each discount with type and amount (FR-023)

**Checkpoint**: All P2 stories complete — catalog, inventory, and promotions are fully operable.

---

## Phase 9: User Story 7 - Customer Directory and Purchase History (Priority: P3)

**Goal**: Staff create customer records, attach them to sales, view purchase history, and process receipt-less returns; customer data is erasable without destroying financial records.

**Independent Test**: Create a customer, attach them to a sale, then look them up and process a return against that sale without a receipt number.

### Tests for User Story 7 ⚠️ WRITE FIRST — MUST FAIL BEFORE IMPLEMENTATION

- [ ] T133 [P] [US7] Write failing customer search tests in `backend/tests/unit/customer-search.test.ts` covering search by name, phone, and email with email format validation
- [ ] T134 [P] [US7] Write failing erasure tests in `backend/tests/unit/customer-erasure.test.ts` verifying PII removal while sale records remain intact and anonymized (FR-040)
- [ ] T135 [P] [US7] Write failing merge tests in `backend/tests/unit/customer-merge.test.ts` verifying duplicate merge preserves all historical sale links
- [ ] T136 [P] [US7] Write failing E2E test in `frontend/tests/e2e/receiptless-return.spec.ts` covering customer lookup → past sale selection → refund without a receipt number

### Implementation for User Story 7

- [ ] T137 [P] [US7] Implement customer CRUD and search endpoints in `backend/src/api/customers.ts` (makes T133 pass)
- [ ] T138 [P] [US7] Implement customer erasure and merge service in `backend/src/services/customer.ts` with PII separable from the immutable ledger (makes T134, T135 pass)
- [ ] T139 [P] [US7] Implement customer management page in `frontend/src/pages/Customers.tsx` with search and edit
- [ ] T140 [P] [US7] Implement customer attach control in `frontend/src/components/CustomerAttach.tsx` for linking a customer at checkout
- [ ] T141 [US7] Implement purchase history view in `frontend/src/components/PurchaseHistory.tsx` with a "return this sale" action feeding the US3 refund flow (makes T136 pass)

**Checkpoint**: Receipt-less returns work; customer privacy obligations are satisfiable.

---

## Phase 10: User Story 8 - Sales History and Reports (Priority: P3)

**Goal**: Manager filters sales history and runs daily summary, sales-by-product, sales-by-category, and low-stock reports, all exportable and reconciling exactly.

**Independent Test**: Complete several sales, then generate a daily sales summary whose totals reconcile exactly with the sum of individual sales in the same period.

### Tests for User Story 8 ⚠️ WRITE FIRST — MUST FAIL BEFORE IMPLEMENTATION

- [ ] T142 [P] [US8] Write failing reconciliation tests in `backend/tests/integration/report-reconciliation.test.ts` asserting daily summary totals equal the sum of underlying sales to the minor unit
- [ ] T143 [P] [US8] Write failing ledger-derivation tests in `backend/tests/unit/report-derivation.test.ts` verifying every report is computable purely from the event log with caches dropped (Principle III)
- [ ] T144 [P] [US8] Write failing refund-inclusion tests in `backend/tests/unit/report-refunds.test.ts` verifying refunds appear as negative adjustments to net sales, never omitted (FR-046)
- [ ] T145 [P] [US8] Write failing export-parity tests in `backend/tests/unit/report-export.test.ts` verifying CSV values match on-screen report values exactly (FR-045)
- [ ] T146 [P] [US8] Write failing category rollup tests in `backend/tests/unit/category-rollup.test.ts` verifying subtotals roll up the category tree correctly

### Implementation for User Story 8

- [ ] T147 [P] [US8] Implement report SQL views in `backend/prisma/migrations/` deriving daily summary, by-product, by-category, and low-stock from the event ledger (makes T143 pass)
- [ ] T148 [P] [US8] Implement report endpoints in `backend/src/api/reports.ts` per contracts/sync-api.md, including the `lastSyncedAt` watermark (makes T142, T144, T146 pass)
- [ ] T149 [P] [US8] Implement CSV export in `backend/src/services/report-export.ts` reading the same SQL views as the screen, with formula-injection protection on exported cells (makes T145 pass)
- [ ] T150 [P] [US8] Implement sales history page in `frontend/src/pages/SalesHistory.tsx` with date-range, cashier, payment-method, and status filters
- [ ] T151 [P] [US8] Implement reports page in `frontend/src/pages/Reports.tsx` rendering all four reports with export buttons
- [ ] T152 [US8] Implement unsynced-data watermark banner in `frontend/src/components/SyncWatermark.tsx` showing "last synced" when offline sales are pending

**Checkpoint**: Business insight available and provably reconciling.

---

## Phase 11: User Story 9 - Admin Dashboard and User Management (Priority: P3)

**Goal**: Admin sees today's business at a glance and manages staff accounts with roles; all role restrictions are enforced server-side.

**Independent Test**: Create a cashier account, log in as that cashier, confirm they cannot access admin screens, then deactivate the account and confirm login is refused.

### Tests for User Story 9 ⚠️ WRITE FIRST — MUST FAIL BEFORE IMPLEMENTATION

- [ ] T153 [P] [US9] Write failing RBAC tests in `backend/tests/contract/rbac.test.ts` verifying deny-by-default on every endpoint and 403 for under-privileged roles (not merely hidden UI)
- [ ] T154 [P] [US9] Write failing auth tests in `backend/tests/unit/auth.test.ts` covering argon2id password and PIN hashing, rate limiting, and lockout after repeated failures
- [ ] T155 [P] [US9] Write failing deactivation tests in `backend/tests/unit/user-deactivation.test.ts` verifying login refusal with historical attribution preserved
- [ ] T156 [P] [US9] Write failing offline-auth tests in `frontend/tests/unit/offline-auth.test.ts` verifying cached-verifier PIN login while disconnected, with all events attributed
- [ ] T157 [P] [US9] Write failing dashboard aggregation tests in `backend/tests/unit/dashboard.test.ts` covering today's sales, transaction count, average basket, refunds, low-stock count, and top 5 products
- [ ] T158 [P] [US9] Write failing E2E test for quickstart Scenario 6 in `frontend/tests/e2e/role-enforcement.spec.ts` including direct-URL access denial
- [ ] T159 [P] [US9] Write failing audit coverage test in `backend/tests/integration/audit-coverage.test.ts` asserting every sensitive operation emits an AuditEntry with actor and before/after values (SC-012)

### Implementation for User Story 9

- [ ] T160 [P] [US9] Implement authentication endpoints in `backend/src/api/auth.ts` — `POST /auth/login`, `POST /auth/pin`, `POST /auth/override` with argon2id, rate limiting, and lockout (makes T154 pass)
- [ ] T161 [P] [US9] Implement RBAC middleware in `backend/src/api/rbac.ts` enforcing deny-by-default per endpoint role requirements (makes T153 pass)
- [ ] T162 [P] [US9] Implement user management endpoints in `backend/src/api/users.ts` for create, role change, and deactivate (makes T155 pass)
- [ ] T163 [P] [US9] Implement dashboard aggregation endpoint in `backend/src/api/dashboard.ts` derived from the event ledger (makes T157 pass)
- [ ] T164 [P] [US9] Implement session and offline auth cache in `frontend/src/app/session.ts` with a salted verifier for disconnected PIN login (makes T156 pass)
- [ ] T165 [P] [US9] Implement cashier PIN quick-switch screen in `frontend/src/pages/Login.tsx` completing operator change in under 3 seconds
- [ ] T166 [P] [US9] Implement user management page in `frontend/src/pages/Admin.tsx` for account creation, role assignment, and deactivation
- [ ] T167 [P] [US9] Implement admin dashboard in `frontend/src/pages/Dashboard.tsx` showing today's totals, average basket, refunds, low-stock count, and top products
- [ ] T168 [US9] Implement client-side route guards in `frontend/src/app/App.tsx` mirroring server RBAC — UI hiding supplements, never replaces, server enforcement

**Checkpoint**: All user stories independently functional; access control and oversight complete.

---

## Phase 12: Sync Engine Hardening (Cross-Cutting — Constitution Principle II)

**Purpose**: Prove the offline-first guarantees end-to-end. These tasks span all stories and must complete before release.

### Tests ⚠️ WRITE FIRST — MUST FAIL BEFORE IMPLEMENTATION

- [ ] T169 [P] Write failing offline→online sync integration test in `frontend/tests/integration/offline-sync.test.ts` covering full offline sale, queue persistence, reconnect, and server-side appearance
- [ ] T170 [P] Write failing 500-event burst test in `backend/tests/integration/sync-burst.test.ts` asserting 500 queued events drain in ≤ 10 batches under 60 seconds with zero duplicates (research §3)
- [ ] T171 [P] Write failing batch-resume test in `backend/tests/integration/sync-resume.test.ts` simulating mid-batch server failure and verifying resume without duplication
- [ ] T172 [P] Write failing backoff test in `frontend/tests/unit/sync-backoff.test.ts` verifying exponential backoff bounds and no thundering-herd on reconnect
- [ ] T173 [P] Write failing E2E test for quickstart Scenario 2 in `frontend/tests/e2e/offline-sale.spec.ts` including mid-sale process kill and recovery
- [ ] T174 [P] Write failing E2E test for quickstart Scenario 7 in `frontend/tests/e2e/printer-failure.spec.ts` verifying tender succeeds despite print failure and the receipt is reprintable

### Implementation

- [ ] T175 Implement sync push engine in `frontend/src/services/sync/push.ts` with 50-event batching, per-event idempotency keys, and exponential backoff (makes T169–T172 pass)
- [ ] T176 Implement sync scheduler in `frontend/src/services/sync/scheduler.ts` triggering on reconnect, on queue growth, and on a periodic timer
- [ ] T177 Implement `GET /health/terminal/{terminalId}` in `backend/src/api/health.ts` returning backlog depth, last synced timestamp, and clock skew (Principle VIII)
- [ ] T178 Implement sync backlog alerting thresholds in `backend/src/services/alerting.ts` for backlog age, failed-payment rate, and drawer variance

**Checkpoint**: Offline-first guarantees proven under adversarial conditions.

---

## Phase 13: Polish & Cross-Cutting Concerns

**Purpose**: Operational readiness, onboarding, and documentation

- [ ] T179 [P] Implement CSV catalog import parser in `frontend/src/services/import.ts` — streaming parse, row-level validation via shared Zod schemas, preview of valid/rejected rows with reasons, confirm-before-commit (FR-006a)
- [ ] T180 [P] Implement `POST /catalog/import` commit endpoint in `backend/src/api/catalog-import.ts` writing products plus `INITIAL` StockMovements atomically
- [ ] T181 [P] Implement import UI in `frontend/src/pages/Import.tsx` with the rejection report table
- [ ] T182 [P] Create demo seed script in `backend/prisma/seed.ts` with synthetic catalog, tax rules, discounts, and the three demo accounts — synthetic data only, never production data (Quality Gates)
- [ ] T183 [P] Write the operator runbook in `docs/runbook.md` covering unsynced-transaction flush, receipt reprint, orphaned-authorization reconciliation, shift reopen with audit, stuck-terminal cart handoff, and alerting thresholds (Principle VIII)
- [ ] T184 [P] Write the deployment guide in `docs/deployment.md` covering Docker Compose topology, environment variables, TLS termination, backup/restore, and the rollback procedure
- [ ] T185 [P] Implement audit chain verification job in `backend/src/services/audit-verify.ts` recomputing the hash chain on a schedule and alerting on any break
- [ ] T186 [P] Implement locale and currency formatting module in `frontend/src/app/i18n.ts` with all user-facing strings externalized and no concatenated sentence fragments (Principle VII)
- [ ] T187 [P] Add accessibility audit test in `frontend/tests/e2e/accessibility.spec.ts` asserting WCAG 2.1 AA contrast, 44 px touch targets, and keyboard reachability of every sale-flow step
- [ ] T188 Run the complete quickstart.md validation — all 10 scenarios plus latency budgets — and record results
- [ ] T189 Verify the full CI gate set passes: unit, integration, contract, money-invariant, offline-sync, hardware-fake, dependency scan, latency budgets, migration up/down
- [ ] T190 [P] Update `README.md` with project overview, setup instructions, and links to spec, plan, and runbook

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup — **BLOCKS all user stories**
  - Sub-order is strict: 2a (shared money/schemas/ports) → 2b (backend schema/ledger) and 2c (frontend shell/stores) in parallel → 2d (CI gates)
  - 2a must finish first: every money computation in every story imports from it
- **User Stories (Phases 3–11)**: All depend on Foundational completion
  - P1 stories (US1, US2, US3) should be completed first — they constitute a tradeable system
  - US2 and US3 build on US1's sale/tender/receipt foundation
  - P2 and P3 stories are independent of each other once Foundational is done
- **Sync Hardening (Phase 12)**: Depends on US1 (needs a real sale flow to sync); should complete before release
- **Polish (Phase 13)**: Depends on all desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: Depends only on Foundational — the MVP
- **US2 (P1)**: Depends on US1 (extends tender flow with card and split)
- **US3 (P1)**: Depends on US1 (refunds require completed sales to refund)
- **US4 (P2)**: Depends on Foundational only — independently testable
- **US5 (P2)**: Depends on Foundational; the oversell guard (T119) touches US1's sale service
- **US6 (P2)**: Depends on Foundational; integrates with US1 checkout and US3 refunds
- **US7 (P3)**: Depends on Foundational; receipt-less return (T141) feeds US3's refund flow
- **US8 (P3)**: Depends on Foundational; reports are richer once US1–US6 generate data
- **US9 (P3)**: Depends on Foundational; RBAC middleware (T161) should land early if multiple developers work in parallel

### Within Each User Story

- Tests MUST be written and observed to FAIL before implementation (Constitution Principle IV — NON-NEGOTIABLE)
- Shared domain logic before services; services before endpoints and UI
- Backend ingestion before frontend sync wiring
- Story complete and independently demonstrable before moving to the next priority

### Parallel Opportunities

- All Setup tasks marked [P] can run together (T003–T009)
- Within Foundational 2a, schema/port definition tasks (T022–T024) run in parallel; the money/tax/pricing/tender/refund test-then-implement pairs are sequential within each pair but the five pairs can be split across developers after T012 lands
- Foundational 2b and 2c run fully in parallel (different workspaces)
- All CI gate tasks (T046–T051) run in parallel
- Within each user story, all test tasks marked [P] can be written together, then all model/UI tasks marked [P] implemented together
- After Foundational, US4, US5, US6, US7, US8, and US9 can each be owned by a different developer

---

## Parallel Example: User Story 1

```bash
# Write all US1 tests together (they must all FAIL before implementation):
Task: "Cart state tests in frontend/tests/unit/cart.test.ts"
Task: "Catalog search tests in frontend/tests/unit/catalog-search.test.ts"
Task: "Sale completion tests in frontend/tests/unit/sale-complete.test.ts"
Task: "Sync push contract test in backend/tests/contract/sync-push-sale.test.ts"
Task: "Power loss integration test in frontend/tests/integration/power-loss.test.ts"
Task: "E2E cash sale in frontend/tests/e2e/cash-sale.spec.ts"
Task: "Perf test in frontend/tests/perf/scan-latency.spec.ts"

# Then implement the parallelizable pieces together:
Task: "Catalog repository in frontend/src/stores/catalog.ts"
Task: "Cart store in frontend/src/stores/cart.ts"
Task: "Checkout screen in frontend/src/pages/Checkout.tsx"
Task: "Cart line component in frontend/src/components/CartLine.tsx"
Task: "Receipt builder in frontend/src/services/receipt.ts"
Task: "Printer adapters in frontend/src/services/peripherals/printer.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational — **critical, blocks everything**; do not shortcut 2a
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: run quickstart Scenario 1 and the offline portion of Scenario 2; confirm money invariants and latency budgets pass
5. Demo: a cashier completing an offline cash sale with receipt and stock decrement

### Incremental Delivery

| Increment | Phases | Delivers |
|-----------|--------|----------|
| MVP | 1, 2, 3 | Offline cash sale with receipt and stock decrement |
| Tradeable | + 4, 5, 12 | Card/split payments, returns/refunds, proven sync — a store can legally open |
| Operable | + 6, 7, 8 | Catalog maintenance, inventory control, promotions |
| Complete | + 9, 10, 11, 13 | Customers, reports, dashboard/RBAC, onboarding, runbook |

### Non-Negotiable Gates

Per the constitution, these block release regardless of schedule pressure:

- Money-invariant suite passing (Principle I)
- Offline checkout proven with zero network calls (Principle II)
- Audit entries for every sensitive operation (Principle III)
- Tests written failing-first (Principle IV)
- Latency budgets met on target hardware (Principle V)
- Full checkout suite passing with zero physical hardware (Principle VI)
- Runbook complete with alerting thresholds documented (Principle VIII)

