# Feature Specification: Retail Point of Sale (POS) System

**Feature Branch**: `001-retail-pos-system`

**Created**: 2026-08-28

**Status**: Draft

**Input**: User description: "Build a web-based Point of Sale (POS) system for a small retail business. The system should allow staff to manage products/items, categories, pricing, discounts, inventory, customers, and sales. Provide a fast and simple checkout experience where cashiers can search or scan products, add items to a cart, apply discounts, apply payments, and generate receipts. Inventory should automatically update when sales or returns occur, with support for stock adjustments and low-stock tracking. Include sales history, refunds/returns, basic sales and inventory reports, user roles such as admin, manager, and cashier, and an admin dashboard for managing the business. The system should be easy to use on desktop and tablet POS terminals, with clear workflows for daily retail operations. Focus on defining the user needs, core workflows, functional requirements, and acceptance criteria without specifying the technology stack"

## Clarifications

### Session 2026-08-28

- Q: Should product prices include sales tax in the shelf price, or should tax be added on top at checkout? → A: Configurable — a store setting selects tax-inclusive or tax-exclusive pricing mode.
- Q: How should the system capture card payments — through a bank-connected terminal or by typing card details into the POS? → A: Terminal-integrated — the card terminal communicates with the payment processor directly; the POS records only the outcome (approved amount, token, last four).
- Q: How many POS terminals should the system support operating at the same time in the store? → A: Both — the system must operate as a single terminal and scale to multiple (2–5) concurrent stations sharing live stock and sales data.
- Q: How should cashiers sign in at the start of their shift — personal username and password, or a quick personal PIN on a shared terminal? → A: Both — username + password for admin/manager sign-in; quick personal PIN for cashiers on shared terminals.
- Q: When the store first adopts the system, how should the existing product catalog and stock levels be loaded in — manual entry one at a time, or bulk import from a spreadsheet? → A: Both — manual entry for day-to-day additions, plus CSV bulk import for initial onboarding.

### Session 2026-08-29

- Q: At the start and end of each shift, should the system track the cash in the drawer — opening float, cash drops, and a closing count that flags shortfall or surplus? → A: Full shift management — open shift with declared float, record cash drops and payouts, close shift with counted cash, system reports variance and produces a shift/Z-report.
- Q: When a cashier steps away mid-sale or a terminal breaks with items already scanned, should the system let them park the cart and resume it later, possibly on another terminal? → A: Park and resume across terminals — a cashier suspends a numbered cart and any terminal can retrieve and complete it.
- Q: When a customer buys something priced by weight or length, should the system accept fractional quantities such as 1.75 kg? → A: Whole units only — every product sells in integer quantities; weighed and measured goods are out of scope for v1.
- Q: How long should the system keep customer contact details and sales records before deleting or archiving them? → A: Configurable — retention periods are store settings with sensible defaults (financial records 7 years, customer PII anonymized after 2 years of inactivity).
- Q: Should a manager be able to override an item's price at the counter, separately from applying a defined discount? → A: Manager-authenticated price override — a manager sets a line's price with a required reason, fully audited with before/after values.

## User Scenarios & Testing *(mandatory)*

<!--
  IMPORTANT: User stories should be PRIORITIZED as user journeys ordered by importance.
  Each user story/journey must be INDEPENDENTLY TESTABLE - meaning if you implement just ONE of them,
  you should still have a viable MVP (Minimum Viable Product) that delivers value.

  Assign priorities (P1, P2, P3, etc.) to each story, where P1 is the most critical.
  Think of each story as a standalone slice of functionality that can be:
  - Developed independently
  - Tested independently
  - Deployed independently
  - Demonstrated to users independently
-->

### User Story 1 - Cash Sale at the Counter (Priority: P1)

A cashier rings up a walk-in customer. The cashier scans a product's barcode (or searches for it by name/SKU), the item appears in the cart with its price, the cashier adjusts quantity if needed, takes a cash payment, gives change, and prints or emails the receipt. The sale is recorded, and stock for each sold item is reduced automatically.

**Why this priority**: Selling is the reason the system exists. Every other capability (inventory, reports, customers) either feeds into or derives from a completed sale. A store can operate on day one with just this story.

**Independent Test**: Can be fully tested by scanning two products, completing a cash sale with change due, verifying the receipt content, and confirming stock levels dropped by the sold quantities — delivering the core "take money, record it, decrement stock" value with zero other features.

**Acceptance Scenarios**:

1. **Given** the cashier is on the checkout screen, **When** they scan a known barcode, **Then** the product is added to the cart (or its quantity incremented if already present) with name, unit price, and line total visible within one second.
2. **Given** a product in the cart, **When** the cashier changes its quantity to 3, **Then** the line total and cart subtotal update immediately to reflect 3 × unit price.
3. **Given** a cart with a total of $25.00, **When** the customer tenders $30.00 cash, **Then** the system displays change due of $5.00 and allows the sale to complete only after confirming the amount received.
4. **Given** a completed sale, **When** the cashier views the receipt, **Then** it shows store name, date/time, cashier identity, line items with quantities and prices, discounts, tax, tender, change given, and a unique receipt number.
5. **Given** a completed sale of 2 units of Product A, **When** the cashier checks Product A's stock, **Then** its on-hand count has decreased by exactly 2.
6. **Given** the checkout screen, **When** the cashier searches by partial product name or SKU, **Then** matching products are shown with price and stock indicator, and selecting one adds it to the cart.
7. **Given** a scanned barcode that matches no product, **When** the scan completes, **Then** the system shows a clear "product not found" message with a next action (search or add product) and does not add anything to the cart.

---

### User Story 2 - Card and Split Payment (Priority: P1)

A customer pays by card, or splits payment across cash and card (e.g., $10 cash, remainder on card). The cashier selects the payment method(s), the system records each tender, computes change or remaining balance, and completes the sale with a receipt.

**Why this priority**: Card is the dominant tender in most retail settings; a POS that cannot take card cannot serve the majority of customers. Split tender is a frequent real-world case at this priority level.

**Independent Test**: Can be tested by completing one full card sale and one split cash+card sale, verifying tender records and receipt accuracy for each.

**Acceptance Scenarios**:

1. **Given** a cart with a total of $40.00, **When** the cashier selects card payment and confirms, **Then** the system records a card tender of $40.00 and completes the sale.
2. **Given** a cart with a total of $40.00, **When** the customer pays $10.00 cash then $30.00 on card, **Then** the system records both tenders, shows the remaining balance after the first tender, and completes the sale when tenders sum to the total.
3. **Given** a split payment in progress, **When** the tendered amounts exceed the total, **Then** the excess is applied as change to the cash portion only, and the card portion is never over-charged.
4. **Given** any completed sale, **When** the cashier opens its receipt, **Then** each tender method and amount is listed separately.
5. **Given** a card terminal that fails to respond, **When** the cashier retries or switches to another payment method, **Then** the sale is not double-charged and the original cart remains intact.

---

### User Story 3 - Returns and Refunds (Priority: P1)

A customer returns an item with a receipt. The cashier locates the original sale, selects the item(s) being returned, the system validates return eligibility, processes the refund to the original or another payment method, restores stock, and issues a refund receipt.

**Why this priority**: Returns are a legal obligation and a daily occurrence in retail. Without them, the system cannot be used for real trading.

**Independent Test**: Can be tested by completing a sale, then refunding one line item, verifying the refund record, stock restoration, and refund receipt.

**Acceptance Scenarios**:

1. **Given** a completed sale, **When** the cashier looks it up by receipt number or customer, **Then** the original line items, prices, discounts, and tenders are displayed.
2. **Given** an original sale displayed, **When** the cashier selects 1 unit of a line item for return, **Then** the system computes the refund amount using the price and discounts actually paid, not current shelf price.
3. **Given** a return of an item originally paid by card, **When** the refund is processed, **Then** the refund is recorded against the original tender method by default, with an option to refund to an alternate method (e.g., store credit or cash) where policy allows.
4. **Given** a completed refund, **When** the cashier checks stock, **Then** the returned item's on-hand count increases by the returned quantity.
5. **Given** an item already fully returned, **When** the cashier attempts to return it again, **Then** the system rejects the duplicate return with a clear message.
6. **Given** a refund attempt on a sale older than the store's return window, **When** the cashier submits it, **Then** the system requires manager approval before proceeding.
7. **Given** any refund, **When** it completes, **Then** a refund receipt is issued and the refund is linked to the original sale in sales history.

---

### User Story 4 - Product and Category Management (Priority: P2)

An admin or manager creates and maintains the product catalog: adding products with name, SKU, barcode, category, price, cost (optional), tax rate, and initial stock; editing them; deactivating products that are discontinued; and organizing products into a category tree.

**Why this priority**: The catalog is the foundation the cashier sells from. It is P2 rather than P1 only because a minimal seed catalog can be loaded before launch; day-to-day catalog maintenance is essential from week one.

**Independent Test**: Can be tested by creating a category, adding a product with barcode and price, editing its price, deactivating it, and confirming it no longer appears in checkout search while historical sales remain intact.

**Acceptance Scenarios**:

1. **Given** the admin is on the product management screen, **When** they create a product with name, SKU, barcode, category, price, and tax rate, **Then** the product is saved and immediately findable via checkout search and scan.
2. **Given** an existing product, **When** the admin edits its price, **Then** new sales use the new price while previously completed sales retain the price at time of sale.
3. **Given** an existing product, **When** the admin deactivates it, **Then** it no longer appears in checkout search results or scan matches, but its historical sales and stock records remain unchanged.
4. **Given** the category management screen, **When** the admin creates nested categories (e.g., Apparel > Men > Shirts), **Then** products can be assigned to any category level and browsed by that hierarchy.
5. **Given** a product form, **When** the admin submits it with a duplicate SKU or barcode, **Then** the system rejects the save with a field-level error identifying the conflict.
6. **Given** any product, **When** the admin views its detail page, **Then** they can see current price, stock on hand, category, tax rate, and recent sales history for that product.
7. **Given** the product list, **When** the admin filters by category, status (active/inactive), or searches by name/SKU/barcode, **Then** only matching products are shown with pagination for large catalogs.

---

### User Story 5 - Inventory Control (Priority: P2)

Staff perform a stock count, discover a discrepancy, and record a stock adjustment with a reason (damage, theft, count correction, supplier delivery). The system maintains stock levels, tracks adjustments, and flags products whose stock falls below their reorder threshold.

**Why this priority**: Stock accuracy is what makes the checkout's stock counts and the reports trustworthy. Adjustments are the mechanism that keeps them honest between deliveries.

**Independent Test**: Can be tested by recording a stock adjustment of −2 with reason "damaged", verifying the new on-hand count, the adjustment entry in history, and that a low-stock flag appears when a product drops below its threshold.

**Acceptance Scenarios**:

1. **Given** a product with 10 units on hand, **When** staff record an adjustment of −2 with reason "damaged", **Then** on-hand becomes 8 and the adjustment is recorded with who, when, reason, and before/after values.
2. **Given** a product with a reorder threshold of 5, **When** a sale brings its stock to 4, **Then** the product appears on the low-stock list immediately after the sale.
3. **Given** the inventory list, **When** the manager filters to low-stock items, **Then** all products at or below their threshold are shown with current stock, threshold, and supplier (if set).
4. **Given** any product, **When** the manager views its stock history, **Then** every change is listed with its source: sale, return, adjustment, or initial load.
5. **Given** a product with 3 units on hand, **When** a cashier attempts to sell 5 units, **Then** the system warns that stock is insufficient and requires explicit override (with reason and manager approval) to proceed, or the quantity is capped.
6. **Given** the adjustment form, **When** staff submit an adjustment without a reason, **Then** the system rejects it with a clear validation message.
7. **Given** the adjustment form, **When** staff submit an adjustment that would make on-hand negative, **Then** the system rejects it with a clear validation message.

---

### User Story 6 - Discounts and Promotions (Priority: P2)

A manager defines discounts — percentage off, fixed amount off, or sale price — applicable store-wide, to a category, or to specific products. At checkout, the cashier applies a discount to the cart or a line, and the system computes the discounted totals and tax correctly.

**Why this priority**: Discounts drive retail traffic and margin management. It is P2 because a store can trade without promotions, but not for long competitively.

**Independent Test**: Can be tested by creating a 10%-off category promotion, applying it at checkout, and verifying line totals, cart total, and tax all reflect the discount.

**Acceptance Scenarios**:

1. **Given** the discount management screen, **When** a manager creates a 10% discount on the "Apparel" category, **Then** it is available to apply at checkout on qualifying items.
2. **Given** a cart containing an apparel item, **When** the cashier applies the category discount, **Then** the item's line total reflects 10% off and the cart total and tax recompute accordingly.
3. **Given** a cart with a line-item discount and a cart-level discount, **When** both are applied, **Then** the system computes the line discount first, then the cart discount on the post-line-discount subtotal, and displays the stacking order.
3. **Given** a discount that would reduce a line or cart below zero, **When** it is applied, **Then** the total is floored at zero and the over-application is not silently lost but shown as a warning.
4. **Given** a discount requiring manager approval (per policy threshold), **When** a cashier applies it, **Then** the system prompts for manager authentication before the discount takes effect.
5. **Given** a discount with a validity window, **When** it is applied outside that window, **Then** the system rejects it with a message stating the valid dates.
6. **Given** any discounted sale, **When** the receipt is printed, **Then** each discount is itemized with its type and amount so the customer sees exactly what was saved.
7. **Given** a discounted sale is later refunded, **When** the refund is computed, **Then** the refund uses the discounted price actually paid, not the shelf price.

---

### User Story 7 - Customer Directory and Purchase History (Priority: P3)

Staff create a customer record (name, phone, email), attach it to a sale, and later look up that customer to see their purchase history and process returns without a paper receipt.

**Why this priority**: Customer records enable receipt-less returns and loyalty basics, but a store can operate entirely on anonymous cash sales. This is the first layer of customer relationship value.

**Independent Test**: Can be tested by creating a customer, attaching them to a sale, then looking them up and processing a return against that sale without a receipt number.

**Acceptance Scenarios**:

1. **Given** the customer management screen, **When** staff create a customer with name and phone, **Then** the customer is saved and searchable by name, phone, or email.
2. **Given** a customer record, **When** staff attach it to a sale at checkout, **Then** the sale is linked to the customer and appears in their purchase history.
3. **Given** a customer with prior purchases, **When** staff look them up, **Then** their full purchase history is displayed with dates, items, and totals.
4. **Given** a customer's purchase history, **When** staff select a past sale to process a return, **Then** the return flow works identically to a receipt-based return.
5. **Given** a customer record, **When** staff edit or merge duplicate records, **Then** the changes are saved and historical sales links are preserved.
6. **Given** a customer requests deletion of their data, **When** staff process the erasure, **Then** personal details are removed while the financial records of past sales remain intact and anonymized.

---

### User Story 8 - Sales History and Reports (Priority: P3)

A manager reviews sales history with filters (date range, cashier, payment method, status) and runs basic reports: daily sales summary, sales by product, sales by category, and inventory valuation/low-stock report. Reports can be exported.

**Why this priority**: Reports turn transaction data into business insight, but they consume data the system already produces. Valuable, yet not required to trade.

**Independent Test**: Can be tested by completing several sales, then generating a daily sales summary whose totals reconcile exactly with the sum of individual sales in the same period.

**Acceptance Scenarios**:

1. **Given** sales history, **When** the manager filters by date range and cashier, **Then** only sales matching all filters are listed with receipt number, time, cashier, items count, and total.
2. **Given** a set of sales for a day, **When** the manager runs the daily sales summary, **Then** gross sales, discounts, net sales, tax, refunds, and transaction count are shown and reconcile to the sum of underlying sales.
3. **Given** the sales-by-product report, **When** the manager runs it for a period, **Then** each product shows units sold, gross revenue, and refunds, ranked by revenue.
4. **Given** the sales-by-category report, **When** the manager runs it for a period, **Then** each category shows units and revenue, with subtotals rolling up the category tree.
5. **Given** the low-stock report, **When** the manager runs it, **Then** products at or below reorder threshold are listed with on-hand, threshold, and suggested reorder quantity.
6. **Given** any report, **When** the manager exports it, **Then** a file is produced (e.g., CSV) whose numbers match the on-screen report exactly.
7. **Given** a refund, **When** it is included in a period report, **Then** it is shown as a negative adjustment to that day's net sales, not omitted.

---

### User Story 9 - Admin Dashboard and User Management (Priority: P3)

An admin sees a dashboard with today's sales, transaction count, average basket, refunds, low-stock alerts, and top products; and manages user accounts — creating staff accounts with roles (admin, manager, cashier), resetting passwords, and deactivating leavers.

**Why this priority**: Oversight and access control matter from day one but can start minimal (a single admin). The dashboard is a window onto data already captured.

**Independent Test**: Can be tested by creating a cashier account, logging in as that cashier, confirming they cannot access admin screens, then deactivating the account and confirming login is refused.

**Acceptance Scenarios**:

1. **Given** the admin dashboard, **When** the admin opens it, **Then** today's total sales, transaction count, average basket value, refund total, and count of low-stock items are displayed.
2. **Given** the user management screen, **When** the admin creates a cashier account, **Then** the cashier can log in and access checkout but receives "access denied" on admin and manager screens.
3. **Given** a cashier account, **When** the admin changes the role to manager, **Then** on next login the user gains manager capabilities (discounts, adjustments, reports).
4. **Given** a staff member leaves, **When** the admin deactivates their account, **Then** login is refused but their historical sales and audit records remain attributed to them.
5. **Given** the dashboard, **When** the admin views top products, **Then** the top 5 products by revenue today are listed with units sold.
6. **Given** any screen, **When** a user attempts an action outside their role, **Then** the action is blocked server-side with a clear message, not merely hidden in the UI.

### Edge Cases

- What happens when a barcode is scanned that matches no product? (Covered in US-1 scenario 7: clear not-found message, no cart change.)
- What happens when a sale is attempted on an out-of-stock or insufficient-stock item? (Covered in US-5 scenario 4: warn, require override with manager approval or cap quantity.)
- What happens when a discount would make a total negative? (Covered in US-6 scenario 3: floor at zero, surface the over-application.)
- What happens when tenders exceed the sale total with no cash component? (Covered in US-2 scenario 3: excess applies to cash only; card never over-charged.)
- What happens when a refund is attempted twice for the same item? (Covered in US-3 scenario 5: rejected as duplicate.)
- What happens when the network or power fails mid-sale? (Constitution Principle II: checkout must complete offline; sale is captured locally and synced later. Acceptance criteria for offline behavior are deferred to the implementation plan per Constitution Principle II.)
- What happens when two terminals sell the last unit of a product simultaneously? (Constitution Principle II conflict resolution: financial transactions are append-only and never conflict; stock is reconciled deterministically at sync. Detailed criteria deferred to plan.)
- What happens when a receipt printer fails mid-sale? (Constitution Principle VI: printer failure must not block tender; receipt is queued and reprintable.)
- What happens when a customer's refund exceeds the original payment amount? (Refund is capped at the amount actually paid for the returned items.)
- What happens when a product's price is changed mid-shift? (Covered in US-4 scenario 2: new sales use the new price; historical sales are immutable.)
- What happens when a user with a shared account leaves? (Constitution Security section: shared accounts are prohibited; individual accounts with role-based access are required.)
- What happens when a report is generated for a period containing offline sales not yet synced? (Report shows synced data with a clear "last synced" indicator; totals are recomputed when sync completes.)
- What happens when a cashier tries to sell before opening a shift? (Covered in FR-052g: cash sale blocked with a prompt to open a shift.)
- What happens when the counted cash at shift close does not match the expected amount? (Covered in FR-052c: variance computed and displayed; the shift still closes, and the variance is auditable and alertable per Constitution Principle VIII.)
- What happens when a shift is left open at the end of the day, or a terminal fails mid-shift? (Manager may close or reopen the shift with authentication per FR-052f; the original records remain immutable.)
- What happens when a terminal breaks with a customer's items already scanned? (Covered in FR-014a: the cart is parked and retrieved from another terminal — no rescanning.)
- What happens when two cashiers try to resume the same parked cart at once? (Covered in FR-014b: exclusive resume; the second terminal is told the cart is already in use.)

## Requirements *(mandatory)*

### Functional Requirements

**Catalog & Pricing**

- **FR-001**: System MUST allow authorized users to create, read, update, and deactivate products with name, SKU, barcode, category, unit price, cost (optional), tax rate, and reorder threshold.
- **FR-002**: System MUST enforce uniqueness of SKU and barcode among active products and reject duplicates with field-level errors.
- **FR-003**: System MUST support a nested category tree (at least 3 levels) with products assignable to any level and browsable by hierarchy.
- **FR-004**: System MUST retain the price at time of sale on completed sales; price edits MUST NOT alter historical sales.
- **FR-005**: System MUST allow deactivating a product such that it disappears from checkout search and scan but historical data remains intact.
- **FR-006**: System MUST support per-product tax rates and compute tax per line based on the product's rate at time of sale. Tax pricing mode MUST be a store-level setting selecting either tax-inclusive (shelf prices contain tax; receipt extracts the tax component) or tax-exclusive (tax added as a separate line at checkout); the mode in effect at sale time MUST be recorded on the sale.
- **FR-006a**: System MUST support bulk import of the product catalog and initial stock levels from a CSV file for store onboarding, validating each row (duplicate SKU/barcode, missing required fields, malformed prices) and reporting rejected rows with reasons before committing valid rows.

**Checkout**

- **FR-007**: System MUST allow cashiers to add products to a cart by barcode scan, by search (name/SKU/barcode), or by browsing the category tree.
- **FR-008**: System MUST increment quantity when a scanned product is already in the cart, and allow manual quantity edits and line removal. Quantities MUST be whole units (integers ≥ 1); fractional quantities are out of scope for v1.
- **FR-009**: System MUST display cart subtotal, discounts, tax, and total in real time as items are added or modified.
- **FR-010**: System MUST support cash tender with change computation, card tender, and split tender across cash and card in a single sale.
- **FR-010a**: System MUST record each tender method and amount separately on the sale record and receipt.
- **FR-011**: System MUST generate a unique receipt number per sale and produce a receipt containing store name, date/time, cashier identity, line items with quantities and prices, discounts itemized, tax, tenders, and change given.
- **FR-012**: System MUST decrement stock on-hand for each sold item at the moment the sale completes, atomically with the sale record.
- **FR-013**: System MUST warn when a sale quantity exceeds on-hand stock and require explicit override with manager approval, or cap the quantity, per store policy.
- **FR-014**: System MUST allow a sale to be voided before tender with no financial or stock effect, and record the void with actor and reason.
- **FR-014a**: System MUST allow a cashier to park (suspend) an in-progress cart, assigning it a retrievable reference, and MUST allow any terminal in the store to retrieve and complete a parked cart (Constitution Principle VIII: stuck-terminal cart handoff).
- **FR-014b**: System MUST prevent two terminals from resuming the same parked cart simultaneously, and MUST show parked carts with their reference, item count, total, parking cashier, and age.
- **FR-014c**: System MUST retain parked carts across terminal restart and MUST surface parked carts older than a configurable age for staff review; parking and resuming a cart MUST have no financial or stock effect until tender.
- **FR-015**: System MUST allow a sale to be voided after tender only as a refund flow, never by deleting the original sale.
- **FR-016**: System MUST support keyboard-only completion of the entire sale flow (scan, quantity, discount, tender, receipt) per Constitution Principle V.
- **FR-017**: System MUST complete the checkout path with zero network calls per Constitution Principle II (offline-first); network-dependent features MUST degrade, never block.

**Discounts**

- **FR-018**: System MUST support discount types: percentage off, fixed amount off, and sale price, applicable store-wide, per category, or per product.
- **FR-019**: System MUST compute line-level discounts before cart-level discounts and display the stacking order.
- **FR-020**: System MUST floor discounted totals at zero and surface over-application as a warning rather than silently discarding it.
- **FR-021**: System MUST support validity windows on discounts and reject application outside the window.
- **FR-022**: System MUST require manager authentication for discounts exceeding a configurable threshold, per policy.
- **FR-023**: System MUST itemize each discount with type and amount on the receipt.
- **FR-023a**: System MUST allow a manager to override the price of a cart line, requiring manager authentication and a reason, and MUST record an audit entry with the original and overridden price (Constitution Principle III).
- **FR-023b**: System MUST show an overridden line as price-overridden at checkout and on the receipt, distinct from a discount, and MUST use the overridden price for refund computation.

**Returns & Refunds**

- **FR-024**: System MUST allow locating an original sale by receipt number or by customer lookup.
- **FR-025**: System MUST compute refund amounts using the price and discounts actually paid, not current shelf price.
- **FR-026**: System MUST restore stock on-hand for returned items at the moment the refund completes, atomically with the refund record.
- **FR-026a**: System MUST prevent returning more units of a line item than were originally sold and not already returned.
- **FR-027**: System MUST record refunds against the original tender method by default, with alternate refund methods (cash, store credit) where policy allows.
- **FR-028**: System MUST require manager approval for refunds outside the return window or above a policy threshold.
- **FR-029**: System MUST issue a refund receipt linked to the original sale and record the refund as a distinct transaction type in sales history.
- **FR-030**: System MUST treat refunds as compensating entries; original sales MUST NOT be mutated or deleted.

**Inventory**

- **FR-031**: System MUST maintain per-product on-hand stock derived from sale, return, and adjustment events, with every change traceable to its source.
- **FR-032**: System MUST allow authorized users to record stock adjustments with reason (damage, theft, count correction, supplier delivery) and before/after values.
- **FR-033**: System MUST reject adjustments without a reason or that would result in negative on-hand.
- **FR-034**: System MUST flag products at or below their reorder threshold as low-stock immediately when a sale brings them to that level.
- **FR-035**: System MUST provide a low-stock list filterable in the inventory view and exportable as a report.
- **FR-036**: System MUST provide per-product stock history showing each change with source (sale, return, adjustment, initial load), actor, and timestamp.

**Customers**

- **FR-037**: System MUST allow creating, editing, and searching customer records by name, phone, or email.
- **FR-038**: System MUST allow attaching a customer to a sale at checkout, linking the sale to their purchase history.
- **FR-039**: System MUST support receipt-less returns by locating the original sale via customer purchase history.
- **FR-040**: System MUST support customer data erasure that removes personal details while preserving anonymized financial records.
- **FR-040a**: System MUST expose retention periods as store-configurable settings — one for financial/transaction records (default 7 years) and one for customer PII inactivity before automatic anonymization (default 2 years) — and MUST state the configured values in the admin settings screen.
- **FR-040b**: System MUST automatically anonymize customer PII once the configured inactivity period elapses, preserving the linked sales as anonymized financial records; automatic anonymization MUST be audited.
- **FR-040c**: System MUST support customer data export on request (portability), returning the customer's stored details and purchase history in a machine-readable file.

**Sales History & Reports**

- **FR-041**: System MUST provide sales history filterable by date range, cashier, payment method, and status (completed, refunded, voided).
- **FR-042**: System MUST provide a daily sales summary showing gross sales, discounts, net sales, tax, refunds, and transaction count that reconciles to underlying sales.
- **FR-043**: System MUST provide sales-by-product and sales-by-category reports for a chosen period, with category subtotals rolling up the tree.
- **FR-044**: System MUST provide an inventory report listing on-hand, reorder threshold, and suggested reorder quantity for low-stock products.
- **FR-045**: System MUST allow exporting reports to a file (e.g., CSV) whose values match the on-screen report exactly.
- **FR-046**: System MUST include refunds as negative adjustments in period reports rather than omitting them.

**Users, Roles & Dashboard**

- **FR-047**: System MUST support roles admin, manager, and cashier with permissions enforced server-side, deny-by-default, per Constitution Security section.
- **FR-048**: System MUST require individual accounts for every actor; shared accounts are prohibited. Sign-in MUST use username + password for admin and manager roles, and a quick personal PIN for cashiers on shared terminals; every sale and action MUST be attributable to the signed-in individual.
- **FR-049**: System MUST allow admins to create, deactivate, and change roles of user accounts; deactivated users cannot log in but historical attribution is preserved.
- **FR-050**: System MUST record an audit entry (actor, timestamp, action, before/after) for every sensitive operation: price override, discount above threshold, void, refund, stock adjustment, user administration, and permission change.
- **FR-051**: System MUST provide an admin dashboard showing today's sales total, transaction count, average basket, refund total, low-stock alert count, and top products by revenue.
- **FR-052**: System MUST authenticate manager overrides by identifying the overriding manager, not merely prompting the cashier.

**Shift & Cash Drawer Management**

- **FR-052a**: System MUST allow a cashier to open a shift on a terminal by declaring the opening cash float, recording actor, terminal, and timestamp.
- **FR-052b**: System MUST record cash drops (cash removed to a safe) and payouts during a shift, each with amount, reason, and actor.
- **FR-052c**: System MUST allow closing a shift by entering the counted cash on hand, and MUST compute and display the variance against the expected amount (`expected = opening float + cash sales − cash refunds − cash drops + payouts`).
- **FR-052d**: System MUST produce a shift report (Z-report) showing opening float, cash sales, card sales, refunds, drops, payouts, expected cash, counted cash, and variance — derived from the event log, not mutable state.
- **FR-052e**: System MUST record an audit entry for every shift open, shift close, cash drop, payout, and no-sale drawer open (Constitution Principle III).
- **FR-052f**: System MUST require manager authentication to reopen a closed shift, and MUST retain the original close record as an immutable entry alongside the reopen.
- **FR-052g**: System MUST prevent completing a cash sale on a terminal with no open shift, prompting the cashier to open one first.
- **FR-052h**: System MUST allow shift operations to complete offline, queuing them like all other events (Constitution Principle II).

**Cross-cutting (from Constitution)**

- **FR-053**: System MUST store and compute all monetary amounts as integer minor units with an ISO 4217 currency code; binary floating point MUST NOT be used for money.
- **FR-054**: System MUST make sale, void, refund, and tender operations idempotent, keyed by a client-generated idempotency key.
- **FR-055**: System MUST make each transaction atomic across line items, payments, and inventory effects; partial commits MUST be impossible.
- **FR-056**: System MUST maintain a tamper-evident append-only audit log per Constitution Principle III.
- **FR-057**: System MUST keep the checkout path operable offline per Constitution Principle II, with sync surviving process kill and device reboot.
- **FR-058**: System MUST meet counter-speed latency budgets: scan-to-line-rendered p95 < 100 ms; tender-to-receipt-issued p95 < 500 ms locally.
- **FR-059**: System MUST remain operable on desktop and tablet POS terminals with touch targets ≥ 44 px, WCAG 2.1 AA contrast, and no color-only status encoding.
- **FR-059a**: System MUST keep peripherals (receipt printer, barcode scanner, cash drawer, card terminal) behind replaceable interfaces per Constitution Principle VI, with hardware failure non-fatal to the sale.

### Key Entities *(include if feature involves data)*

- **Product**: A sellable item with name, SKU, barcode, category, unit price, cost, tax rate, reorder threshold, and active/inactive status. Relates to Category (many-to-one) and StockMovement (one-to-many).
- **Category**: A node in a nested tree organizing products; has name, parent, and effective tax defaults (optional).
- **StockMovement**: An append-only record of a stock change for a product: quantity delta, source (sale, return, adjustment, initial), reason, actor, timestamp, and before/after on-hand values.
- **Sale**: A completed financial transaction with unique receipt number, line items, discounts, tax, tenders, change, cashier, timestamp, and optional linked customer. Immutable once complete.
- **SaleLine**: A line on a sale: product, quantity, unit price at time of sale, discounts applied, and line total.
- **Tender**: A payment record on a sale: method (cash, card), amount, and reference (e.g., card last-four token). Card tenders are captured terminal-integrated — the terminal talks to the processor directly and the POS stores only the outcome (approved amount, token, last four).
- **Refund**: A compensating transaction linked to an original sale, containing returned lines, refund amount, method, approver, and reason.
- **Discount**: A promotion definition: type (percentage, fixed, sale price), scope (store, category, product), validity window, and approval threshold flag.
- **Customer**: A person record with name, phone, email, and linked purchase history; erasable without destroying financial records.
- **User**: A staff account with individual identity, role (admin, manager, cashier), and status; all actions attributed to a User. Admins and managers sign in with username + password; cashiers sign in with a quick personal PIN on shared terminals.
- **Shift**: A cashier's session of drawer accountability on a terminal: opening float, open/close timestamps, opening and closing actor, counted cash at close, computed expected cash, and variance. Immutable once closed; a reopen creates a linked follow-on record.
- **CashMovement**: An append-only record of cash entering or leaving the drawer outside of sales: type (drop, payout, no-sale open), amount, reason, actor, terminal, and timestamp, linked to its Shift.
- **ParkedCart**: A suspended in-progress cart: retrievable reference, line items with quantities and applied discounts, parking cashier, originating terminal, and parked-at timestamp. Has no financial or stock effect; resumable from any terminal, with exclusive resume locking.
- **AuditEntry**: A tamper-evident, append-only record of a sensitive operation: actor, terminal, timestamp (UTC + local offset), action, before/after values, and correlation ID.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A cashier can complete a standard cash sale (scan → tender → receipt) in under 30 seconds on a desktop or tablet terminal.
- **SC-002**: Scan-to-line-rendered latency stays within p95 < 100 ms, and tender-to-receipt-issued within p95 < 500 ms locally, on the lowest supported hardware.
- **SC-003**: 100% of completed sales produce a receipt with a unique receipt number and correctly itemized discounts, tax, tenders, and change.
- **SC-004**: Stock on-hand matches the sum of stock movements for every product at all times (verifiable by a reconciliation check).
- **SC-005**: Daily sales summary totals reconcile exactly (to the minor unit) with the sum of underlying sales for the same period.
- **SC-006**: Refund amounts always equal the discounted amount actually paid for the returned items, verifiable on any refunded sale.
- **SC-007**: 90% of new cashiers can complete their first unassisted sale after a single training session of 15 minutes or less.
- **SC-008**: The full checkout flow is operable via keyboard and scanner alone, with no mouse required, verified by usability testing.
- **SC-009**: The system sustains a simulated 8-hour trading day at a target of 20 transactions per hour per terminal without latency budget violations.
- **SC-010**: All role-restricted actions attempted by under-privileged users are blocked server-side with a clear denial message (100% of attempts in testing).
- **SC-011**: Checkout remains fully functional during a simulated network outage, with zero lost sales and complete post-outage sync.
- **SC-012**: 100% of sensitive operations (price override, threshold discount, void, refund, stock adjustment, user admin, shift open/close, cash drop) produce an audit entry with actor and before/after values.
- **SC-013**: Expected drawer cash always equals opening float + cash sales − cash refunds − cash drops + payouts, verifiable on every closed shift (drawer invariant, Constitution Principle I).

## Assumptions

- Single store, single currency for v1; multi-store and multi-currency are modeled in data but out of scope for v1 behavior.
- The system must operate both as a single terminal and with multiple concurrent terminals (2–5 stations) sharing live stock and sales data; concurrent-sale coordination (e.g., two terminals selling the last unit) is in scope per the Edge Cases section.
- Tax is computed per line from the product's tax rate; a single tax regime applies for v1 (jurisdiction list per Constitution TODO(FISCAL_JURISDICTIONS)). Tax pricing mode (inclusive vs. exclusive) is a store-level setting, configurable per clarification 2026-08-28.
- Card payments are terminal-integrated: a PCI-validated card terminal communicates directly with the payment processor, and the POS records only the outcome (approved amount, token, last four) — never card numbers (Constitution Security section). Manual card entry in the POS is out of scope for v1.
- Receipt printing uses a standard receipt printer behind a replaceable interface; a dead printer never blocks tender (Constitution Principle VI).
- Users have stable connectivity most of the time; offline capability exists for continuity, not as the primary mode.
- Product catalog size is small-retail scale (up to ~10,000 SKUs); enterprise-scale catalogs are out of scope.
- Return window, discount approval threshold, and stock-override policy are store-configurable settings with sensible defaults.
- Data retention is store-configurable: financial records default to 7 years (tax audit), customer PII to anonymization after 2 years of inactivity. Erasure never destroys financial records — PII is separable from the transaction ledger per Constitution Principle on privacy.
- Desktop and tablet terminals are the supported form factors; mobile phones are out of scope for v1.
- All products sell in whole units; weighed or measured goods (produce, deli, fabric) and scale peripherals are out of scope for v1. Per-product units of measure can be introduced later without invalidating existing sales data.
- Existing infrastructure provides user authentication primitives (or the system provides its own); no SSO integration is required for v1.
- Reports cover the listed set (daily summary, by product, by category, low-stock); advanced analytics (forecasting, basket analysis) are out of scope.
