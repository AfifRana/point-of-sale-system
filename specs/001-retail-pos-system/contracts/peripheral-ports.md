# Peripheral Port Contracts

**Feature**: 001-retail-pos-system | **Version**: 1.0

Domain-defined interfaces between the application and POS hardware (Constitution
Principle VI). No vendor SDK type appears in domain or application layers. Every port
ships with a deterministic fake used by CI — the full checkout suite runs with zero
physical hardware.

Interfaces live in `shared/ports/`; adapters live in `frontend/src/services/peripherals/`.

**Common rules for all ports**:
- Every operation is async and MUST NOT block the sale path on failure — hardware errors
  return typed results, never throw into the checkout flow.
- Peripheral state is observable: `status()` returns `CONNECTED | DEGRADED | OFFLINE`
  plus a human-readable detail, surfaced on the terminal health panel.
- All operations accept an `idempotencyKey` where a retry could double-fire hardware
  effects (drawer kick, card charge).

---

## ReceiptPrinterPort

```ts
interface ReceiptPrinterPort {
  status(): Promise<PeripheralStatus>;
  print(receipt: ReceiptPayload, idempotencyKey: string): Promise<PrintResult>;
  queueDepth(): Promise<number>;   // queued receipts awaiting retry
}

type PrintResult =
  | { ok: true; printedAt: string }
  | { ok: false; queued: true; reason: "OFFLINE" | "OUT_OF_PAPER" | "UNKNOWN" }
  | { ok: false; queued: false; reason: "REJECTED"; detail: string };
```

**Contract**: printer failure MUST NOT block tender (Principle VI) — a failed print
returns `{ ok: false, queued: true }` and the receipt is stored in the IndexedDB
`receipts` store for reprint. `ReceiptPayload` is a domain type (store name, lines,
totals, tenders, receipt number) — never ESC/POS bytes; adapters translate.

**Adapters (v1)**: `EscPosWebUsbAdapter` (WebUSB), `BrowserPrintAdapter` (fallback).
**Fake**: `FakePrinter` — records payloads, scriptable failure injection.

---

## BarcodeScannerPort

```ts
interface BarcodeScannerPort {
  status(): Promise<PeripheralStatus>;
  onScan(handler: (barcode: string) => void): () => void;  // returns unsubscribe
}
```

**Contract**: keyboard-wedge scanners (the default) emit keystrokes ending in Enter —
the `KeyboardWedgeAdapter` treats the focused search field as the scanner input and
needs no device API. Scanned barcodes are untrusted input (Security section): they are
treated as opaque strings, length-capped, and never interpolated into queries.

**Adapters (v1)**: `KeyboardWedgeAdapter` (default, zero-hardware), optional
`HidAdapter`. **Fake**: `FakeScanner` — programmatic `scan(barcode)` for E2E tests.

---

## CashDrawerPort

```ts
interface CashDrawerPort {
  status(): Promise<PeripheralStatus>;
  open(reason: "SALE_TENDER" | "NO_SALE" | "REFUND", idempotencyKey: string): Promise<DrawerResult>;
}
```

**Contract**: `NO_SALE` opens require an authenticated actor and emit an audit event
(`NO_SALE_DRAWER`, data-model AuditEntry). Drawer kick is typically wired through the
printer; the adapter composes with `ReceiptPrinterPort` — the port abstraction keeps
that an implementation detail.

**Adapters (v1)**: `PrinterKickAdapter`. **Fake**: `FakeDrawer` — records opens.

---

## CardTerminalPort

```ts
interface CardTerminalPort {
  status(): Promise<PeripheralStatus>;
  charge(request: ChargeRequest): Promise<ChargeResult>;
  refund(request: TerminalRefundRequest): Promise<TerminalRefundResult>;
}

interface ChargeRequest {
  amountMinor: number;
  currency: string;
  idempotencyKey: string;   // retried charge returns original outcome — never double-charges
}

type ChargeResult =
  | { ok: true; token: string; lastFour: string; approvalCode: string }
  | { ok: false; reason: "DECLINED" | "TERMINAL_OFFLINE" | "TIMEOUT" | "UNKNOWN"; detail?: string };
```

**Contract**: the terminal talks to the payment processor directly; the POS records only
the outcome (token, last four, approval code) — PAN/CVV never enter the system (Security
section, PCI SAQ scope reduction). A failed/timeout charge MUST leave the cart intact;
the cashier retries or switches tender method (US-2 scenario 5). `TERMINAL_OFFLINE` is
surfaced as an explicit degraded state, not a spinner (Principle II).

**Adapters (v1)**: `LocalHttpTerminalAdapter` (terminal's local API), `FakeCardTerminal`
— deterministic approve/decline/timeout scripting for CI and demos.

---

## CustomerDisplayPort

```ts
interface CustomerDisplayPort {
  status(): Promise<PeripheralStatus>;
  render(view: CartView | IdleView | TenderView): Promise<void>;
}
```

**Contract**: optional peripheral; absence is never an error (`status()` → OFFLINE, UI
omits the panel). `CartView` mirrors line items and running total.

**Adapters (v1)**: `SecondaryWindowAdapter` (opens a second browser window — works on
any dual-screen terminal without device APIs). **Fake**: `FakeDisplay`.

---

## FakePeripheralSet

`FakePeripheralSet` bundles all five fakes behind one constructor. It is the default in
CI, E2E tests, and the demo seed. Determinism rules: fakes expose `reset()` and a
scripted event queue (e.g., `printer.failNext("OUT_OF_PAPER")`); no timers, no random
behavior — every test run is reproducible.
