# LLD — Story 03-app

**Story:** Checkout empties the cart on OrderCreated with retry  
**Repo:** `app`  
**Type:** Implementation  
**Covers AC:** AC5, AC8

---

## Acceptance Criteria

### AC5: The cart is emptied, and a failed delete is retried

**Given** an order created for a checkout  
**When** deleting the cart succeeds (or fails but later succeeds on retry)  
**Then** the cart is removed and the checkout status is `cart_cleared`, and the order is not created again

---

### AC8: Services do not share databases (checkout consumer side)

**Given** a completed checkout  
**When** the checkout consumer is inspected  
**Then** checkout reads and writes only `checkout_db` (updates `checkouts` collection), and no other service's database is accessed

---

## File Plan

### New Files

1. **`checkout/consumers.py`**
   - RabbitMQ consumer for `OrderCreated` messages
   - Validates message body (JSON schema check)
   - Queries `checkouts_collection` by `payment_reference` for idempotency
   - Updates checkout status to `order_created`, sets `order_id`
   - Attempts cart deletion with exponential backoff retry (5s, 10s, 20s, max 3 retries)
   - Updates checkout status to `cart_cleared` on success
   - Manual message acknowledgement
   - Logs cart delete failures
   - Tests: `test_cart_deleted_on_order_created()`, `test_cart_delete_retried_on_failure()`, `test_cart_delete_retry_exhausted()`, `test_duplicate_order_created_message()`, `test_database_isolation_checkout_consumer()`

2. **`checkout/rabbitmq_config.py`** (if not already created in Story 1)
   - RabbitMQ connection setup (reuse from Story 1 if exists)
   - Constants: `RABBITMQ_URI`, `EXCHANGE_NAME = "ecom.events"`
   - Tests: covered by integration tests

---

### Modified Files

1. **`checkout/main.py`**
   - Import and start RabbitMQ consumer on app startup (after line 14, after FastAPI app creation)
   - `import asyncio`
   - `from consumers import consume_order_created`
   - Add startup event: `@app.on_event("startup")` → `asyncio.create_task(consume_order_created())`
   - Tests: no test validates startup logic directly; consumer tests fail if consumer is not started

2. **`checkout/db.py`**
   - Add `checkouts_collection = db["checkouts"]` (line 15, after `db = client["checkout_db"]`)
   - Tests: `test_cart_deleted_on_order_created()`, `test_database_isolation_checkout_consumer()`

3. **`checkout/models.py`**
   - No changes (checkout status field already exists per Story 1 schema)
   - Tests: no new tests for models

4. **`checkout/requirements.txt`**
   - Add `pika` (RabbitMQ client for Python) if not already present from Story 1
   - Full list after: `fastapi`, `uvicorn`, `pymongo`, `pydantic`, `httpx`, `email-validator`, `pika`
   - Tests: consumer tests fail if `pika` is missing

---

## Interfaces

### Database Schema: `checkouts` collection (read/write)

**Collection:** `checkout_db.checkouts`

**Document:**
```python
{
  "_id": ObjectId,
  "checkout_id": str,
  "payment_reference": str,
  "cart_id": str,
  "shopper_id": str | None,
  "email": str | None,
  "ship_to": {
    "street": str,
    "city": str,
    "postal_code": str,
    "country": str
  },
  "lines": [
    {
      "product_id": str,
      "quantity": int,
      "price_snapshot_minor_units": int
    }
  ],
  "total_minor_units": int,
  "status": "accepted" | "order_created" | "cart_cleared" | "refunded",
  "order_id": str | None,
  "created_at": datetime,
  "updated_at": datetime
}
```

**Write Operations (this story):**
1. **On OrderCreated receipt:** Query by `payment_reference`, update `status` to `"order_created"`, set `order_id`, update `updated_at`.
2. **On cart delete success:** Update by `payment_reference`, set `status` to `"cart_cleared"`, update `updated_at`.

**Idempotency:**
- If `status` is already `"order_created"` or `"cart_cleared"` when `OrderCreated` arrives, skip the update and ack the message.

---

### RabbitMQ Message Schema: `OrderCreated` (consumed)

**Published by:** Order service  
**Routing key:** `order.created`  
**Queue:** `order.created.queue`

**Payload:**
```json
{
  "event_type": "OrderCreated",
  "payment_reference": "pay_abc123",
  "checkout_id": "chk_xyz789",
  "order_id": "507f1f77bcf86cd799439011",
  "timestamp": "2025-01-10T14:30:05Z"
}
```

**Field Notes:**
- `payment_reference` (string, required): Idempotency key. Checkout queries `checkouts` by this field.
- `checkout_id` (string, required): Traceability; not stored in checkout document (checkout already has its own `checkout_id`).
- `order_id` (string, required): MongoDB ObjectId as hex string. Stored in checkout document.
- `timestamp` (string, ISO 8601, required): Event timestamp for observability (logged, not stored).

---

### HTTP API: Cart Service (called by checkout)

**Endpoint:** `DELETE /cart/{cart_id}`  
**Service:** Cart (port 8003)  
**URL:** `http://localhost:8003/cart/{cart_id}` (verify in `checkout/routes.py:30` — `CART_SERVICE_URL`)  
**Expected Response:** `204 No Content`  
**Failure Handling:** Network error, 500, 503, or timeout triggers retry (exponential backoff: 5s, 10s, 20s, max 3 retries).

---

### Python Module: `checkout/consumers.py`

**Entry point:**
```python
async def consume_order_created() -> None:
    """
    Connect to RabbitMQ, subscribe to order.created.queue, consume messages.
    For each message:
      1. Validate message body (JSON schema check: event_type, payment_reference, order_id)
      2. Query checkouts_collection by payment_reference
      3. If checkout not found: log warning, ack message (orphaned message)
      4. If status is already order_created or cart_cleared: ack message (idempotency)
      5. Update status to order_created, set order_id, updated_at
      6. Attempt cart delete with retry (delete_cart_with_retry)
      7. On success: update status to cart_cleared, updated_at
      8. On failure after retries: log warning, leave status as order_created
      9. Ack message after processing (do not hold message on retry delays)
    Runs forever until process exits.
    """
```

**Helper function:**
```python
async def delete_cart_with_retry(
    cart_id: str,
    max_retries: int = 3,
    delays: list[int] = [5, 10, 20]
) -> bool:
    """
    Attempt to delete a cart with exponential backoff retry.
    
    Args:
        cart_id: Cart id to delete
        max_retries: Maximum number of retries (default 3)
        delays: Retry delays in seconds (default [5, 10, 20])
    
    Returns:
        True if deletion succeeded, False if all retries failed
    
    Raises:
        No exceptions (all failures are caught and logged)
    
    Algorithm:
        1. For retry_count in range(max_retries + 1):
            a. Try DELETE /cart/{cart_id} with httpx.AsyncClient
            b. If 204: return True
            c. If network error or 500/503: log failure, wait delays[retry_count] if not last retry
        2. Return False (all retries failed)
    """
```

---

## Algorithm Notes

### Message Validation

**Goal:** Reject invalid messages before processing.

**Schema:**
```python
# Minimal validation (not Joi; Python has no Joi; use dict checks)
def is_valid_order_created(msg: dict) -> bool:
    return (
        msg.get("event_type") == "OrderCreated"
        and isinstance(msg.get("payment_reference"), str)
        and len(msg.get("payment_reference", "")) > 0
        and isinstance(msg.get("order_id"), str)
        and len(msg.get("order_id", "")) > 0
    )
```

**Algorithm:**
1. Parse message body: `msg = json.loads(message.body.decode())`
2. Validate: `if not is_valid_order_created(msg): log error, nack (no requeue), return`
3. If valid: proceed with checkout update

**Code:**
```python
try:
    msg = json.loads(message.body.decode())
except json.JSONDecodeError:
    print(f"Failed to parse OrderCreated message: {message.body}")
    channel.basic_nack(delivery_tag=message.delivery_tag, requeue=False)
    return

if not is_valid_order_created(msg):
    print(f"Invalid OrderCreated message: {msg}")
    channel.basic_nack(delivery_tag=message.delivery_tag, requeue=False)
    return
```

---

### Idempotency: Duplicate `OrderCreated` Message

**Goal:** A repeated `OrderCreated` message does not update the checkout twice or attempt cart deletion twice.

**Key:** `payment_reference`

**Algorithm:**
1. Receive `OrderCreated` message from RabbitMQ
2. Parse and validate message body
3. Query `checkouts_collection.find_one({"payment_reference": msg["payment_reference"]})`
4. If checkout not found:
   - Log: `Checkout not found for payment_reference {msg["payment_reference"]}, orphaned OrderCreated message`
   - Ack message (do not requeue; checkout service may have restarted and lost the checkout)
   - Return
5. If `status` is already `"order_created"` or `"cart_cleared"`:
   - Log: `OrderCreated already processed for payment_reference {msg["payment_reference"]}, status={status}`
   - Ack message (idempotency)
   - Return
6. If `status == "accepted"`:
   - Update `status` to `"order_created"`, set `order_id`, update `updated_at`
   - Proceed with cart deletion
7. If `status == "refunded"`:
   - Log: `OrderCreated received for refunded checkout, payment_reference {msg["payment_reference"]}`
   - Ack message (order should not have been created after refund; log for observability)
   - Return

**Code:**
```python
checkout = checkouts_collection.find_one({"payment_reference": payment_reference})
if checkout is None:
    print(f"Checkout not found for payment_reference {payment_reference}, orphaned OrderCreated")
    channel.basic_ack(delivery_tag=message.delivery_tag)
    return

status = checkout["status"]
if status in ["order_created", "cart_cleared"]:
    print(f"OrderCreated already processed for payment_reference {payment_reference}, status={status}")
    channel.basic_ack(delivery_tag=message.delivery_tag)
    return

if status == "refunded":
    print(f"OrderCreated received for refunded checkout, payment_reference {payment_reference}")
    channel.basic_ack(delivery_tag=message.delivery_tag)
    return

# Update status to order_created
checkouts_collection.update_one(
    {"payment_reference": payment_reference},
    {"$set": {
        "status": "order_created",
        "order_id": msg["order_id"],
        "updated_at": datetime.utcnow()
    }}
)
print(f"Updated checkout to order_created for payment_reference {payment_reference}")
```

---

### Cart Deletion with Retry

**Goal:** Delete the cart after the order is created, retrying on transient failures.

**Algorithm:**
1. For `retry_count` in `range(max_retries + 1)` (0, 1, 2, 3):
   - `cart_url = f"{CART_SERVICE_URL}/cart/{cart_id}"`
   - Try `httpx.AsyncClient().delete(cart_url, timeout=5.0)`
   - If response status is 204:
     - Return `True` (success)
   - If response status is 404:
     - Log: `Cart {cart_id} not found (already deleted or never existed)`
     - Return `True` (cart is gone; treat as success)
   - If response status is 500, 503, or network error:
     - Log: `Cart delete failed (attempt {retry_count + 1}/{max_retries + 1}): {error}`
     - If `retry_count < max_retries`:
       - `await asyncio.sleep(delays[retry_count])`
     - Continue to next retry
2. Return `False` (all retries failed)

**Code:**
```python
async def delete_cart_with_retry(
    cart_id: str,
    max_retries: int = 3,
    delays: list[int] = [5, 10, 20]
) -> bool:
    cart_url = f"{CART_SERVICE_URL}/cart/{cart_id}"
    
    for retry_count in range(max_retries + 1):
        try:
            async with httpx.AsyncClient(timeout=5.0) as client:
                response = await client.delete(cart_url)
            
            if response.status_code == 204:
                print(f"Cart {cart_id} deleted successfully")
                return True
            
            if response.status_code == 404:
                print(f"Cart {cart_id} not found (already deleted or never existed)")
                return True
            
            # 500, 503, or other failure
            print(f"Cart delete failed (attempt {retry_count + 1}/{max_retries + 1}): status {response.status_code}")
        
        except httpx.RequestError as e:
            print(f"Cart delete network error (attempt {retry_count + 1}/{max_retries + 1}): {e}")
        
        # Wait before retry (skip on last attempt)
        if retry_count < max_retries:
            await asyncio.sleep(delays[retry_count])
    
    # All retries failed
    print(f"Cart delete exhausted all retries for cart {cart_id}")
    return False
```

---

### Consumer Main Loop

**Goal:** Connect to RabbitMQ, subscribe to `order.created.queue`, consume messages.

**Algorithm:**
1. Connect to RabbitMQ: `connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URI))`
2. Create channel: `channel = connection.channel()`
3. Declare exchange: `channel.exchange_declare(exchange="ecom.events", exchange_type="topic", durable=True)`
4. Declare queue: `channel.queue_declare(queue="order.created.queue", durable=True)`
5. Bind queue: `channel.queue_bind(exchange="ecom.events", queue="order.created.queue", routing_key="order.created")`
6. Set prefetch: `channel.basic_qos(prefetch_count=1)` (process one message at a time)
7. Define callback: `def on_message(channel, method, properties, body):`
   - Parse and validate message
   - Query checkout by `payment_reference`
   - If not found or already processed: ack and return
   - Update checkout status to `order_created`, set `order_id`
   - Attempt cart delete with retry
   - On success: update status to `cart_cleared`
   - On failure: log warning, leave status as `order_created`
   - Ack message: `channel.basic_ack(delivery_tag=method.delivery_tag)`
8. Consume: `channel.basic_consume(queue="order.created.queue", on_message_callback=on_message, auto_ack=False)`
9. Start consuming: `channel.start_consuming()`

**Code:**
```python
import asyncio
import json
import os
from datetime import datetime

import pika
import httpx
from bson import ObjectId

from db import checkouts_collection

RABBITMQ_URI = os.environ.get("RABBITMQ_URI", "amqp://localhost:5672")
CART_SERVICE_URL = "http://localhost:8003"


def is_valid_order_created(msg: dict) -> bool:
    return (
        msg.get("event_type") == "OrderCreated"
        and isinstance(msg.get("payment_reference"), str)
        and len(msg.get("payment_reference", "")) > 0
        and isinstance(msg.get("order_id"), str)
        and len(msg.get("order_id", "")) > 0
    )


async def delete_cart_with_retry(
    cart_id: str,
    max_retries: int = 3,
    delays: list[int] = [5, 10, 20]
) -> bool:
    cart_url = f"{CART_SERVICE_URL}/cart/{cart_id}"
    
    for retry_count in range(max_retries + 1):
        try:
            async with httpx.AsyncClient(timeout=5.0) as client:
                response = await client.delete(cart_url)
            
            if response.status_code == 204:
                print(f"Cart {cart_id} deleted successfully")
                return True
            
            if response.status_code == 404:
                print(f"Cart {cart_id} not found (already deleted or never existed)")
                return True
            
            print(f"Cart delete failed (attempt {retry_count + 1}/{max_retries + 1}): status {response.status_code}")
        
        except httpx.RequestError as e:
            print(f"Cart delete network error (attempt {retry_count + 1}/{max_retries + 1}): {e}")
        
        if retry_count < max_retries:
            await asyncio.sleep(delays[retry_count])
    
    print(f"Cart delete exhausted all retries for cart {cart_id}")
    return False


def on_message(channel, method, properties, body):
    try:
        msg = json.loads(body.decode())
    except json.JSONDecodeError:
        print(f"Failed to parse OrderCreated message: {body}")
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
        return

    if not is_valid_order_created(msg):
        print(f"Invalid OrderCreated message: {msg}")
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
        return

    payment_reference = msg["payment_reference"]
    order_id = msg["order_id"]

    checkout = checkouts_collection.find_one({"payment_reference": payment_reference})
    if checkout is None:
        print(f"Checkout not found for payment_reference {payment_reference}, orphaned OrderCreated")
        channel.basic_ack(delivery_tag=method.delivery_tag)
        return

    status = checkout["status"]
    if status in ["order_created", "cart_cleared"]:
        print(f"OrderCreated already processed for payment_reference {payment_reference}, status={status}")
        channel.basic_ack(delivery_tag=method.delivery_tag)
        return

    if status == "refunded":
        print(f"OrderCreated received for refunded checkout, payment_reference {payment_reference}")
        channel.basic_ack(delivery_tag=method.delivery_tag)
        return

    # Update status to order_created
    checkouts_collection.update_one(
        {"payment_reference": payment_reference},
        {"$set": {
            "status": "order_created",
            "order_id": order_id,
            "updated_at": datetime.utcnow()
        }}
    )
    print(f"Updated checkout to order_created for payment_reference {payment_reference}")

    # Attempt cart deletion with retry
    cart_id = checkout["cart_id"]
    # Run async delete in sync context (pika callback is sync)
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    success = loop.run_until_complete(delete_cart_with_retry(cart_id))
    loop.close()

    if success:
        checkouts_collection.update_one(
            {"payment_reference": payment_reference},
            {"$set": {
                "status": "cart_cleared",
                "updated_at": datetime.utcnow()
            }}
        )
        print(f"Updated checkout to cart_cleared for payment_reference {payment_reference}")
    else:
        print(f"[KNOWN_GAP] Cart delete failed for cart {cart_id}, checkout status remains order_created")

    # Ack message after processing
    channel.basic_ack(delivery_tag=method.delivery_tag)


async def consume_order_created():
    try:
        connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URI))
        channel = connection.channel()

        channel.exchange_declare(exchange="ecom.events", exchange_type="topic", durable=True)
        channel.queue_declare(queue="order.created.queue", durable=True)
        channel.queue_bind(exchange="ecom.events", queue="order.created.queue", routing_key="order.created")
        channel.basic_qos(prefetch_count=1)

        print("OrderCreated consumer started, waiting for messages...")
        channel.basic_consume(queue="order.created.queue", on_message_callback=on_message, auto_ack=False)
        channel.start_consuming()
    except Exception as e:
        print(f"Failed to start OrderCreated consumer: {e}")
        raise
```

---

### Startup Integration

**Goal:** Start the RabbitMQ consumer when the FastAPI app starts.

**Algorithm:**
1. Import `asyncio` and `consume_order_created` in `checkout/main.py`
2. Add a startup event handler: `@app.on_event("startup")`
3. Create a background task: `asyncio.create_task(consume_order_created())`

**Code (in `checkout/main.py`):**
```python
import asyncio
import os

import uvicorn
from fastapi import FastAPI

import db
from consumers import consume_order_created
from routes import router

app = FastAPI(
    title="Checkout Service",
    version="1.0.0",
    docs_url="/docs",
    description=(
        "Checkout orchestrates cart, payment, and order over HTTP. "
        "Consumes OrderCreated from RabbitMQ to empty carts. "
        "Public, no auth. [AUTH_DEFERRED]"
    ),
)
app.state.db = db.db
app.include_router(router)


@app.on_event("startup")
async def startup_event():
    asyncio.create_task(consume_order_created())
    print("OrderCreated consumer task started")


if __name__ == "__main__":
    port = int(os.environ.get("PORT", "8004"))
    uvicorn.run(app, host="0.0.0.0", port=port)
```

---

## Tests

### Test 1: `test_cart_deleted_on_order_created()`

**Covers:** AC5 (happy path)

**Code (write once, in `checkout/tests/test_distributed_checkout.py` or similar test file):**

```python
import json
import time
from datetime import datetime
from pathlib import Path

import pika
from bson import ObjectId
from pymongo import MongoClient

MONGODB_URI = "mongodb://localhost:27017"
RABBITMQ_URI = "amqp://localhost:5672"

# Reference to collections (adjust imports as needed for your test setup)
mongo_client = MongoClient(MONGODB_URI)
checkouts_collection = mongo_client["checkout_db"]["checkouts"]
carts_collection = mongo_client["cart_db"]["carts"]


def test_cart_deleted_on_order_created():
    """AC5: Cart is deleted after OrderCreated is received (happy path)."""
    checkouts_collection.delete_many({})
    carts_collection.delete_many({})

    checkout = {
        "_id": ObjectId(),
        "checkout_id": "chk_test_1",
        "payment_reference": "pay_test_1",
        "cart_id": "cart_test_1",
        "email": "guest@example.com",
        "ship_to": {"street": "123 Main St", "city": "Springfield", "postal_code": "12345", "country": "US"},
        "lines": [{"product_id": "prod_001", "quantity": 2, "price_snapshot_minor_units": 1500}],
        "total_minor_units": 3000,
        "status": "accepted",
        "order_id": None,
        "created_at": datetime.utcnow(),
        "updated_at": datetime.utcnow()
    }
    checkouts_collection.insert_one(checkout)

    cart = {
        "_id": "cart_test_1",
        "lines": [{"product_id": "prod_001", "quantity": 2, "price_snapshot_minor_units": 1500}],
        "total_minor_units": 3000
    }
    carts_collection.insert_one(cart)

    msg = {
        "event_type": "OrderCreated",
        "payment_reference": "pay_test_1",
        "checkout_id": "chk_test_1",
        "order_id": "ord_test_1",
        "timestamp": datetime.utcnow().isoformat()
    }

    connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URI))
    channel = connection.channel()
    channel.basic_publish(
        exchange="ecom.events",
        routing_key="order.created",
        body=json.dumps(msg),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    channel.close()
    connection.close()

    time.sleep(2)

    updated_checkout = checkouts_collection.find_one({"payment_reference": "pay_test_1"})
    assert updated_checkout is not None
    assert updated_checkout["status"] == "cart_cleared"
    assert updated_checkout["order_id"] == "ord_test_1"

    cart_after = carts_collection.find_one({"_id": "cart_test_1"})
    assert cart_after is None
```

---

### Test 2: `test_cart_delete_retried_on_failure()`

**Covers:** AC5 (retry path)

**Note:** This test requires mocking the cart service or using a test HTTP server to simulate failures. For the LLD artifact, we document the test logic and defer full implementation to the integration test suite.

```python
def test_cart_delete_retried_on_failure():
    """AC5: Cart delete is retried on failure (requires cart service mock)."""
    # Setup: same as Test 1
    # Mock: cart service returns 500 on first DELETE, 204 on second
    # Publish OrderCreated
    # Wait 15 seconds (first attempt fails at 0s, waits 5s, second attempt at 5s succeeds)
    # Assert checkout status is cart_cleared
    # Assert cart is deleted
    pass  # Deferred to integration test suite with mock HTTP server
```

---

### Test 3: `test_cart_delete_retry_exhausted()`

**Covers:** AC5 (retry exhaustion path)

```python
def test_cart_delete_retry_exhausted(capsys):
    """AC5: Cart delete retry exhaustion (requires cart service mock)."""
    # Setup: same as Test 1
    # Mock: cart service returns 500 on all DELETE requests
    # Publish OrderCreated
    # Wait 40 seconds (retries at 0s, 5s, 15s, 35s)
    # Assert checkout status is order_created (not cart_cleared)
    # Assert cart still exists
    # Assert warning is logged
    pass  # Deferred to integration test suite with mock HTTP server
```

---

### Test 4: `test_duplicate_order_created_message()`

**Covers:** AC5 (idempotency)

```python
def test_duplicate_order_created_message():
    """AC5: Duplicate OrderCreated message does not update checkout twice."""
    checkouts_collection.delete_many({})
    carts_collection.delete_many({})

    checkout = {
        "_id": ObjectId(),
        "checkout_id": "chk_dup",
        "payment_reference": "pay_dup",
        "cart_id": "cart_dup",
        "email": "guest@example.com",
        "ship_to": {"street": "123 Main St", "city": "Springfield", "postal_code": "12345", "country": "US"},
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000,
        "status": "accepted",
        "order_id": None,
        "created_at": datetime.utcnow(),
        "updated_at": datetime.utcnow()
    }
    checkouts_collection.insert_one(checkout)

    cart = {
        "_id": "cart_dup",
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000
    }
    carts_collection.insert_one(cart)

    msg = {
        "event_type": "OrderCreated",
        "payment_reference": "pay_dup",
        "checkout_id": "chk_dup",
        "order_id": "ord_dup",
        "timestamp": datetime.utcnow().isoformat()
    }

    connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URI))
    channel = connection.channel()
    # Publish twice
    channel.basic_publish(
        exchange="ecom.events",
        routing_key="order.created",
        body=json.dumps(msg),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    channel.basic_publish(
        exchange="ecom.events",
        routing_key="order.created",
        body=json.dumps(msg),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    channel.close()
    connection.close()

    time.sleep(5)

    updated_checkout = checkouts_collection.find_one({"payment_reference": "pay_dup"})
    assert updated_checkout is not None
    assert updated_checkout["status"] == "cart_cleared"
    assert updated_checkout["order_id"] == "ord_dup"

    cart_after = carts_collection.find_one({"_id": "cart_dup"})
    assert cart_after is None
```

---

### Test 5: `test_database_isolation_checkout_consumer()`

**Covers:** AC8

```python
def test_database_isolation_checkout_consumer():
    """AC8: Checkout consumer only accesses checkout_db."""
    checkouts_collection.delete_many({})
    carts_collection.delete_many({})

    checkout = {
        "_id": ObjectId(),
        "checkout_id": "chk_iso",
        "payment_reference": "pay_iso",
        "cart_id": "cart_iso",
        "email": "guest@example.com",
        "ship_to": {"street": "123 Main St", "city": "Springfield", "postal_code": "12345", "country": "US"},
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000,
        "status": "accepted",
        "order_id": None,
        "created_at": datetime.utcnow(),
        "updated_at": datetime.utcnow()
    }
    checkouts_collection.insert_one(checkout)

    cart = {
        "_id": "cart_iso",
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000
    }
    carts_collection.insert_one(cart)

    msg = {
        "event_type": "OrderCreated",
        "payment_reference": "pay_iso",
        "checkout_id": "chk_iso",
        "order_id": "ord_iso",
        "timestamp": datetime.utcnow().isoformat()
    }

    connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URI))
    channel = connection.channel()
    channel.basic_publish(
        exchange="ecom.events",
        routing_key="order.created",
        body=json.dumps(msg),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    channel.close()
    connection.close()

    time.sleep(2)

    updated_checkout = checkouts_collection.find_one({"payment_reference": "pay_iso"})
    assert updated_checkout is not None
    assert updated_checkout["status"] == "cart_cleared"

    # Scan source files for other database names
    OTHER_DATABASES = ("order_db", "payment_db", "cart_db", "identity_db", "catalog_db")
    consumers_path = Path(__file__).resolve().parents[1] / "consumers.py"
    source = consumers_path.read_text()
    for db_name in OTHER_DATABASES:
        assert db_name not in source, f"{db_name} found in consumers.py"
```

---

**End of LLD.**
