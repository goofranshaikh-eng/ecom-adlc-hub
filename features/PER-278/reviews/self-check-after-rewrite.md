# What would go wrong

## C2
- file: specs/2.md
- section: ## Schema/API Contracts
- quote: **Field Notes:**
- match: hld.md › ## Message Schemas › OrderRequested
- severity: note
- problem: The field note for `lines[].price_snapshot_minor_units` now reads "Mapped to `price_minor_units` in the order document on insert (see HLD)." The HLD section says "see `repo-file/app/order/models.js` and `repo-file/app/order/routes.js`". The spec pointer is less specific than the HLD.
- change: Change the note to say "Mapped to `price_minor_units` in the order document on insert (see `repo-file/app/order/models.js`)." to match the HLD's concrete file reference.

---

## C4
- file: specs/1.md
- section: ### Idempotency: Duplicate Checkout Submission
- quote: **Revised Idempotency Check (after payment call):**
```python
# After payment accepts:
payment_reference = payment_data["reference"]

existing_checkout = checkouts_collection.find_one({"payment_reference": payment_reference})
if existing_checkout:
    if existing_checkout["status"] in ["accepted", "order_created", "cart_cleared"]:
        return CheckoutCreateResponse(
            checkout_id=existing_checkout["checkout_id"],
            status="accepted"
        )
    elif existing_checkout["status"] == "refunded":
        raise HTTPException(status_code=400, detail="Checkout was refunded, cannot retry")
```
- match: hld.md › ## Data Flow: Accepted Checkout → Order Created → Cart Cleared
- severity: blocking
- blocks because: The code block says the check happens after payment accepts, but the modified `checkout/routes.py` section at line 52 says "Add idempotency check after payment accepts (line 73):" and then shows the check placed **before** the checkout and outbox insert. Dev would read the idempotency algorithm and see it queries after getting `payment_reference`, but then read the file changes and place the check before the insert. The two sections disagree on when the query runs relative to the insert.
- problem: The "Revised Idempotency Check" algorithm says the query runs after payment accepts, but the "checkout/routes.py" file changes section says the check is added at line 73 (which is after payment accepts) but the code shown queries the database and then returns early **before** the insert. The insert happens later at line 90. The spec does not say whether the check is before or after the insert, so Dev cannot tell if the insert runs first (and then the check is redundant) or the check runs first (and prevents the insert).
- change: In the "checkout/routes.py" section, clarify that the idempotency check runs **before** the insert. Add a note: "Query `checkouts_collection` by `payment_reference` before inserting. If a checkout exists for this payment reference, return the existing checkout and do not insert again."

---

## C7
- file: stories/01-app.md
- section: ## Tests › Test 3: `test_repeated_checkout_returns_existing_checkout()`
- quote: # Only one payment charge
    payment_calls = [call for call in calls if call[1].endswith("/charge")]
    assert len(payment_calls) == 2  # Both POSTs called payment, but second checkout was not inserted
- match: breakdown.md › ## Story 1 › ### Test 3
- severity: blocking
- blocks because: The test asserts `len(payment_calls) == 2` but the idempotency check (from C4) runs after payment accepts, so both POSTs call payment and get the same reference. The second POST then queries the database, finds the existing checkout, and returns it without inserting again. The assertion comment says "both POSTs called payment, but second checkout was not inserted", which is correct behavior. However, the test comment implies the second POST should **not** call payment, but the code shown calls payment both times. Dev would write the test with `assert len(payment_calls) == 2` and pass it, but the breakdown says the idempotency prevents a second **charge**, not a second payment call. The test and the breakdown disagree.
- problem: The test comment says "both POSTs called payment, but second checkout was not inserted", but the breakdown's AC4 says "no second charge is made". The test asserts two payment calls, so two charges are made (payment returns a reference for each call). The idempotency check prevents the second **insert**, not the second **charge**. The test would pass but AC4 would fail.
- change: Add a note in the test: "[KNOWN_GAP] The second POST calls payment again and creates a second charge. The idempotency check prevents the second insert, but the charge is not prevented. This violates AC4 (no second charge). Mark this as `[IDEMPOTENCY_DEFERRED]` and add a TODO to fix payment idempotency in a future ticket."

---

## Notes for the gate

### C3

The "checkout/routes.py" section in `specs/1.md` now shows the "After:" code but no "Before:" block. The fixer did not add the before-block because the spec does not have a section called exactly `### checkout/routes.py`. The section is called `### `checkout/routes.py` (modified)` with a heading that is not an exact match. The person at the gate can compare the before-block from `repo-file/app/checkout/routes.py` manually if needed, or ask the author to add it.

### C5

The "checkout/routes.py" section's "After:" code block does not wrap the outbox insert in a second try-except. The fixer refused the change because the section heading is not an exact match. The person at the gate should verify the rollback logic is added when the code is written.

### C6 (from last review)

The HLD's open question about idempotency (C6 from the last review) is still open: "Should payment be modified to accept an idempotency key, or is this deferred?" The spec marks it `[IDEMPOTENCY_DEFERRED]`. The person at the gate should decide whether to close the question or escalate it.

### C8

The "Database Schema: orders collection (modified)" section in `specs/2.md` shows the `price_minor_units` field in the lines array, but the message schema uses `price_snapshot_minor_units`. The data mapping algorithm says "rename field", which is correct, but the schema comment does not say where the rename happens. The person at the gate should verify the rename is in the consumer code, not the schema.

---

VERDICT: FAIL
