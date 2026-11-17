# Critic notes for PER-278

Not blocking. For whoever reviews the specs at the gate.

## Notes (4)

- **C3** · `specs/1.md` › ## Risks › R3: Checkout and outbox insert is not atomic: Fallback Strategy: 1. _(new in a re-check, in text no fix touched)_
- **C4** · `stories/01-app.md` › ## Tests › Test 6: `test_idempotency_duplicate_payment_reference(): def test_idempotency_duplicate_payment_reference(): """AC4: Duplicate cart_id returns existing checkout.""" cart_id = "cart-dup" _seed_cart(cart_id, quantity=1, price=1000) body = {"cart_id": cart_id, "email" … _(new in a re-check, in text no fix touched)_
- **C5** · `hld.md` › ## Open Questions: 1. _(the critic marked it a note)_
- **C6** · `stories/02-app.md` › ## Algorithm Notes › Data Mapping: `OrderRequested` → Order Document: Field Mapping:** - `msg.payment_reference` → `orderDoc.payment_reference` - `msg.shopper_id` → `orderDoc.shopper_id` (optional; omit field if `null` or `undefined`) - `msg.email` → `orderDoc.email` (optional; omit field … _(the critic marked it a note)_

## Outdated: about text that has changed since (2)

- **C3** · `stories/02-app.md` › ## Algorithm Notes: The LLD's data mapping algorithm maps `line.price_snapshot_minor_units` to `orderLine.price_snapshot_minor_units`, but the order schema in `repo-file/app/order/models.js` uses `price_minor_units`. _(its quote is in no spec file)_
- **C5** · `stories/01-app.md` › ## Algorithm Notes: Story 1's LLD states `payment_reference = body.cart_id` (line 227 in the idempotency check code), but the HLD says `payment_reference = payment_data["reference"]` (returned by payment after accepting). _(its quote is in no spec file)_
