# breakdown.md

**Epic:** PER-278 — A paid checkout still becomes one order when order or cart is briefly down  
**Initiative:** `epic-distributed-checkout/`  
**Covers:** CAP-3, CAP-7, CAP-11

---

## Overview

Checkout becomes async and resilient through an outbox, RabbitMQ, and event-driven order creation. After an accepted charge, checkout persists the checkout and an outbox row atomically, then returns immediately. A relay publishes the outbox row to RabbitMQ. Order consumes `OrderRequested` and publishes `OrderCreated`. Checkout empties the cart when the order exists. If the order never appears, payment refunds the charge.

This breakdown splits the work into six stories, one per service slice: checkout outbox and publish, order consume and create, cart retry, refund logic, status endpoint, and integration tests. Each story is a session-sized implementation step that can be built, tested, and merged independently. Stories are sequenced so providers are ready before consumers: checkout outbox before order consumer, order consumer before cart retry, refund outbox before payment consumer, and all services before the integration test suite.

---

## Story 1: Checkout outbox and OrderRequested publish via RabbitMQ

**Repo:** `app`

### Context

Checkout must persist the checkout and an outbox row atomically after an accepted charge, then return immediately without waiting for the order service. A relay process polls the outbox and publishes `OrderRequested` to RabbitMQ. This story adds the outbox pattern to checkout, the RabbitMQ publisher relay, and the completion window checker that triggers refunds when orders never appear.

### Scope

- `checkout/db.py` — add `checkouts_collection` and `outbox_collection` references
- `checkout/models.py` — add `CheckoutRecord`, `OutboxRow`, `CheckoutCreateResponse` models
- `checkout/routes.py` — modify `POST /checkout` to insert checkout + outbox atomically, return before order exists
- `checkout/outbox.py` (new) — outbox model, `insert_outbox()`, `mark_published()` helpers
- `checkout/relay.py` (new) — relay process main loop, RabbitMQ publisher
- `checkout/completion_checker.py` (new) — completion window checker main loop
- `checkout/rabbitmq_config.py` (new) — RabbitMQ connection and channel setup
- `checkout/requirements.txt` — add `pika`

### Implementation Pointers

- `checkout/db.py:4` — after `db = client["checkout_db"]`, add `checkouts_collection = db["checkouts"]` and `outbox_collection = db["outbox"]`
- `checkout/models.py` — add `CheckoutRecord(BaseModel)` with `checkout_id: str`, `payment_reference: str`, `cart_id: str`, `shopper_id: Optional[str]`, `email: Optional[str]`, `ship_to: ShipTo`, `lines: List[CheckoutLine]`, `total_minor_units: int`, `status: Literal["accepted", "order_created", "cart_cleared", "refunded"]`, `order_id: Optional[str]`, `created_at: datetime`, `updated_at: datetime`
- `checkout/models.py` — add `OutboxRow(BaseModel)` with `event_type: Literal["OrderRequested", "RefundRequested"]`, `payload: dict`, `payment_reference: str`, `published: bool`, `published_at: Optional[datetime]`, `created_at: datetime`
- `checkout/models.py` — add `CheckoutCreateResponse(BaseModel)` with `checkout_id: str`, `status: Literal["accepted", "refused"]`, `reason: Optional[str]`
- `checkout/routes.py:72` (payment success block after `payment_response.status_code == 200`) — replace order creation and cart deletion with: generate `checkout_id` (UUID), build `CheckoutRecord` and `OutboxRow`, insert both atomically (checkout first, then outbox; rollback checkout on outbox failure), return `CheckoutCreateResponse(checkout_id, status="accepted")`
- `checkout/routes.py:52` (before payment call) — add idempotency check: query `checkouts_collection.find_one({"payment_reference": payment_data["reference"]})` after payment accepts, return existing checkout if found and status is `accepted`, `order_created`, or `cart_cleared`; raise 400 if status is `refunded`
- `checkout/outbox.py` (new) — `insert_outbox(row: OutboxRow)` inserts to `outbox_collection`, `mark_published(row_id: ObjectId)` updates `published = True` and `published_at = now`
- `checkout/relay.py` (new) — `publish_outbox_messages()` polls `outbox_collection.find({"published": False})`, publishes each to RabbitMQ (`order.requested` or `payment.refund.requested` routing key), marks published after RabbitMQ acks
- `checkout/completion_checker.py` (new) — `check_expired_checkouts()` polls `checkouts_collection.find({"status": "accepted", "created_at": {"$lt": now - COMPLETION_WINDOW_SECONDS}})`, inserts `RefundRequested` outbox row, updates checkout status to `refund_pending`
- `checkout/rabbitmq_config.py` (new) — `get_rabbitmq_connection()` returns `pika.BlockingConnection`, `setup_exchange_and_queue(channel, routing_key, queue_name)` declares `ecom.events` exchange and binds queue

### Acceptance Criteria

1. **Accepted checkout returns before the order exists** (AC2)  
   **Given** a cart and a payment service that accepts  
   **When** the client checks out  
   **Then** the response is accepted with a checkout id while the order may not exist yet, and checkout has stored that checkout and an unpublished outbox row for it in one write

2. **A repeated checkout does not charge twice** (AC4, checkout side)  
   **Given** an accepted checkout for a payment reference  
   **When** the same checkout is submitted again  
   **Then** the response returns the existing checkout id and status, no second charge is made, and no second outbox row is inserted

3. **Services do not share databases** (AC8, checkout side)  
   **Given** a completed or refunded checkout  
   **When** the checkout service is inspected  
   **Then** checkout reads and writes only `checkout_db` (collections: `checkouts`, `outbox`), and no other service's database is accessed

### Dependencies

None. This story is foundational.

### Out of Scope

- Order creation from `OrderRequested` (Story 2)
- Cart deletion on `OrderCreated` (Story 3)
- Payment refund from `RefundRequested` (Story 4)
- Checkout status endpoint (Story 5)

### Verification Approach

**7 tests:**

1. `test_accepted_checkout_returns_before_order_exists()` — call `POST /checkout` with valid cart, assert response status is `accepted` and contains `checkout_id`, assert checkout record and unpublished outbox row exist in `checkout_db`, assert no order exists yet
2. `test_checkout_and_outbox_inserted_atomically()` — call `POST /checkout`, assert both checkout and outbox exist or both are absent (implicit atomicity check: if both queries succeed, the write was atomic)
3. `test_repeated_checkout_returns_existing_checkout()` — call `POST /checkout` twice with same cart, assert second call returns same `checkout_id` and no second outbox row is inserted
4. `test_relay_publishes_outbox_message()` — insert an unpublished outbox row, run relay for 2 seconds, assert outbox row is marked published and `OrderRequested` message is in RabbitMQ queue
5. `test_completion_checker_inserts_refund_request()` — insert a checkout with `status: "accepted"` and `created_at < now - COMPLETION_WINDOW_SECONDS`, run completion checker for 2 seconds, assert `RefundRequested` outbox row exists and checkout status is `refund_pending`
6. `test_idempotency_duplicate_payment_reference()` — accept a checkout, call `POST /checkout` again with a new cart but same payment reference (edge case: payment service returns same reference), assert second call returns existing checkout
7. `test_database_isolation_checkout_side()` — accept a checkout, assert `checkout_db` contains only `checkouts` and `outbox` collections, assert no writes to `order_db`, `payment_db`, or `cart_db`

---

## Story 2: Order service consumes OrderRequested and publishes OrderCreated

**Repo:** `app`

### Context

Order must consume `OrderRequested` from RabbitMQ, create one order per payment reference (idempotency), and publish `OrderCreated`. This story adds the RabbitMQ consumer to the Node.js order service, the `payment_reference` field to the order schema, and the idempotency logic.

### Scope

- `order/models.js` — add `payment_reference` field to `createOrderSchema` (required, non-empty string)
- `order/db.js` — add unique index on `payment_reference` in `orders` collection
- `order/consumer.js` (new) — RabbitMQ consumer for `OrderRequested`, idempotency check, insert order, publish `OrderCreated`
- `order/rabbitmq.js` (new) — RabbitMQ connection setup, channel creation, exchange and queue declaration, publisher and consumer helpers
- `order/index.js` — start RabbitMQ consumer after MongoDB connection, add graceful shutdown for RabbitMQ connection and channel
- `order/package.json` — add `amqplib` dependency

### Implementation Pointers

- `order/models.js:7` (inside `createOrderSchema`) — add `payment_reference: Joi.string().trim().required()`
- `order/db.js:11` (inside `connectDB()`, after `await client.connect()`) — add `await db.collection("orders").createIndex({ payment_reference: 1 }, { unique: true })`
- `order/consumer.js` (new) — `consumeOrderRequested()` connects to RabbitMQ, subscribes to `order.requested.queue`, validates message body with Joi schema, queries `orders.findOne({ payment_reference })`, inserts order if not exists (catches duplicate key error code 11000), publishes `OrderCreated` with `{event_type, payment_reference, checkout_id, order_id, timestamp}`, acks message after processing
- `order/rabbitmq.js` (new) — `connectRabbitMQ()` returns connection and channel, `setupExchangeAndQueue(channel, exchange, queue, routingKey)` declares exchange and binds queue, `publishMessage(channel, exchange, routingKey, payload)` publishes with publisher confirms
- `order/index.js:24` (after `connectDB()` resolves) — `import { consumeOrderRequested } from './consumer.js'`, call `consumeOrderRequested()`, add `process.on('SIGTERM')` and `process.on('SIGINT')` to close RabbitMQ channel and connection
- `order/package.json:12` (inside `dependencies`) — add `"amqplib": "^0.10.3"`

### Acceptance Criteria

1. **Order is created after the order service recovers** (AC3)  
   **Given** an accepted checkout whose `OrderRequested` message could not be handled because the order service was down  
   **When** the order service is running again  
   **Then** exactly one order exists with the lines, total, ship-to address, and payment reference from that checkout

2. **Duplicate message does not create two orders** (AC4, order side)  
   **Given** an accepted checkout whose `OrderRequested` message is delivered twice  
   **When** the order service consumes both messages  
   **Then** there is still one order for that payment reference

3. **Services do not share databases** (AC8, order side)  
   **Given** a completed or refunded checkout  
   **When** the order service is inspected  
   **Then** order reads and writes only `order_db` (collection: `orders`), and no other service's database is accessed

### Dependencies

**Story 1** — Checkout outbox and OrderRequested publish via RabbitMQ (checkout publishes `OrderRequested` messages)

### Out of Scope

- Cart deletion on `OrderCreated` (Story 3)
- Payment refund from `RefundRequested` (Story 4)
- Checkout status endpoint (Story 5)

### Verification Approach

**6 tests:**

1. `test_order_created_from_order_requested()` — publish `OrderRequested` to RabbitMQ, wait for order service to consume, assert order exists in `order_db` with correct `payment_reference`, `lines`, `total_minor_units`, `ship_to`
2. `test_order_created_after_service_recovers()` — stop order service, publish `OrderRequested`, start order service, wait for message consumption, assert order exists
3. `test_duplicate_message_creates_one_order()` — publish same `OrderRequested` message twice, assert only one order exists for that `payment_reference`
4. `test_duplicate_key_error_handled()` — publish `OrderRequested`, manually insert duplicate order with same `payment_reference` before first message is consumed, assert consumer catches duplicate key error (code 11000) and publishes `OrderCreated` with existing order id
5. `test_order_created_published_after_insert()` — publish `OrderRequested`, wait for order service to consume and publish `OrderCreated`, consume `OrderCreated` from `order.created.queue`, assert message contains correct `order_id` and `payment_reference`
6. `test_database_isolation_order_side()` — create an order from `OrderRequested`, assert `order_db` contains only `orders` collection, assert no writes to `checkout_db`, `payment_db`, or `cart_db`

---

## Story 3: Checkout empties the cart on OrderCreated with retry

**Repo:** `app`

### Context

Checkout must consume `OrderCreated` from RabbitMQ, update the checkout status to `order_created`, attempt to delete the cart, and retry on failure (exponential backoff: 5s, 10s, 20s, max 3 retries). If all retries fail, log a warning and leave the checkout status as `order_created`. On success, update the checkout status to `cart_cleared`.

### Scope

- `checkout/consumers.py` (new) — RabbitMQ consumer for `OrderCreated`, cart delete with retry logic
- `checkout/main.py` — start RabbitMQ consumer on app startup (background task or thread)
- `checkout/requirements.txt` — add `pika` (already added in Story 1, verify)

### Implementation Pointers

- `checkout/consumers.py` (new) — `consumeOrderCreated()` connects to RabbitMQ, subscribes to `order.created.queue`, validates message body, queries `checkouts_collection.find_one({"payment_reference": msg["payment_reference"]})`, updates `status` to `order_created` and sets `order_id`, attempts `DELETE /cart/{cart_id}` (verify URL in `checkout/routes.py:30` — `CART_SERVICE_URL`), retries on failure (5s, 10s, 20s delays), updates `status` to `cart_cleared` on success, acks message after processing
- `checkout/main.py:14` (after FastAPI app creation) — `import { consumeOrderCreated } from './consumers.py'`, start `consumeOrderCreated()` as a background task with `asyncio.create_task()` or threading
- `checkout/consumers.py` (retry logic) — `async def delete_cart_with_retry(cart_id: str, max_retries=3, delays=[5, 10, 20])` calls `httpx.AsyncClient().delete(f"{CART_SERVICE_URL}/cart/{cart_id}")`, catches `httpx.RequestError` and 500 responses, waits `delays[retry_count]` between retries, logs failures, returns success boolean

### Acceptance Criteria

1. **The cart is emptied** (AC5, happy path)  
   **Given** an order created for a checkout  
   **When** deleting the cart succeeds  
   **Then** the cart is removed and the checkout status is `cart_cleared`

2. **A failed delete is retried** (AC5, retry path)  
   **Given** an order created for a checkout  
   **When** deleting the cart fails (cart service down or returns 500)  
   **Then** the cart is still removed on a later attempt (after retry succeeds), and the order is not created again

3. **Services do not share databases** (AC8, checkout consumer side)  
   **Given** a completed checkout  
   **When** the checkout consumer is inspected  
   **Then** checkout reads and writes only `checkout_db` (updates `checkouts` collection), and no other service's database is accessed

### Dependencies

**Story 1** — Checkout outbox and OrderRequested publish via RabbitMQ (checkout database schema exists)  
**Story 2** — Order service consumes OrderRequested and publishes OrderCreated (order service publishes `OrderCreated` messages)

### Out of Scope

- Payment refund from `RefundRequested` (Story 4)
- Checkout status endpoint (Story 5)

### Verification Approach

**5 tests:**

1. `test_cart_deleted_on_order_created()` — publish `OrderCreated`, wait for checkout consumer to process, assert cart no longer exists in `cart_db`, assert checkout status is `cart_cleared`
2. `test_cart_delete_retried_on_failure()` — publish `OrderCreated`, mock cart service to return 500 on first DELETE request, return 204 on second, wait for retry, assert cart is deleted after retry, assert checkout status is `cart_cleared`
3. `test_cart_delete_retry_exhausted()` — publish `OrderCreated`, mock cart service to return 500 on all DELETE requests, wait for all retries to fail, assert cart still exists, assert checkout status is `order_created` (not `cart_cleared`), assert warning is logged
4. `test_duplicate_order_created_message()` — publish same `OrderCreated` message twice, assert checkout is updated once (idempotency: status is already `order_created` or `cart_cleared` on second message), assert cart delete is attempted once
5. `test_database_isolation_checkout_consumer()` — consume `OrderCreated`, assert `checkout_db.checkouts` is updated, assert no writes to `order_db`, `payment_db`, or `cart_db` (cart delete is HTTP call, not database write)

---

## Story 4: Refund a charge when the order never appears within the completion window

**Repo:** `app`

### Context

When a checkout remains in `accepted` status beyond the completion window (default 5 minutes), the completion window checker inserts a `RefundRequested` outbox row and updates the checkout status to `refund_pending`. The relay publishes the message to RabbitMQ. Payment consumes `RefundRequested`, refunds the charge (updates `status` to `refunded`, sets `refunded_at`), and publishes `RefundCompleted` (optional, deferred). This story adds the payment consumer for `RefundRequested`.

### Scope

- `payment/models.py` — add `"refunded"` to `ChargeResponse.status` Literal, add `refunded_at: Optional[datetime]` to internal charge schema (docstring)
- `payment/consumers.py` (new) — RabbitMQ consumer for `RefundRequested`, refund logic, idempotency check
- `payment/main.py` — start RabbitMQ consumer on app startup (background task or thread)
- `payment/requirements.txt` — add `pika`
- `checkout/completion_checker.py` — (already added in Story 1, verify)

### Implementation Pointers

- `payment/models.py:9` (inside `ChargeResponse`) — change `status: Literal["accepted", "refused"]` to `status: Literal["accepted", "refused", "refunded"]`
- `payment/consumers.py` (new) — `consumeRefundRequested()` connects to RabbitMQ, subscribes to `payment.refund.requested.queue`, validates message body, queries `charges.find_one({"reference": msg["payment_reference"]})`, checks `status == "refunded"` (idempotency), updates `status` to `refunded` and sets `refunded_at` if not already refunded, acks message after processing
- `payment/main.py:14` (after FastAPI app creation) — `import { consumeRefundRequested } from './consumers.py'`, start `consumeRefundRequested()` as a background task with `asyncio.create_task()` or threading
- `checkout/completion_checker.py:22` (inside `check_expired_checkouts()` loop) — query `checkouts_collection.find({"status": "accepted", "created_at": {"$lt": now - timedelta(seconds=COMPLETION_WINDOW_SECONDS)}})`, for each expired checkout, insert `{"event_type": "RefundRequested", "payload": {event_type, payment_reference, checkout_id, timestamp}, "payment_reference": ..., "published": False, "created_at": now}` to `outbox_collection`, update checkout `status` to `refund_pending`

### Acceptance Criteria

1. **A charge with no order is refunded** (AC6)  
   **Given** an accepted charge whose order has not been created within the completion window  
   **When** that window passes  
   **Then** the charge is refunded, no order is created, and the checkout status says refunded (or `refund_pending` if `RefundCompleted` event is not consumed yet)

2. **Services do not share databases** (AC8, payment consumer side)  
   **Given** a refunded checkout  
   **When** the payment service is inspected  
   **Then** payment reads and writes only `payment_db` (updates `charges` collection), and no other service's database is accessed

### Dependencies

**Story 1** — Checkout outbox and OrderRequested publish via RabbitMQ (completion window checker and outbox exist)

**Why Story 2 is not a dependency:** Refund logic is independent of order creation. The completion window checker inserts `RefundRequested` when no order exists, and payment refunds the charge. Order service is not involved.

### Out of Scope

- Checkout status endpoint (Story 5)
- `RefundCompleted` event from payment to checkout (deferred; checkout status update to `refunded` is not required for AC6)

### Verification Approach

**5 tests:**

1. `test_charge_refunded_after_completion_window()` — insert a checkout with `status: "accepted"` and `created_at < now - 5 minutes`, run completion checker for 2 seconds, assert `RefundRequested` outbox row exists, run relay to publish message, run payment consumer, assert charge status is `refunded` in `payment_db`
2. `test_refund_idempotency()` — publish same `RefundRequested` message twice, assert charge is refunded once, assert second message is acked without error
3. `test_no_order_created_after_refund()` — refund a checkout, start order service, assert no order is created for that `payment_reference` (order service does not consume `OrderRequested` after refund)
4. `test_checkout_status_refund_pending()` — insert expired checkout, run completion checker, assert checkout status is `refund_pending`, assert `RefundRequested` outbox row exists
5. `test_database_isolation_payment_consumer()` — consume `RefundRequested`, assert `payment_db.charges` is updated, assert no writes to `checkout_db`, `order_db`, or `cart_db`

---

## Story 5: Checkout status endpoint returns accepted, order created, cart cleared, or refunded with order id when present

**Repo:** `app`

### Context

Clients need a way to see how their checkout finished. The status endpoint returns the current status (`accepted`, `order_created`, `cart_cleared`, or `refunded`) and the order id once the order exists. This story adds the `GET /checkout/{checkout_id}` endpoint to the checkout service.

### Scope

- `checkout/models.py` — add `CheckoutStatusResponse` model
- `checkout/routes.py` — add `GET /checkout/{checkout_id}` endpoint

### Implementation Pointers

- `checkout/models.py` — add `CheckoutStatusResponse(BaseModel)` with `checkout_id: str`, `status: Literal["accepted", "order_created", "cart_cleared", "refunded"]`, `order_id: Optional[str]`, `created_at: str` (ISO 8601 timestamp), and a `@model_validator` to enforce `order_id` is present when status is `order_created` or `cart_cleared`
- `checkout/routes.py` (new endpoint) — `@app.get("/checkout/{checkout_id}", response_model=CheckoutStatusResponse)` queries `checkouts_collection.find_one({"checkout_id": checkout_id})`, returns 404 if not found, returns 500 on database query failure, converts `created_at` datetime to ISO 8601 string, returns `CheckoutStatusResponse`

### Acceptance Criteria

1. **The client can see how the checkout finished** (AC7)  
   **Given** a checkout id  
   **When** the client fetches that checkout  
   **Then** the status is one of `accepted`, `order_created`, `cart_cleared`, or `refunded`, and an order id is present once the order exists

### Dependencies

**Story 1** — Checkout outbox and OrderRequested publish via RabbitMQ (checkout database schema exists)  
**Story 3** — Checkout empties the cart on OrderCreated with retry (checkout status is updated to `cart_cleared`)  
**Story 4** — Refund a charge when the order never appears within the completion window (checkout status is updated to `refunded` or `refund_pending`)

### Out of Scope

- Authentication (endpoint is public in v1)
- Backfilling `status` and `created_at` for existing checkouts (migration deferred)

### Verification Approach

**4 tests:**

1. `test_checkout_status_accepted()` — accept a checkout, immediately call `GET /checkout/{checkout_id}`, assert status is `accepted` and no `order_id` is present
2. `test_checkout_status_order_created()` — accept a checkout, wait for order to be created, call `GET /checkout/{checkout_id}`, assert status is `order_created` and `order_id` is present
3. `test_checkout_status_cart_cleared()` — accept a checkout, wait for order to be created and cart to be deleted, call `GET /checkout/{checkout_id}`, assert status is `cart_cleared` and `order_id` is present
4. `test_checkout_status_refunded()` — accept a checkout, prevent order creation, wait for completion window to pass, call `GET /checkout/{checkout_id}`, assert status is `refunded` (or `refund_pending`) and no `order_id` is present

---

## Story 6: Integration test suite for distributed checkout acceptance criteria

**Repo:** `app`

### Context

All acceptance criteria must be verified in integration tests that run against live instances of checkout, order, payment, and cart services with RabbitMQ running. This story adds the test suite to `checkout/tests/test_distributed_checkout.py`. The suite demonstrates that payment refused stays immediate, accepted checkout returns before the order exists, orders are created after recovery, duplicate checkouts and messages are idempotent, cart deletion is retried, charges are refunded when orders never appear, the checkout status endpoint works, and no service shares another service's database.

### Scope

- `checkout/tests/test_distributed_checkout.py` (new) — integration test suite for AC1–AC8
- `checkout/requirements.txt` — add `pytest-asyncio` (if not already present)

### Implementation Pointers

- `checkout/tests/test_distributed_checkout.py` (new) — `setup_function()` clears all databases (`checkout_db`, `order_db`, `payment_db`, `cart_db`, `identity_db`, `catalog_db`), verifies `COMPLETION_WINDOW_SECONDS=5` and `CART_DELETE_RETRY_DELAYS=1,2,3` environment variables are set (skip tests if not), defines service URL constants (`CHECKOUT_URL`, `CART_URL`, `PAYMENT_URL`, `ORDER_URL`), defines helper functions `_create_cart()`, `_ship_to()`, `_poll_until(condition_fn, timeout_seconds=10, poll_interval=0.5)`
- `checkout/tests/test_distributed_checkout.py` — write 8 async test functions (one per AC): `test_payment_refused_stays_immediate()`, `test_accepted_checkout_returns_before_order_exists()`, `test_order_created_after_recovery()`, `test_repeated_checkout_does_not_charge_twice()`, `test_cart_emptied_and_retried()`, `test_charge_refunded_when_order_never_appears()`, `test_checkout_status_endpoint()`, `test_database_isolation()`
- Each test follows the Given/When/Then structure from requirements.md, uses `httpx.AsyncClient()` for HTTP calls, uses `pymongo` for database assertions, uses `pika` for RabbitMQ assertions (verify queue depth, consume messages)

### Acceptance Criteria

1. **Payment refused stays immediate** (AC1)  
   **Given** a cart at checkout  
   **When** payment refuses the charge  
   **Then** the response says refused, no order is created, the cart is unchanged, and nothing is published

2. **Accepted checkout returns before the order exists** (AC2)  
   **Given** a cart and a payment service that accepts  
   **When** the client checks out  
   **Then** the response is accepted with a checkout id while the order may not exist yet, and checkout has stored that checkout and an unpublished outbox row for it in one write

3. **Order is created after the order service recovers** (AC3)  
   **Given** an accepted checkout whose `OrderRequested` message could not be handled because the order service was down  
   **When** the order service is running again  
   **Then** exactly one order exists with the lines, total, ship-to address, and payment reference from that checkout

4. **A repeated checkout does not charge or order twice** (AC4)  
   **Given** an accepted checkout for a payment reference  
   **When** the same checkout is submitted again, or the same message is delivered again  
   **Then** there is still one charge and one order for that reference

5. **The cart is emptied, and a failed delete is retried** (AC5)  
   **Given** an order created for a checkout  
   **When** deleting the cart fails  
   **Then** the cart is still removed on a later attempt, and the order is not created again

6. **A charge with no order is refunded** (AC6)  
   **Given** an accepted charge whose order has not been created within the completion window  
   **When** that window passes  
   **Then** the charge is refunded, no order is created, and the checkout status says refunded

7. **The client can see how the checkout finished** (AC7)  
   **Given** a checkout id  
   **When** the client fetches that checkout  
   **Then** the status is one of accepted, order created, cart cleared, or refunded, and an order id is present once the order exists

8. **Services do not share databases** (AC8)  
   **Given** a completed or refunded checkout  
   **When** each service is inspected  
   **Then** checkout, order, payment, and cart each read and write only their own data, and the only new cross-service path is RabbitMQ

### Dependencies

**Story 1** — Checkout outbox and OrderRequested publish via RabbitMQ  
**Story 2** — Order service consumes OrderRequested and publishes OrderCreated  
**Story 3** — Checkout empties the cart on OrderCreated with retry  
**Story 4** — Refund a charge when the order never appears within the completion window  
**Story 5** — Checkout status endpoint returns accepted, order created, cart cleared, or refunded with order id when present

### Out of Scope

- Performance tests (throughput, latency)
- Chaos engineering tests (network partitions, disk full, memory exhaustion)
- Load tests (concurrent checkouts, message backlog)

### Verification Approach

**8 integration tests:**

1. `test_payment_refused_stays_immediate()` — call `POST /checkout` with card `4111111111110000`, assert response is refused, no checkout, no order, cart unchanged
2. `test_accepted_checkout_returns_before_order_exists()` — call `POST /checkout`, assert response is accepted with checkout id, checkout and outbox exist, no order yet
3. `test_order_created_after_recovery()` — stop order service, call `POST /checkout`, start order service, poll for order, assert order exists with correct data
4. `test_repeated_checkout_does_not_charge_twice()` — call `POST /checkout` twice, assert same checkout id, one charge, one order; publish `OrderRequested` twice, assert one order
5. `test_cart_emptied_and_retried()` — call `POST /checkout`, stop cart service, wait for order, start cart service, poll for cart deletion, assert cart removed
6. `test_charge_refunded_when_order_never_appears()` — stop order service, call `POST /checkout`, wait for completion window (5s), assert refund, no order, checkout status refunded
7. `test_checkout_status_endpoint()` — accept checkout, poll status endpoint, assert status transitions from `accepted` to `order_created` to `cart_cleared`, assert order id present
8. `test_database_isolation()` — accept checkout, assert each service's database contains only its own collections, no cross-database writes

---

## Cross-Story Dependencies

The breakdown ensures providers are ready before consumers:

- **Story 1** must complete before **Story 2** (checkout publishes `OrderRequested` before order consumes it)
- **Story 2** must complete before **Story 3** (order publishes `OrderCreated` before checkout consumes it)
- **Story 1** must complete before **Story 4** (completion window checker inserts `RefundRequested` before payment consumes it)
- **Stories 1, 3, 4** must complete before **Story 5** (checkout status endpoint reads the status written by those stories)
- **Stories 1–5** must complete before **Story 6** (integration tests verify the full flow)

No story depends on a story that comes after it in the breakdown.

---

## Risks

### R1: Completion window too short

**Risk:** Order service is slow; completion window expires before the order is created.  
**Impact:** Checkout refunds the charge; order appears later (orphaned order).  
**Mitigation:** Set completion window to 5 minutes (configurable via `COMPLETION_WINDOW_SECONDS`). Monitor order creation latency. If p99 latency approaches 5 minutes, increase the window.  
**Owned by:** Story 1 (completion window checker), Story 4 (refund logic)

---

### R2: Outbox relay fails to publish a message

**Risk:** Relay crashes or is killed before marking an outbox row as published.  
**Impact:** Message is published twice. Order service creates one order (idempotency). Low risk.  
**Mitigation:** Order service handles duplicates via `payment_reference` (Story 2).  
**Owned by:** Story 1 (relay), Story 2 (idempotency)

---

### R3: Cart delete retry exhausted

**Risk:** Cart service is down for >1 minute; all retries fail.  
**Impact:** Order exists, cart not deleted. Shopper sees a non-empty cart after checkout.  
**Mitigation:** Log a warning. Manual intervention (ops deletes the cart). Future: add a background job to retry cart deletes indefinitely.  
**Owned by:** Story 3 (cart delete retry)

---

### R4: RabbitMQ is down

**Risk:** Relay cannot publish messages; messages pile up in the outbox.  
**Impact:** Orders are not created until RabbitMQ recovers. Completion window may expire, triggering refunds.  
**Mitigation:** Monitor RabbitMQ health. Set completion window to 5 minutes (longer than typical RabbitMQ downtime). Alert on outbox backlog.  
**Owned by:** Story 1 (relay, outbox), Story 4 (refund on timeout)

---

### R5: MongoDB transaction support

**Risk:** Checkout and outbox insert is not atomic if MongoDB does not support transactions.  
**Impact:** Accepted charge with no message to send, or message with no checkout record.  
**Mitigation:** Insert checkout first, then outbox; on outbox failure, delete the checkout record (rollback). Log the failure.  
**Owned by:** Story 1 (checkout atomic write)

---

## Open Questions

1. **MongoDB transaction support:** Does the existing `pymongo` setup support transactions (requires replica set or sharded cluster)? If not, the checkout + outbox insert is not atomic, and rollback is manual.
2. **Relay and completion checker deployment:** How are these processes deployed (systemd, Docker, Kubernetes)? Are they in the same repo or separate? Add startup scripts to `checkout/` or root?
3. **Completion window duration:** Default is 5 minutes. Should this be configurable per checkout, or is a global environment variable acceptable?
4. **RefundCompleted event:** Should payment service publish `RefundCompleted`, or should the completion window checker poll payment status to update checkout to `refunded`?
5. **Cart delete retry limit:** Current limit is 3 retries over 1 minute. Should this be configurable, or should retries be infinite?

---

## End of Breakdown
