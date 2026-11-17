# requirements.md

**Ticket:** PER-278  
**Title:** A paid checkout still becomes one order when order or cart is briefly down  
**Type:** Feature  
**Initiative:** `epic-distributed-checkout/`  
**Covers:** CAP-3, CAP-7, CAP-11

## Overview

When a shopper's payment is accepted, the store finishes the purchase even if the order service or the cart service is down at that moment, and a repeated checkout does not charge them twice.

## Actors

- **Client** — any caller of the checkout API, guest or signed in. It needs a clear result now, and a way to see that the order appeared.
- **Store operator** — can see that an accepted charge produced one order, or was refunded when the order could not be created. No admin UI in this job.

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

### AC1: Payment refused stays immediate

**Given** a cart at checkout  
**When** payment refuses the charge  
**Then** the response says refused, no order is created, the cart is unchanged, and nothing is published

**Verification:**
- Call `POST /checkout` with a cart id and a payment method the payment service will refuse
- Assert the response status indicates refused
- Assert no order exists for that checkout
- Assert the cart still exists with its items unchanged
- Assert no messages were published to RabbitMQ

---

### AC2: Accepted checkout returns before the order exists

**Given** a cart and a payment service that accepts  
**When** the client checks out  
**Then** the response is accepted with a checkout id while the order may not exist yet, and checkout has stored that checkout and an unpublished outbox row for it in one write

**Verification:**
- Call `POST /checkout` with a valid cart id
- Assert the response is accepted and contains a checkout id
- Assert the response is returned before the order service has processed any message
- Inspect the checkout service's database and assert both a checkout record and an outbox row exist for the same payment reference
- Assert the checkout record and outbox row were written in a single atomic operation (verify they both exist or both are absent)

---

### AC3: Order is created after the order service recovers

**Given** an accepted checkout whose `OrderRequested` message could not be handled because the order service was down  
**When** the order service is running again  
**Then** exactly one order exists with the lines, total, ship-to address, and payment reference from that checkout

**Verification:**
- Accept a checkout while the order service is stopped
- Start the order service
- Wait for message processing
- Call `GET /{checkout_id}` on the order service (verify path in `repo-outline/app`)
- Assert exactly one order exists with the payment reference from the checkout
- Assert the order's lines, total, and ship-to address match the checkout's cart contents and provided address

---

### AC4: A repeated checkout does not charge or order twice

**Given** an accepted checkout for a payment reference  
**When** the same checkout is submitted again, or the same message is delivered again  
**Then** there is still one charge and one order for that reference

**Verification:**
- Accept a checkout with a specific payment reference
- Submit the same checkout request again with the same payment reference
- Assert only one charge exists in the payment service for that reference
- Assert only one order exists for that reference
- Deliver the same `OrderRequested` message twice to the order service
- Assert the order count for that payment reference remains one

---

### AC5: The cart is emptied, and a failed delete is retried

**Given** an order created for a checkout  
**When** deleting the cart fails  
**Then** the cart is still removed on a later attempt, and the order is not created again

**Verification:**
- Accept a checkout and let the order be created
- Cause the first `DELETE /cart/{cart_id}` call to fail (network error, cart service down, or timeout)
- Verify the cart still exists after the first attempt
- Allow the retry to succeed
- Assert `GET /cart/{cart_id}` returns 404 or empty
- Assert the order count for that payment reference remains one

---

### AC6: A charge with no order is refunded

**Given** an accepted charge whose order has not been created within the completion window  
**When** that window passes  
**Then** the charge is refunded, no order is created, and the checkout status says refunded

**Verification:**
- Accept a checkout
- Prevent the order service from processing the `OrderRequested` message
- Wait for the completion window to pass
- Assert a `RefundRequested` message was published for the payment reference
- Assert the payment service refunded the charge
- Call the checkout status endpoint (verify path in `repo-outline/app`) and assert the status is refunded
- Assert no order exists for that payment reference

---

### AC7: The client can see how the checkout finished

**Given** a checkout id  
**When** the client fetches that checkout  
**Then** the status is one of accepted, order created, cart cleared, or refunded, and an order id is present once the order exists

**Verification:**
- Accept a checkout and obtain a checkout id
- Call the checkout status endpoint (verify path in `repo-outline/app`) with that id
- Assert the response contains a status field with one of: accepted, order created, cart cleared, or refunded
- When the order has been created, assert the response contains the order id
- When the order has not been created yet, assert no order id is present

---

### AC8: Services do not share databases

**Given** a completed or refunded checkout  
**When** each service is inspected  
**Then** checkout, order, payment, and cart each read and write only their own data, and the only new cross-service path is RabbitMQ

**Verification:**
- Accept a checkout through to completion or refund
- Inspect the checkout service's database connections and queries
- Assert checkout only reads and writes its own MongoDB database
- Inspect the order service's database connections and queries
- Assert order only reads and writes its own MongoDB database
- Inspect the payment service's database connections and queries
- Assert payment only reads and writes its own MongoDB database
- Inspect the cart service's database connections and queries
- Assert cart only reads and writes its own MongoDB database
- Assert no service executes a query against another service's database
- Assert the only new inter-service communication is via RabbitMQ messages

---

## Boundaries

- No storefront or admin UI. The checkout status is an HTTP response.
- Must not change: catalog price rules, the cart total sent to payment, or the refused-payment behavior from [PER-161](https://linear.app/personalgs/issue/PER-161/a-shopper-browses-a-catalog-checks-out-and-receives-a-paid-order).
- Must not change: guest checkout still requires an email and a ship-to address.
- One broker, RabbitMQ, run by the project. No paid hosted broker.
- Money stays in minor units. The payment reference is the idempotency key across checkout, payment, and order.

## Open questions

- What is the duration of the completion window for refunds?
- How is the relay process deployed and monitored?
- What logging and observability are needed for the outbox publish and message handling paths?

## Assumptions

- The payment service remains a test double that can accept, refuse, and now refund.
- v1 is still one product repo, hub key `app`, with these services as separate processes.
- Outbox means the checkout row and the "publish this" row commit together, so a crash cannot leave an accepted charge with no message to send.

## Parked

- Stock reservation
- Email confirmation
- Fraud checks
- A human review queue for refunds
