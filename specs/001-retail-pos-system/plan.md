# Implementation Plan: Retail Point of Sale (POS) System

**Branch**: `001-retail-pos-system` | **Date**: 2026-08-28 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-retail-pos-system/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

A web-based POS for a single small retail store, running on desktop and tablet terminals (1–5 concurrent stations). The system provides fast checkout (scan/search → cart → discounts → tender → receipt), automatic inventory updates on sales and returns, stock adjustments with low-stock tracking, refunds, sales history, basic reports, and role-based access (admin/manager/cashier) with an admin dashboard.

Technical approach: a local-first architecture where each terminal runs a browser client with a durable local store (IndexedDB) holding the product catalog, pending sales, and stock deltas. A central server (Node.js + PostgreSQL) is the sync hub and reporting source of truth. All money is computed in integer minor units. Card payments are terminal-integrated (POS stores only outcome: token, last-four). Peripherals (printer, scanner, drawer, card terminal) sit behind replaceable interfaces with CI fakes. The checkout path completes with zero network calls; writes sync when connectivity returns.

## Technical Context

**Language/Version**: TypeScript 5.x (strict mode) for both client and server — single language across the stack reduces context-switching and enables sharing money/pricing domain types.

**Primary Dependencies**:
- Client: React 18, IndexedDB via `idb` (typed wrapper), Vite build, PWA service worker for install/offline shell
- Server: Fastify (HTTP API), Prisma (PostgreSQL access, typed), Zod (validation shared client/server)
- Money/pricing: hand-rolled integer minor-unit domain module (no decimal money library needed; cents as integers with explicit rounding rules)
- Testing: Vitest (unit + table-driven money/tax/discount suites), Playwright (E2E checkout flows with hardware fakes), contract tests via Fastify `inject` + shared Zod schemas

**Storage**:
- Terminal (client): IndexedDB — catalog snapshot, cart state, pending sale events queue, stock deltas, audit entries
- Server: PostgreSQL 16 — append-only event ledger (sales, refunds, voids, adjustments, audit), materialized current-state tables (products, stock, customers, users), sync sequence log

**Testing**: Vitest for domain logic (table-driven: pricing, discount stacking, tax inclusive/exclusive, tender/change, refund eligibility); Playwright for E2E; contract tests for sync API and peripheral interfaces; CI runs full checkout suite with zero physical hardware (fakes only).

**Target Platform**: Modern evergreen browsers (Chrome/Edge 120+, Safari 17+) on Windows desktop POS terminals and Android/iPadOS tablets; server on Linux (Docker).

**Project Type**: Web application (SPA client + REST API server), offline-first PWA.

**Performance Goals**: scan-to-line-rendered p95 < 100 ms (local); tender-to-receipt-issued p95 < 500 ms (local); sync of 500 queued events completes < 60 s; catalog search < 50 ms on 10k SKUs (local index).

**Constraints**: Offline-first checkout (zero network calls on sale path); integer minor-unit money everywhere; PCI scope reduction (no PAN/CVV in system); WCAG 2.1 AA; touch targets ≥ 44 px; keyboard-only sale completion.

**Scale/Scope**: 1 store, 1–5 terminals, ~10k SKUs, ~20 transactions/hour/terminal, 2���3 staff roles, single currency, single tax regime (configurable inclusive/exclusive mode).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principle | Status | Compliance approach |
|---|-----------|--------|---------------------|
| I | Money Is Sacred (NON-NEGOTIABLE) | ✅ PASS | Integer minor units + ISO 4217 in shared domain types; single rounding module (line → discount → tax → tender) with documented per-step rules; idempotency keys on sale/void/refund/tender; atomic transactions via Prisma `$transaction` server-side + atomic IndexedDB event append client-side; compensating entries only (no mutation of historical sales); money-invariant test suite in CI. |
| II | Offline-First Selling | ✅ PASS | Checkout path reads/writes local IndexedDB only; sale events queued durably; background sync with exponential backoff; deterministic conflict resolution (financial events append-only never conflict; catalog server-authoritative); degraded states surfaced non-modally; plan answers "500 queued events" (batched sync, < 60 s target). |
| III | Append-Only Auditability | ✅ PASS | Append-only event ledger in PostgreSQL (sales, refunds, voids, adjustments, audit entries) with hash-chained rows; reports derived from event log, not mutable state; sensitive ops (price override, threshold discount, void, refund, adjustment, user admin) each emit audit events with actor/terminal/UTC+local-offset/correlation ID. |
| IV | Test-First (NON-NEGOTIABLE) | ✅ PASS | Red-Green-Refactor mandated in tasks.md; table-driven suites for pricing/discount/tax/rounding/tender/refund; contract tests for sync API + peripheral ports; integration tests for offline→online sync, power-loss replay, duplicate-request replay, concurrent catalog edits; every money/tax/inventory defect ships with failing-first regression test. |
| V | Counter-Speed UX | ✅ PASS | Keyboard-only sale flow (scan → qty → discount → tender → receipt) with shortcut map; latency budgets enforced as CI performance tests; destructive actions confirmed + audited, routine actions unconfirmed; error messages state what happened / customer impact / next action; 44 px targets, WCAG 2.1 AA, no color-only status. |
| VI | Hardware Behind Ports | ✅ PASS | Domain-defined `PeripheralPort` interfaces (printer, scanner, drawer, card terminal, display) in shared types; adapters per vendor; deterministic fakes for CI — full checkout suite runs hardware-free; printer failure queues receipt, never blocks tender; peripheral state surfaced to operator. |
| VII | Tax, Currency & Locale Are Data | ✅ PASS | Tax rules as versioned data (rate, inclusive/exclusive mode, effective ranges) — never code branches; store-scoped settings; locale-externalized strings/formats; UTC persistence + store-local business-day boundaries; fiscal conformance tests per jurisdiction (v1: single regime, see research.md TODO resolution). |
| VII-TODO | FISCAL_JURISDICTIONS | ✅ RESOLVED | v1 targets a single configurable tax regime (rate + inclusive/exclusive mode as store settings). The tax-rule data model supports multiple jurisdictions via effective-dated rate tables; concrete jurisdiction enumeration deferred to first multi-jurisdiction deployment. Justified: single-store v1 scope. |
| VII-TODO | PAYMENT_PROVIDERS | ✅ RESOLVED | Card capture is terminal-integrated (clarified 2026-08-28): the terminal talks to the processor directly; POS records only outcome (approved amount, token, last-four). The `CardTerminalPort` interface abstracts the provider; v1 ships a fake + one real adapter behind it. PSP-specific reconciliation APIs are out of scope — the port contract defines the integration surface. |
| — | Security & Compliance section | ✅ PASS | No PAN/CVV storage (terminal-integrated capture); individual accounts (password admin/manager, PIN cashier) with server-side deny-by-default RBAC; secrets via environment injection; TLS 1.2+ everywhere; encrypted IndexedDB (WebCrypto) for local data; parameterized queries via Prisma; rate limiting on auth/refund endpoints; PII separable from ledger for erasure; pinned/scanned dependencies in CI. |
| — | Quality Gates & Workflow | ✅ PASS | This Constitution Check is the plan gate; DoD includes failing-first tests, offline behavior tested, audit events, money invariants, telemetry, runbook, rollback path; CI gates blocking (unit/integration/contract/money-invariant/offline-sync/hardware-fake/dependency-scan/latency/migration up-down); schema evolution expand→migrate→contract. |

**Gate result: PASS** — no violations requiring Complexity Tracking entries.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
backend/
├── src/
│   ├── domain/              # Money, pricing, tax, refund-eligibility (shared logic mirrored)
│   │   ├── money.ts         # Integer minor units, rounding rules
│   │   ├── pricing.ts       # Line/cart discount stacking, tax modes
│   │   └── refund.ts        # Refund eligibility & amount computation
│   ├── models/              # Prisma schema + generated types
│   ├── services/            # Sync ingestion, report builders, auth, audit
│   ├── api/                 # Fastify routes: sync, catalog, reports, auth
│   └── periphery/           # Server-side ports (sync only for v1)
└── tests/
    ├── unit/                # Domain table-driven suites
    ├── contract/            # Sync API contract tests
    └── integration/         # Sync replay, concurrent edits, report reconciliation

frontend/
├── src/
│   ├── domain/              # Shared money/pricing/refund modules (single source, imported)
│   ├── stores/              # IndexedDB repositories (catalog, queue, cart, audit)
│   ├── services/
│   │   ├── sync/            # Background sync engine, backoff, batch push
│   │   └── peripherals/     # PeripheralPort implementations + fakes
│   ├── components/          # Checkout UI, cart, tender, receipts, admin screens
│   ├── pages/               # Checkout, Products, Inventory, Customers, Reports, Admin
│   └── app/                 # Routing, auth/session, PWA shell
└── tests/
    ├── unit/                # Store logic, sync engine unit tests
    ├── e2e/                 # Playwright checkout flows (hardware fakes)
    └── perf/                # Latency budget checks (scan-to-render, tender-to-receipt)

shared/
├── types/                   # Zod schemas + TS types shared client/server
└── ports/                   # PeripheralPort, SyncPort interface definitions
```

**Structure Decision**: Web application structure — `backend/` (Fastify + Prisma + PostgreSQL) and `frontend/` (React SPA + IndexedDB local-first store) with a `shared/` package for domain types, Zod schemas, and port interfaces. The shared package is the architectural keystone: money and pricing logic exist exactly once and are imported by both sides, guaranteeing identical computation on terminal and server. This satisfies Constitution Principles I (single money implementation), II (offline-first with local domain logic), and VI (ports defined in shared, fakes in frontend services).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No violations — table intentionally empty.
