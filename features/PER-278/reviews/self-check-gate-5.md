## What would go wrong

### C1
- file: specs/1.md
- section: ### Idempotency: Duplicate Checkout Submission
- quote: 2. If no existing checkout, generate `checkout_id = str(uuid.uuid4())` and `payment_reference = str(uuid.uuid4())`
3. Insert pending checkout: `checkouts_collection.insert_one({checkout_id, payment_reference, cart_id, ..., status: "pending", created_at: now(), updated_at: now()})`
4. Call `POST /charge` with `{"amount_minor_units": cart_total, "payment_reference": payment_reference}`
- match: checkout/routes.py (repo-file) line 93 - line 103 (the payment success block after line 72)
- severity: blocking
- blocks because: The spec says checkout generates `payment_reference` before calling payment and sends it as the idempotency key, but the existing code at line 103 uses `payment_data["reference"]` from the payment response as the reference, and the payment service (repo-file/app/payment/routes.py) generates its own UUID and does not accept or honor a `payment_reference` in the request. Dev would build a payment_reference field in the charge request that payment ignores, so a repeated cart_id submits twice and creates two charges with different references.
- problem: The spec says checkout generates payment_reference and sends it to payment as the idempotency key, but the existing payment service (repo-file/app/payment/routes.py lines 70-71) generates `reference = str(uuid.uuid4())` and does not accept a payment_reference field in ChargeRequest. The HLD decision says "Checkout implements idempotency by querying its own database before calling payment," which matches the first step, but the payment call at step 4 would send a reference payment does not read.
- change: Remove step 2's `payment_reference = str(uuid.uuid4())` and step 4's `"payment_reference": payment_reference` from the charge request. Keep the pending checkout insert with no payment_reference yet. After payment accepts (step 5), set `payment_reference = payment_data["reference"]` before inserting the outbox row. The outbox insert and checkout update both use that reference. The payment service stays unchanged (does not accept payment_reference). Add a note: "[IDEMPOTENCY_DEFERRED] payment does not accept an idempotency key in v1; a repeated charge creates a second charge with a different reference. Checkout detects duplicate cart_id before calling payment to prevent this."

---

### C2
- file: stories/01-app.md
- section: ## Tests
- quote: # Only one checkout record
checkout = checkouts_collection.find_one({"checkout_id": checkout_id_1})
payment_reference = checkout["payment_reference"]
all_checkouts = [c for c in checkouts_data.values() if c.get("payment_reference") == payment_reference]
assert len(all_checkouts) == 1
- match: test code for test_repeated_checkout_returns_existing_checkout
- severity: blocking
- blocks because: The test asserts one checkout exists for the payment_reference, but if two POSTs to /checkout with the same cart_id both succeed (second returns the first's checkout_id), they would have the same payment_reference only if the second POST did not call payment. The first POST calls payment and gets a reference; the second POST returns the existing checkout without calling payment, so there is no second reference. Dev would read the test, see it assumes the second POST shares the reference of the first, and build code that calls payment twice but stores the same reference, which is wrong.
- problem: The test checks that all checkouts with the same payment_reference is exactly one, but it does not assert that payment was called only once. If the idempotency check failed, payment would be called twice and create two charges with different references, so the two checkouts would have different payment_references and the test would pass even though AC4 (no second charge) is broken.
- change: Before the `all_checkouts` assertion, add: `payment_calls = [call for call in calls if call[1].endswith("/charge")]` and `assert len(payment_calls) == 1, "Payment was called more than once"`. Then keep the `all_checkouts` assertion as-is.

---

## Notes for the gate

### C3
- file: specs/1.md
- section: ## Risks › R3: Checkout and outbox insert is not atomic
- quote: Fallback Strategy:
1. Insert checkout record first: `checkouts_collection.insert_one(checkout_record.model_dump(by_alias=True))`
2. Insert outbox row second: `outbox_collection.insert_one(outbox_row.model_dump(by_alias=True))`
3. On outbox insert failure:
   - Log the error
   - Delete the checkout record: `checkouts_collection.delete_one({"checkout_id": checkout_id})`
   - Raise `HTTPException(status_code=500, detail="Failed to store checkout")`
- match: checkout/routes.py (the modified file section)
- note: The rollback delete at step 3 itself can fail (process crash after the outbox insert fails but before the delete, or the delete itself fails). The spec acknowledges this in the "Known Gap" section below: "If the process crashes between the two inserts, the checkout exists but the outbox row does not." A person at the gate should decide whether to require a MongoDB transaction (enabling replica set) or accept this gap. The fixer cannot add replica set configuration or change the deployment, so this is not a blocking item.

---

### C4
- file: stories/01-app.md
- section: ## Tests › Test 6: `test_idempotency_duplicate_payment_reference()`
- quote: def test_idempotency_duplicate_payment_reference():
    """AC4: Duplicate cart_id returns existing checkout."""
    cart_id = "cart-dup"
    _seed_cart(cart_id, quantity=1, price=1000)
    
    body = {"cart_id": cart_id, "email": "guest@example.com", "ship_to": _ship_to().model_dump()}
    
    response1 = client.post("/checkout", json=body)
    assert response1.status_code == 200
    checkout_id_1 = response1.json()["checkout_id"]
    
    response2 = client.post("/checkout", json=body)
    assert response2.status_code == 200
    checkout_id_2 = response2.json()["checkout_id"]
    
    assert checkout_id_1 == checkout_id_2  # Idempotency by cart_id
    
    all_checkouts = [c for c in checkouts_data.values() if c.get("cart_id") == cart_id and c.get("status") in ["accepted", "order_created", "cart_cleared"]]
    assert len(all_checkouts) == 1
- match: test_repeated_checkout_returns_existing_checkout (Test 3)
- note: Test 6 is identical to Test 3 except for the test name and the final assertion (Test 3 uses payment_reference, Test 6 uses cart_id). Both tests verify the same behavior (repeated POST returns the same checkout_id). The person at the gate should decide whether to keep both tests or remove Test 6 as redundant. This is not a blocking item because the tests do not contradict each other; they both pass when the code is correct.

---

### C5
- file: hld.md
- section: ## Open Questions
- quote: 1. **Completion window duration:** Default is 5 minutes. Is this acceptable, or should it be configurable per checkout?
2. **RefundCompleted event:** Should payment service publish `RefundCompleted`, or should the completion window checker poll payment status to update checkout to `refunded`?
3. **Relay deployment:** How is the relay process deployed (systemd, Docker, Kubernetes)? How many instances?
4. **Completion window checker deployment:** Cron job, systemd timer, or background worker?
5. **RabbitMQ connection pooling:** Should the relay and consumers use connection pooling, or one connection per instance?
6. **Dead Letter Queue handling:** How are DLQ messages processed (manual review, automated retry, alert)?
7. **Cart delete retry limit:** Current limit is 3 retries over 1 minute. Should this be configurable, or should retries be infinite?
- match: (no match needed; these are open questions for the person at the gate)
- note: The HLD lists seven open questions. All are deployment or configuration choices that do not block implementation. The stories defer these to the gate. A person at the gate should answer each question before the merge-to-main gate, or accept the defaults in the HLD and defer the answers to a future ticket.

---

### C6
- file: stories/02-app.md
- section: ## Algorithm Notes › Data Mapping: `OrderRequested` → Order Document
- quote: **Field Mapping:**
- `msg.payment_reference` → `orderDoc.payment_reference`
- `msg.shopper_id` → `orderDoc.shopper_id` (optional; omit field if `null` or `undefined`)
- `msg.email` → `orderDoc.email` (optional; omit field if `null` or `undefined`)
- `msg.ship_to` → `orderDoc.ship_to` (object, required)
- `msg.lines[]` → `orderDoc.lines[]`:
  - `line.product_id` → `orderLine.product_id`
  - `line.quantity` → `orderLine.quantity`
  - `line.price_snapshot_minor_units` → `orderLine.price_minor_units` (rename field)
- `msg.total_minor_units` → `orderDoc.total_minor_units`
- `new Date()` → `orderDoc.created_at`
- match: hld.md › ## Message Schemas › OrderRequested
- note: The message schema uses `price_snapshot_minor_units`, but the order document schema (repo-file/app/order/models.js) uses `price_minor_units`. The story correctly maps the field name on insert. This is not a gap; it is a deliberate rename. The person at the gate should verify that the name change is intentional and documented, or flag it as a data model inconsistency. The HLD's field note at the OrderRequested schema says "Mapped to `price_minor_units` in the order document on insert," so the documents agree.

---

VERDICT: FAIL
