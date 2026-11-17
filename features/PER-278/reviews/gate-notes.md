# Critic notes for PER-278

Not blocking. For whoever reviews the specs at the gate.

## Notes (6)

- **C2** · `specs/2.md` › ## Schema/API Contracts: The field note for `lines[].price_snapshot_minor_units` now reads "Mapped to `price_minor_units` in the order document on insert (see HLD)." The HLD section says "see `repo-file/app/order/models.js` and … _(the critic marked it a note)_
- **C3** · `specs/1.md` › ## File-by-File Changes › checkout/routes.py: The spec shows the new code (checkout + outbox insert with rollback) but does not show the "Before" code it replaces. _(new in a re-check, in text no fix touched)_
- **C5** · `specs/1.md` › ## Risks › R3: Checkout and outbox insert is not atomic: The risk section describes the rollback strategy, but the code block does not implement it. _(new in a re-check, in text no fix touched)_
- **C6** · `hld.md` › ## Open Questions: The HLD lists "What is the duration of the completion window for refunds?" as an open question, but the breakdown and story 4 state the default is 5 minutes (configurable via `COMPLETION_WINDOW_SECONDS`). _(the critic marked it a note)_
- **C6** · `stories/02-app.md` › ## Algorithm Notes › Data Mapping: `OrderRequested` → Order Document: None. _(the critic marked it a note)_
- **C7** · `stories/01-app.md` › ## Tests › Test 3: `test_repeated_checkout_returns_existing_checkout(): The test comment says "both POSTs called payment, but second checkout was not inserted", but the breakdown's AC4 says "no second charge is made". _(new in a re-check, in text no fix touched)_

## Outdated: about text that has changed since (2)

- **C3** · `stories/02-app.md` › ## Algorithm Notes: The LLD's data mapping algorithm maps `line.price_snapshot_minor_units` to `orderLine.price_snapshot_minor_units`, but the order schema in `repo-file/app/order/models.js` uses `price_minor_units`. _(its quote is in no spec file)_
- **C5** · `stories/01-app.md` › ## Algorithm Notes: Story 1's LLD states `payment_reference = body.cart_id` (line 227 in the idempotency check code), but the HLD says `payment_reference = payment_data["reference"]` (returned by payment after accepting). _(its quote is in no spec file)_
