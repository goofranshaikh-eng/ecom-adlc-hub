# Child tickets

- `01-app` → `PER-286`
- `02-app` → `PER-287`
- `03-app` → `PER-288`
- `04-app` → `PER-289`
- `05-app` → `PER-290`
- `06-app` → `PER-291`

## Sub-issues

| Sub-issue | Area | Repository | Title | Depends on | Order |
| --- | --- | --- | --- | --- | --- |
| PER-286 | app | `app` | Checkout outbox and OrderRequested publish via RabbitMQ | None | 1st |
| PER-287 | app | `app` | Order service consumes OrderRequested and publishes OrderCreated | PER-286 | 2nd |
| PER-288 | app | `app` | Checkout empties the cart on OrderCreated with retry | PER-286, PER-287 | 3rd (parallel) |
| PER-289 | app | `app` | Refund a charge when the order never appears within the completion window | PER-286, PER-287 | 3rd (parallel) |
| PER-290 | app | `app` | Checkout status endpoint returns accepted, order created, cart cleared, or refunded with order id when present | PER-286, PER-288, PER-289 | 4th |
| PER-291 | app | `app` | Integration test suite for distributed checkout acceptance criteria | PER-286, PER-287, PER-288, PER-289, PER-290 | 5th |
