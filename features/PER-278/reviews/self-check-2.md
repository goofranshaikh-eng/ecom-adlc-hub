# What would go wrong

## C2
- file: specs/2.md
- section: ## Schema/API Contracts
- quote: "price_snapshot_minor_units": 1500
- match: hld.md › ## Message Schemas › OrderRequested
- severity: blocking
- blocks because: The spec shows `price_snapshot_minor_units` in the `OrderRequested` message but does not state the field must be renamed to `price_minor_units` when inserting the order. Dev would store it as given, and the order document would have the wrong field name. The existing `GET /orders/{order_id}` endpoint returns `price_minor_units` (see `repo-file/app/order/routes.js`), so the response would be missing the price field.
- problem: The spec lists `price_snapshot_minor_units` in the message schema but does not state it maps to `price_minor_units` in the order document.
- change: In the "Message Schema: OrderRequested (consumed)" section, add a field note below `lines[].price_snapshot_minor_units`: "Mapped to `price_minor_units` in the order document on insert (see `repo-file/app/order/models.js`)."

---

## C3
- file: specs/1.md
- section: ## File-by-File Changes › checkout/routes.py
- quote: Insert both to MongoDB atomically (checkout first, then outbox; rollback checkout on outbox failure)
- match: hld.md › ## Algorithm Notes › Atomic Insert: Checkout + Outbox
- severity: blocking
- blocks because: The spec says "insert both atomically" and shows a try-except block with rollback, but the "Before" code block is missing. Dev would not know what code to replace. The spec must show the existing code that creates the order and deletes the cart (lines 90–115 in `repo-file/app/checkout/routes.py`) so Dev knows where to insert the new logic.
- problem: The spec shows the new code (checkout + outbox insert with rollback) but does not show the "Before" code it replaces.
- change: In the "checkout/routes.py" section, add a "Before (payment success block):" code block showing the existing order creation and cart deletion (lines 90–115 from `repo-file/app/checkout/routes.py`). Then show the "After:" code block with the checkout + outbox insert.

---

## C4
- file: specs/1.md
- section: ## File-by-File Changes › checkout/routes.py
- quote: Idempotency Check (before payment call):
- match: hld.md › ## Service Interaction Contracts
- severity: blocking
- blocks because: The spec says "before payment call" but the HLD decision states the check happens "after payment accepts, using the payment response's reference." Dev would query the database before calling payment, but the query uses `payment_reference` which is not available until after the call. The check would always find nothing and never detect duplicates.
- problem: The idempotency check is placed before the payment call, but `payment_reference` is returned by payment after the call.
- change: Move the idempotency check to after the payment accepts (line 73, after `payment_data["reference"]` is available). The revised code block should query by `payment_reference = payment_data["reference"]` and return the existing checkout if found.

---

## C5
- file: specs/1.md
- section: ## Risks › R3: Checkout and outbox insert is not atomic
- quote: Insert checkout first, then outbox; on outbox failure, delete the checkout record (rollback).
- match: stories/01-app.md › ## Algorithm Notes › Atomic Insert: Checkout + Outbox
- severity: blocking
- blocks because: The spec says "insert checkout first, then outbox; on outbox failure, delete the checkout record" and marks it `[ATOMIC_WRITE_DEFERRED]`, but the "After" code block in the "checkout/routes.py" section does not show the rollback delete. Dev would insert the checkout, fail on the outbox, and leave the checkout record in the database with no outbox row. The charge is accepted but no order will ever be created.
- problem: The risk section describes the rollback strategy, but the code block does not implement it.
- change: In the "checkout/routes.py" section's "After:" code block, wrap the outbox insert in a second try-except. On `PyMongoError`, delete the checkout record (`checkouts_collection.delete_one({"checkout_id": checkout_id})`) before raising the 500 error.

---

## C6
- file: stories/02-app.md
- section: ## Algorithm Notes › Data Mapping: `OrderRequested` → Order Document
- quote: line.price_snapshot_minor_units → orderLine.price_minor_units (rename field)
- match: hld.md › ## Message Schemas › OrderRequested
- severity: note
- blocks because: (not blocking; this is correct and already stated in the LLD)
- problem: None. The LLD correctly states the field is renamed. This item is a note to verify the mapping is implemented.
- change: (none)

---

VERDICT: FAIL
