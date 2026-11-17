# ADLC spec critic

## What would go wrong

### C1
- file: specs/1.md
- section: ### Idempotency: Duplicate Checkout Submission
- quote: **Algorithm:**
1. Call `POST /charge` (payment service)
2. If payment refuses, return `CheckoutCreateResponse(status="refused", reason=...)`
3. If payment accepts:
   - `payment_reference = payment_data["reference"]`
   - Query `checkouts_collection.find_one({"payment_reference": payment_reference})`
- match: hld.md › ## Service Interaction Contracts › Checkout → Payment (HTTP, unchanged)
- severity: blocking
- blocks because: The HLD's decision says checkout queries by `cart_id` **before** calling payment, but the spec's algorithm says checkout queries by `payment_reference` **after** payment accepts. Dev would build the after-payment query, leaving the same cart checkable twice with different payment references, so AC4 (no second charge) fails.
- problem: The spec's algorithm queries after payment accepts using `payment_reference`, but the HLD decision says "Before calling `POST /charge`, checkout queries the `checkouts` collection for any checkout with the same `cart_id` and status `accepted`, `order_created`, or `cart_cleared`." The spec does not query by `cart_id` before the payment call, so the two documents disagree on when the query runs and what key it uses.
- change: Change the algorithm to: 1. Query `checkouts_collection.find_one({"cart_id": body.cart_id, "status": {"$in": ["accepted", "order_created", "cart_cleared"]}})`. 2. If found, return existing checkout without calling payment. 3. Call `POST /charge`. 4. If refused, return refused. 5. If accepted, insert checkout + outbox with `payment_reference = payment_data["reference"]`.

---

### C2
- file: specs/1.md
- section: ## File-by-File Changes › checkout/routes.py
- quote: - Add idempotency check after payment accepts (line 73):
  - Query `checkouts_collection.find_one({"payment_reference": payment_data["reference"]})`
  - If found and `status` in `["accepted", "order_created", "cart_cleared"]`, return existing `CheckoutCreateResponse(checkout_id=existing["checkout_id"], status="accepted")`
- match: hld.md › ## Service Interaction Contracts › Checkout → Payment (HTTP, unchanged)
- severity: blocking
- blocks because: The file changes say the idempotency check is added at line 73 (after payment accepts) querying by `payment_reference`, but the HLD decision says the query is by `cart_id` before calling payment. Dev would add the check in the wrong place with the wrong key.
- problem: The "checkout/routes.py" file changes section says the idempotency check is at line 73 after payment accepts, using `payment_reference`, but the HLD says the check is before calling payment using `cart_id`. The two disagree on where the check goes and what field it queries.
- change: Move the idempotency check to line 52 (before the payment call). Query by `cart_id` and status. The code block at line 73 should be removed entirely, and the revised code block should show: Before payment call (line 52): query `checkouts_collection.find_one({"cart_id": body.cart_id, "status": {"$in": ["accepted", "order_created", "cart_cleared"]}})`, return existing checkout if found, otherwise continue to payment call.

---

### C3
- file: specs/1.md
- section: ## Algorithm Notes › Atomic Insert: Checkout + Outbox
- quote: **Code:**
```python
try:
    checkouts_collection.insert_one(checkout_record.model_dump(by_alias=True))
except PyMongoError:
    raise HTTPException(status_code=500, detail="Failed to store checkout") from None

try:
    outbox_collection.insert_one(outbox_row.model_dump(by_alias=True))
except PyMongoError:
    # Rollback: delete checkout record
    checkouts_collection.delete_one({"checkout_id": checkout_id})
    raise HTTPException(status_code=500, detail="Failed to store outbox") from None
```
- match: hld.md › ## Service Interaction Contracts › Checkout → Payment (HTTP, unchanged)
- severity: blocking
- blocks because: The code shows the rollback deletes by `checkout_id`, but the HLD decision says the idempotency check queries by `cart_id`. If the rollback fails (network blip, process crash), the checkout exists with no outbox, and a retry with the same `cart_id` would find the orphaned checkout and return it without calling payment again. The retry would succeed but no order would ever be created because no outbox row exists. The HLD's idempotency rule makes the rollback unreliable.
- problem: The HLD decision says idempotency is by `cart_id`, but the rollback code deletes by `checkout_id`. A failed rollback leaves a checkout with no outbox. A retry with the same `cart_id` finds the orphaned checkout, returns it, and never creates an outbox row, so no order is created. The two decisions (idempotency by `cart_id` and rollback by `checkout_id`) conflict.
- change: Add a note in the "Known Gap" section: "[ATOMIC_WRITE_DEFERRED] If the outbox insert fails and the rollback delete also fails (process crash, network error), the checkout exists with no outbox. A retry with the same `cart_id` returns the orphaned checkout without creating an outbox row, so no order is created. This gap is deferred until MongoDB transactions are enabled." Keep the rollback code as-is but mark the gap.

---

### C4
- file: stories/01-app.md
- section: ## Tests › Test 3: `test_repeated_checkout_returns_existing_checkout()`
- quote: def test_repeated_checkout_returns_existing_checkout():
    """AC4: A repeated checkout does not charge twice."""
    cart_id = "cart-repeat"
    _seed_cart(cart_id, quantity=2, price=1500)
    
    body = {
        "cart_id": cart_id,
        "email": "guest@example.com",
        "ship_to": _ship_to().model_dump(),
    }
    
    response1 = client.post("/checkout", json=body)
    assert response1.status_code == 200
    checkout_id_1 = response1.json()["checkout_id"]
    
    response2 = client.post("/checkout", json=body)
    assert response2.status_code == 200
    checkout_id_2 = response2.json()["checkout_id"]
    
    assert checkout_id_1 == checkout_id_2
    assert response2.json()["status"] == "accepted"
    
    # Only one checkout record
    checkout = checkouts_collection.find_one({"checkout_id": checkout_id_1})
    payment_reference = checkout["payment_reference"]
    all_checkouts = [c for c in checkouts_data.values() if c.get("payment_reference") == payment_reference]
    assert len(all_checkouts) == 1
    
    # Only one outbox row
    all_outbox = [r for r in outbox_data.values() if r.get("payment_reference") == payment_reference]
    assert len(all_outbox) == 1
    
    # Only one payment charge
    payment_calls = [call for call in calls if call[1].endswith("/charge")]
    assert len(payment_calls) == 2  # Both POSTs called payment, but second checkout was not inserted
- match: hld.md › ## Service Interaction Contracts › Checkout → Payment (HTTP, unchanged)
- severity: blocking
- blocks because: The test asserts `len(payment_calls) == 2` and says "Both POSTs called payment, but second checkout was not inserted," but the HLD decision says the second POST should not call payment at all (the idempotency check is before the payment call). Dev would write the test and see it fail because the second POST does not call payment, contradicting the assertion.
- problem: The test comment says "Both POSTs called payment," but the HLD decision says the idempotency check is before the payment call, so the second POST should not call payment. The test expects two payment calls, but the HLD expects one.
- change: Change the test assertion to `assert len(payment_calls) == 1` and the comment to "First POST called payment; second POST returned existing checkout without calling payment."

---

## Notes for the gate

### N1
- file: specs/1.md
- section: ## File-by-File Changes › checkout/routes.py
- note: The "Before (payment success block):" code block is missing. The spec says to modify lines 72–120, but does not show what those lines are today. The gate may want a before-and-after diff for clarity, as the fixer's earlier note requested.

---

### N2
- file: hld.md
- section: ## Service Interaction Contracts › Checkout → Payment (HTTP, unchanged)
- note: The HLD's decision says checkout queries by `cart_id` before calling payment, but the existing code in `repo-file/app/checkout/routes.py` has no such query. The decision changes the idempotency strategy from the original plan (which queried by `payment_reference` after payment) to a new strategy (query by `cart_id` before payment). The gate should confirm this is the intended behavior, because it means two POSTs with the same `cart_id` but different ship-to addresses would return the same checkout, which may not be the desired UX.

---

VERDICT: FAIL
