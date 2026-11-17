## What would go wrong

### C1
- file: hld.md
- section: ## Message Schemas
- quote: "price_snapshot_minor_units": 1500
- match: order/models.js › const orderLineSchema
- severity: blocking
- blocks because: Dev would name the field `price_snapshot_minor_units` in the `OrderRequested` message payload but the order service expects `price_minor_units` in the database, and the HLD says the consumer should map `price_snapshot_minor_units` to `price_minor_units` on insert. The HLD message schema shows `price_snapshot_minor_units` but does not state the mapping explicitly in the schema section, only in story 2's LLD. Dev reading the HLD alone would not know to rename the field.
- problem: The HLD's `OrderRequested` message schema shows `price_snapshot_minor_units` matching the checkout outbox, but the order service's existing schema uses `price_minor_units`. The HLD does not state in the message schema section that the consumer must map this field on insert.
- change: In the `OrderRequested` message schema, add a field note: `price_snapshot_minor_units` (integer, required): Mapped to `price_minor_units` in the order document on insert.

### C2
- file: breakdown.md
- section: ## Story 2: Order service consumes OrderRequested and publishes OrderCreated
- quote: price_snapshot_minor_units: int
- match: hld.md › ## Message Schemas › OrderRequested
- severity: blocking
- blocks because: The breakdown lists `price_snapshot_minor_units: int` in the `OrderRequested` schema but does not state that the order service must map it to `price_minor_units` when inserting. Dev would see the schema, build the consumer to store the field as given, and the order document would have the wrong field name, breaking the existing `GET /orders/{order_id}` endpoint which returns `price_minor_units`.
- problem: The breakdown's story 2 schema lists `price_snapshot_minor_units` but does not state the field must be renamed to `price_minor_units` on insert.
- change: In story 2's schema section, add a field note below `price_snapshot_minor_units`: "Mapped to `price_minor_units` in the order document (see `repo-file/app/order/models.js`)."

### C3
- file: stories/02-app.md
- section: ## Algorithm Notes
- quote: price_snapshot_minor_units: line.price_snapshot_minor_units
- match: repo-file/app/order/models.js › price_minor_units
- severity: blocking
- blocks because: Dev following the LLD would map the message's `price_snapshot_minor_units` to the order document's `price_snapshot_minor_units`, but the existing order schema uses `price_minor_units`. The order document would have the wrong field name, and the HTTP API would break because it expects `price_minor_units`.
- problem: The LLD's data mapping algorithm maps `line.price_snapshot_minor_units` to `orderLine.price_snapshot_minor_units`, but the order schema in `repo-file/app/order/models.js` uses `price_minor_units`.
- change: In the data mapping code, change `price_snapshot_minor_units: line.price_snapshot_minor_units` to `price_minor_units: line.price_snapshot_minor_units  // rename`.

### C4
- file: hld.md
- section: ## Service Interaction Contracts
- quote: POST http://localhost:8005/charge
- match: repo-file/app/payment/routes.py › @router.post('/charge'
- severity: blocking
- blocks because: The HLD says the payment endpoint accepts an idempotency key (`payment_reference`) but the existing payment service does not. Dev would build the checkout to send a `payment_reference` field in the charge request, but the payment service would ignore it, and duplicate checkout submissions would charge twice. The HLD's idempotency design depends on the payment service accepting and honoring this key, which it does not in v1.
- problem: The HLD states "Payment generates a unique reference per charge. Checkout does not retry the charge; a repeated checkout submission reuses the same payment reference if the checkout already exists." This implies checkout sends a stable `payment_reference` to payment, but the existing `POST /charge` endpoint does not accept or use an idempotency key. Payment generates a fresh reference on every call.
- change: In the HLD's Service Interaction Contracts section, state: "Payment generates a unique reference per charge (no idempotency key accepted in v1). Checkout detects duplicates by querying its own database after payment returns the reference. Mark this as `[IDEMPOTENCY_DEFERRED]`." Update the idempotency algorithm in story 1's LLD to query the checkout database after payment accepts, not before.

### C5
- file: stories/01-app.md
- section: ## Algorithm Notes
- quote: payment_reference = body.cart_id
- match: hld.md › ## Algorithm Notes › Idempotency: Duplicate Checkout Submission
- severity: blocking
- blocks because: Story 1's LLD says `payment_reference` is derived from `cart_id`, but the HLD says `payment_reference` is returned by the payment service after accepting the charge. Dev would generate `payment_reference = str(uuid.uuid4())` before calling payment and send it as an idempotency key, but payment does not accept one (see C4). The LLD's assumption contradicts both the HLD and the existing payment API.
- problem: Story 1's LLD states `payment_reference = body.cart_id` (line 227 in the idempotency check code), but the HLD says `payment_reference = payment_data["reference"]` (returned by payment after accepting).
- change: In story 1's LLD, replace the assumption `payment_reference = body.cart_id` with `payment_reference = payment_data["reference"]` (after payment accepts). Move the idempotency check to after the payment call, not before.

### C6
- file: hld.md
- section: ## Open Questions
- quote: What is the duration of the completion window for refunds?
- match: breakdown.md › ## Story 4
- severity: note
- blocks because: (not blocking)
- problem: The HLD lists "What is the duration of the completion window for refunds?" as an open question, but the breakdown and story 4 state the default is 5 minutes (configurable via `COMPLETION_WINDOW_SECONDS`).
- change: (none; this is a note for the gate)

### C7
- file: breakdown.md
- section: ## Story 1: Checkout outbox and OrderRequested publish via RabbitMQ
- quote: Insert checkout + outbox in one write
- match: hld.md › ## Algorithm Notes › Atomic Insert: Checkout + Outbox
- severity: blocking
- blocks because: The breakdown says "insert checkout + outbox in one write (atomic)" but the HLD states MongoDB transactions require a replica set, and the existing URI is `mongodb://localhost:27017` (standalone). Dev would attempt a transaction, fail, and get stuck. The HLD provides a fallback (insert checkout first, then outbox, with manual rollback on failure), but the breakdown does not mention this.
- problem: The breakdown states the write is atomic but does not mention the fallback strategy when transactions are unavailable.
- change: In the breakdown's story 1 implementation pointers, state: "Insert checkout first, then outbox; on outbox failure, delete the checkout record (rollback). MongoDB transactions are not supported in v1 (standalone instance). Mark as `[ATOMIC_WRITE_DEFERRED]`."

## Notes for the gate

### N1
The HLD's open question "How is the relay process deployed?" is answered in the breakdown (systemd service, Docker container, or Kubernetes pod) but not marked as decided in the HLD. The person at the gate may want to close the open question.

### N2
The HLD lists "RefundCompleted event" as an open question, and story 4 defers it. The checkout status will remain `refund_pending` until a future story updates it to `refunded`. This is acceptable for v1 but should be noted at the gate.

### N3
The integration test suite (story 6) includes manual tests that require stopping and starting services. The gate may want to confirm these are acceptable or move them to a separate manual test suite.

VERDICT: FAIL
