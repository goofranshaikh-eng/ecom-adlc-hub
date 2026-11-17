---
key: PER-278
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: A paid checkout still becomes one order when order or cart is briefly down
---

## Job

When a shopper's payment is accepted, the store finishes the purchase even if the order service or the cart service is down at that moment, and a repeated checkout does not charge them twice.

## Who

* Client: any caller of the checkout API, guest or signed in. It needs a clear result now, and a way to see that the order appeared.
* Store operator: can see that an accepted charge produced one order, or was refunded when the order could not be created. No admin UI in this job.

## What exists when this is done

Checkout still reads the cart and asks payment over HTTP, because the shopper is waiting for accepted or refused. After an accepted charge, checkout records the purchase in its own database together with an outbox row, and returns immediately. A relay publishes that row to RabbitMQ. Order creates the order from the message. Checkout empties the cart when the order exists. If the order never appears, payment refunds the charge.

Cart's read of catalog stays HTTP. No service reads another service's database.

| Service | New responsibility |
| -- | -- |
| checkout | After an accepted charge, persist the checkout and an outbox row, publish `OrderRequested`, empty the cart on `OrderCreated`, ask for a refund if the order never arrives |
| order | Create one order per payment reference from `OrderRequested`, then publish `OrderCreated` |
| payment | Refund a charge when checkout publishes `RefundRequested` |
| cart | Unchanged API. Checkout still deletes the cart; the delete is retried |

## Acceptance criteria

1. **Payment refused stays immediate**

   **Given** a cart at checkout

   **When** payment refuses the charge

   **Then** the response says refused, no order is created, the cart is unchanged, and nothing is published
2. **Accepted checkout returns before the order exists**

   **Given** a cart and a payment service that accepts

   **When** the client checks out

   **Then** the response is accepted with a checkout id while the order may not exist yet, and checkout has stored that checkout and an unpublished outbox row for it in one write
3. **Order is created after the order service recovers**

   **Given** an accepted checkout whose `OrderRequested` message could not be handled because the order service was down

   **When** the order service is running again

   **Then** exactly one order exists with the lines, total, ship-to address, and payment reference from that checkout
4. **A repeated checkout does not charge or order twice**

   **Given** an accepted checkout for a payment reference

   **When** the same checkout is submitted again, or the same message is delivered again

   **Then** there is still one charge and one order for that reference
5. **The cart is emptied, and a failed delete is retried**

   **Given** an order created for a checkout

   **When** deleting the cart fails

   **Then** the cart is still removed on a later attempt, and the order is not created again
6. **A charge with no order is refunded**

   **Given** an accepted charge whose order has not been created within the completion window

   **When** that window passes

   **Then** the charge is refunded, no order is created, and the checkout status says refunded
7. **The client can see how the checkout finished**

   **Given** a checkout id

   **When** the client fetches that checkout

   **Then** the status is one of accepted, order created, cart cleared, or refunded, and an order id is present once the order exists
8. **Services do not share databases**

   **Given** a completed or refunded checkout

   **When** each service is inspected

   **Then** checkout, order, payment, and cart each read and write only their own data, and the only new cross-service path is RabbitMQ

## Boundaries

* No storefront or admin UI. The checkout status is an HTTP response.
* Must not change: catalog price rules, the cart total sent to payment, or the refused-payment behavior from [PER-161](https://linear.app/personalgs/issue/PER-161/a-shopper-browses-a-catalog-checks-out-and-receives-a-paid-order).
* Must not change: guest checkout still requires an email and a ship-to address.
* One broker, RabbitMQ, run by the project. No paid hosted broker.
* Money stays in minor units. The payment reference is the idempotency key across checkout, payment, and order.

## Notes

* Assumption: the payment service remains a test double that can accept, refuse, and now refund.
* Assumption: v1 is still one product repo, hub key `app`, with these services as separate processes.
* Outbox means the checkout row and the "publish this" row commit together, so a crash cannot leave an accepted charge with no message to send.
* Parked: stock reservation, email confirmation, fraud checks, and a human review queue for refunds.
