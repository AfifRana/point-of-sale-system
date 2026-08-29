# Contracts: Retail Point of Sale (POS) System

**Feature**: 001-retail-pos-system | **Date**: 2026-08-28

This feature exposes two contract surfaces:

1. **Sync API** — the REST interface between terminals and the server (`contracts/sync-api.md`)
2. **Peripheral Ports** — the domain interfaces between the application and hardware (`contracts/peripheral-ports.md`)

All request/response bodies are validated by Zod schemas in `shared/types/` — the schemas
are the single source of truth for these documents. Contract tests in
`backend/tests/contract/` and `frontend/tests/unit/` verify both sides against the same
schemas.

---

## Contents

- [sync-api.md](./sync-api.md) — Terminal ↔ Server REST contract: authentication, event
  push (batched, idempotent), catalog pull, report queries, health
- [peripheral-ports.md](./peripheral-ports.md) — `ReceiptPrinterPort`,
  `BarcodeScannerPort`, `CashDrawerPort`, `CardTerminalPort`, `CustomerDisplayPort`
  interfaces and their deterministic fakes
