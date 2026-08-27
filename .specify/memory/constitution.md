<!--
SYNC IMPACT REPORT
Version change: TEMPLATE (unversioned) → 1.0.0
Rationale: Initial ratification. All placeholder tokens replaced with concrete, testable
governance for a point-of-sale system. Version 1.0.0 is used because this is the first
binding version rather than an amendment to an existing one.

Modified principles (placeholder → concrete):
- [PRINCIPLE_1_NAME] → I. Money Is Sacred (NON-NEGOTIABLE)
- [PRINCIPLE_2_NAME] → II. Offline-First Selling
- [PRINCIPLE_3_NAME] → III. Append-Only Auditability
- [PRINCIPLE_4_NAME] → IV. Test-First (NON-NEGOTIABLE)
- [PRINCIPLE_5_NAME] → V. Counter-Speed UX

Added sections:
- VI. Hardware Behind Ports
- VII. Tax, Currency & Locale Are Data
- VIII. Observability & Operator Recoverability
- Security, Compliance & Data Protection (replaces [SECTION_2_NAME])
- Quality Gates & Development Workflow (replaces [SECTION_3_NAME])

Removed sections: none.

Deferred TODOs:
- TODO(FISCAL_JURISDICTIONS): Confirm the tax/fiscal jurisdictions in scope for v1 so
  Principle VII conformance rules can name concrete regimes.
- TODO(PAYMENT_PROVIDERS): Confirm the acquirer/PSP set so Principle I idempotency and
  the PCI clause can reference concrete reconciliation APIs.

Templates requiring no change: plan-template.md, spec-template.md, tasks-template.md,
checklist-template.md (all read this constitution at runtime).
-->

# Point of Sale System Constitution

## Core Principles

### I. Money Is Sacred (NON-NEGOTIABLE)

Monetary correctness outranks every other concern, including deadlines and UX polish.

- All monetary amounts MUST be stored and computed as integer minor units (e.g. cents)
  paired with an ISO 4217 currency code. Binary floating point (`float`, `double`,
  `REAL`) MUST NOT appear in any money type, column, or wire field.
- Rounding MUST be explicit, single-point, and documented per operation (line total,
  discount, tax, tender). Rounding MUST NOT be an emergent side effect of display
  formatting.
- Every sale, void, refund, and tender operation MUST be idempotent, keyed by a
  client-generated idempotency key. A retried request MUST return the original result,
  never a second charge.
- A transaction MUST be atomic across its line items, payments, and inventory effects.
  Partial commits MUST be impossible; a failed tender MUST leave zero financial residue.
- Corrections MUST be compensating entries (void / refund / adjustment). Historical
  financial records MUST NOT be mutated or deleted.
- Every release MUST pass a money-invariant suite proving: sum(lines) + tax − discounts
  = order total; sum(tenders) − change = order total; and drawer expected = opening
  float + cash in − cash out.

*Rationale*: A POS is a system of financial record. A one-cent drift or a double charge
is a legal, tax, and trust failure that no feature velocity can offset.

### II. Offline-First Selling

The store MUST keep selling when the network does not.

- The checkout path (scan, price, discount, tax, tender cash, print/queue receipt, open
  drawer) MUST complete with zero network calls. Network-dependent features MUST
  degrade, never block.
- Writes MUST be captured to a durable local store first, then synchronized. Sync MUST
  survive process kill and device reboot without data loss or duplication.
- Conflict resolution MUST be deterministic and documented per entity. Financial
  transactions are immutable and append-only, so they MUST never conflict; mutable
  catalog and configuration data resolves server-authoritative unless an explicit
  exception is recorded.
- Degraded capabilities (card payments, loyalty lookup, price sync, stock levels) MUST
  be surfaced to the cashier as explicit, non-modal state — never as silent failure or
  an unexplained spinner.
- Every feature plan MUST answer: "What happens offline, and what happens when 500
  queued events sync at once?" An unanswered question is a blocked plan.

*Rationale*: Lost connectivity is normal retail conditions. A POS that stops taking
money during an outage is worse than no POS at all.

### III. Append-Only Auditability

If it moved money, stock, or permission, it MUST be reconstructable after the fact.

- An append-only audit log MUST record actor identity, terminal and store, UTC timestamp
  plus captured local offset, action, before/after values, and correlation ID.
- Audit records MUST be tamper-evident (hash-chained or equivalent) and MUST NOT be
  editable or deletable by application code, including administrative code paths.
- Sensitive operations — price override, discount above threshold, void, refund, no-sale
  drawer open, cash drop, tax exemption, shift open/close, permission change — MUST
  require an identified actor and MUST be individually auditable.
- Any reported total (Z-report, shift report, tax report) MUST be derivable purely from
  the event log. Reports MUST NOT treat mutable denormalized state as the source of
  truth; caches are optimizations and MUST be rebuildable.

*Rationale*: Shrinkage investigations, tax audits, and chargeback disputes are answered
with evidence. Evidence must be immutable and complete.

### IV. Test-First (NON-NEGOTIABLE)

Tests define behavior before implementation exists.

- Red-Green-Refactor MUST be followed: failing test written and reviewed → implementation
  → refactor. Tests written after a merged implementation do not satisfy this principle.
- Pricing, discount stacking, tax calculation, rounding, tender/change, and refund
  eligibility MUST be covered by table-driven tests including boundary and adversarial
  cases (zero, negative, max quantity, mixed tax rates, multi-currency change).
- Contract tests MUST exist for every external boundary: payment terminal, PSP API, sync
  API, fiscal/tax service, hardware drivers, and any inter-service call.
- Integration tests MUST cover offline→online sync, mid-transaction power loss,
  duplicate-request replay, and concurrent edits to the same catalog entity.
- Every fixed defect in a money, tax, or inventory path MUST ship with a regression test
  that fails against the pre-fix code.

*Rationale*: Retail logic is combinatorial and its failures are financial. Tests are the
only affordable proof of correctness at this branching factor.

### V. Counter-Speed UX

The cashier's throughput is a hard functional requirement, not a design preference.

- The primary sale flow MUST be completable without a mouse or trackpad: barcode scan,
  keyboard, and hardware keys MUST reach every step.
- Input latency budgets: scan-to-line-rendered p95 < 100 ms; tender-to-receipt-issued
  p95 < 500 ms locally. Regressions beyond budget MUST block release.
- Destructive or financially significant actions (void, refund, discount) MUST require
  explicit confirmation and MUST be reversible or auditable; routine actions MUST NOT
  require confirmation.
- Error messages MUST state what happened, what it means for the customer in line, and
  the next action. Stack traces, bare error codes, and dead ends are defects.
- The UI MUST remain operable on the lowest supported target hardware and MUST be
  legible under glare and at arm's length (minimum touch target 44 px, no color-only
  status encoding, WCAG 2.1 AA contrast).

*Rationale*: A queue is forming behind every interaction. Ergonomics here converts
directly into revenue and staff retention.

### VI. Hardware Behind Ports

Peripherals MUST be replaceable without touching business logic.

- Every peripheral class (receipt printer, barcode scanner, cash drawer, card terminal,
  scale, customer display, fiscal device) MUST sit behind a domain-defined interface. No
  vendor SDK type may appear in domain or application layers.
- Every peripheral adapter MUST provide a deterministic fake or simulator used by CI.
  The full checkout suite MUST run with zero physical hardware attached.
- Hardware failure MUST be non-fatal to the sale. A dead printer MUST NOT block tender;
  the receipt MUST be queued, reprintable, or offered digitally.
- Peripheral state MUST be reported to the operator (connected, paper low, offline) and
  MUST be recoverable without restarting the application or losing the current cart.

*Rationale*: Store hardware is heterogeneous, ages independently, and fails mid-shift.
Coupling to it makes the software unshippable and untestable.

### VII. Tax, Currency & Locale Are Data

Jurisdictional and linguistic behavior MUST be configuration, never code branches.

- Tax rules (rate, base, inclusive/exclusive, exemptions, rounding, effective date
  ranges) MUST be data-driven and versioned. Rates MUST NOT be hardcoded, and a
  historical transaction MUST always reprice under the rules effective at its sale time.
- Multi-currency, multi-store, and multi-tenant boundaries MUST be modeled from the
  first release. Tenant and store scoping MUST be enforced at the data-access layer, not
  by caller discipline.
- All user-facing strings, number formats, date formats, and currency formats MUST be
  externalized and locale-resolved. Concatenated sentence fragments are prohibited.
- Timestamps MUST be persisted in UTC with the originating time zone retained; business
  day and shift boundaries MUST be computed in store-local time.
- Fiscal and receipt-content requirements for each in-scope jurisdiction MUST be encoded
  as automated conformance tests.
  TODO(FISCAL_JURISDICTIONS): enumerate the v1 jurisdiction list.

*Rationale*: Expansion into a new state, country, or tax holiday must be a configuration
change and a data migration — not a re-architecture.

### VIII. Observability & Operator Recoverability

A store manager MUST be able to diagnose and recover without engineering intervention.

- Logs MUST be structured, correlated by transaction, terminal, and store ID, and MUST
  NOT contain PAN, CVV, track data, full PII, or credentials.
- Health MUST be observable per terminal: sync backlog depth, last successful sync,
  peripheral status, pending offline transactions, and clock skew.
- The system MUST expose supported recovery paths for unsynced-transaction flush,
  receipt reprint, orphaned-authorization reconciliation, shift reopen with audit, and
  stuck-terminal handoff of an in-progress cart.
- Every release MUST be rollback-capable; schema changes MUST be backward compatible for
  at least one prior application version so terminals can update out of lockstep.
- Alerting thresholds MUST exist for sync backlog age, failed-payment rate, and drawer
  variance, and MUST be documented in the runbook before the feature ships.

*Rationale*: Terminals are deployed where no engineer is present. Recoverability is the
feature that keeps stores open.

## Security, Compliance & Data Protection

These constraints are binding on every feature, spike, and demo.

- **Cardholder data**: The system MUST NOT store, log, or transmit PAN, CVV/CVC,
  magnetic-stripe or chip track data, or PIN blocks. Card capture MUST be delegated to
  PCI-validated terminals or a tokenizing PSP; the application MUST handle tokens and
  last-four only, and MUST target PCI DSS SAQ scope reduction by design.
  TODO(PAYMENT_PROVIDERS): confirm the acquirer/PSP set.
- **Authentication & authorization**: Every actor MUST be individually identified — no
  shared accounts, no shared PINs. Authorization MUST be role-based, deny-by-default,
  and enforced server-side (and in the local domain layer for offline mode); hiding UI
  is not authorization. Manager-override flows MUST authenticate the overriding manager,
  not merely prompt the cashier.
- **Secrets & keys**: Credentials, API keys, and signing keys MUST NOT be committed or
  embedded in client builds. Secrets MUST be injected at runtime, scoped per
  environment, and rotatable without a code change.
- **Encryption**: All network traffic MUST use TLS 1.2+ with certificate validation
  enabled in every build configuration. Local databases on terminals MUST be encrypted
  at rest, because devices are physically accessible to the public.
- **Input & injection defense (OWASP Top 10)**: All external input — including barcodes,
  QR codes, and NFC payloads — MUST be treated as untrusted. Data access MUST use
  parameterized queries; output MUST be context-encoded; file and report exports MUST be
  protected against formula injection. Rate limiting and lockout MUST protect
  authentication and refund endpoints.
- **Privacy & data minimization**: Customer PII MUST be collected only with a stated
  purpose, MUST be retained per an explicit retention schedule, and MUST support export
  and erasure requests. Erasure MUST NOT destroy financial records; PII MUST be
  separable from the immutable transaction ledger so it can be redacted independently.
- **Dependencies**: Third-party dependencies MUST be pinned and scanned in CI. A build
  with a known critical vulnerability in a shipped path MUST NOT be released.
- **Segregation of duties**: The actor who configures a discount or price rule SHOULD NOT
  be the sole approver of its release; refund and cash-adjustment privileges MUST be
  grantable independently of general cashier access.

## Quality Gates & Development Workflow

- **Constitution gate**: Every `/speckit-plan` MUST include a Constitution Check that
  states, per principle, either compliance or an explicit justified deviation in the
  Complexity Tracking section. Unjustified deviation blocks the plan.
- **Definition of Done**: A change is done only when it has failing-first tests now
  passing, offline behavior specified and tested, audit events emitted for financial and
  permission effects, money invariants asserted, telemetry and health signals added,
  documentation and runbook updates, and a rollback path.
- **CI gates (all blocking)**: unit, integration, and contract suites; money-invariant
  suite; offline/sync scenario suite; hardware-fake checkout suite; dependency and secret
  scanning; latency budget checks on the sale path; migration up/down verification.
- **Review requirements**: Changes touching pricing, tax, tender, refund, audit, or auth
  MUST receive review from at least one reviewer outside the authoring pair, and the
  review MUST explicitly confirm Principles I and III and the Security section.
- **Schema & API evolution**: Breaking changes MUST follow expand → migrate → contract
  across at least one release. Terminals MUST be assumed to be running mixed versions
  simultaneously.
- **Simplicity (YAGNI)**: The simplest design satisfying the specified requirement MUST
  be chosen. New services, queues, caches, or abstraction layers MUST be justified in
  Complexity Tracking with the concrete failure mode they prevent. Speculative
  generality is a review rejection.
- **Data safety in non-production**: Production customer or cardholder data MUST NOT be
  copied into development, staging, or test environments. Fixtures MUST be synthetic.

## Governance

- **Supremacy**: This constitution supersedes all other practices, conventions, team
  habits, and prior guidance. Where a document conflicts with it, this document wins.
- **Amendment procedure**: Amendments MUST be proposed as a pull request modifying
  `.specify/memory/constitution.md`, MUST state the motivating problem and the affected
  principles, and MUST include a migration plan for any code or spec thereby out of
  compliance. Approval requires the project maintainers plus, for changes to Principle I,
  Principle III, or the Security section, sign-off from the finance/compliance owner.
- **Versioning policy**: Semantic versioning applies to this document.
  - MAJOR — a principle is removed, redefined incompatibly, or a NON-NEGOTIABLE is
    relaxed.
  - MINOR — a principle or section is added, or guidance is materially expanded.
  - PATCH — clarification, wording, typo, or non-semantic refinement.
- **Compliance review**: Compliance MUST be verified at three points — plan approval
  (Constitution Check gate), pull request review (checklist), and a quarterly audit of
  the prior quarter's merged work. Audit findings MUST be logged as remediation tasks
  with owners and dates.
- **Deviation handling**: Time-boxed deviations are permitted only when recorded in the
  feature's Complexity Tracking with an expiry date and a removal task. An expired
  deviation is a release blocker.
- **Runtime guidance**: Agent- and contributor-facing operational guidance lives in the
  repository agent guidance file and in `.specify/templates/`. Those files MUST be kept
  consistent with this constitution; on conflict, this constitution governs and the
  guidance file MUST be corrected.

**Version**: 1.0.0 | **Ratified**: 2026-08-26 | **Last Amended**: 2026-08-26
