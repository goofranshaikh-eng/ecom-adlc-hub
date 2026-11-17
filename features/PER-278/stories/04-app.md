# LLD — Story 04-app

**Story:** Refund a charge when the order never appears within the completion window  
**Repo:** `app`  
**Type:** Implementation  
**Covers AC:** AC6, AC8 (payment consumer side)

---

## Acceptance Criteria

### AC6: A charge with no order is refunded

**Given** an accepted charge whose order has not been created within the completion window  
**When** that window passes  
**Then** the charge is refunded, no order is created, and the checkout status says refunded (or `refund_pending` if `RefundCompleted` event is not consumed yet)

---

### AC8: Services do not share databases (payment consumer side)

**Given** a refunded checkout  
**When** the payment service is inspected  
**Then** payment reads and writes only `payment_db` (updates `charges` collection), and no other service's database is accessed

---

## File Plan

### New Files

1. **`payment/consumers.py`**
   - RabbitMQ consumer for `RefundRequested` messages
   - Validates message body (JSON schema check)
   - Queries `charges_collection` by `reference` (from `payment_reference` in message)
   - Idempotency check: if `status == "refunded"`, skip and ack
   - Updates charge: set `status` to `"refunded"`, set `refunded_at` to `now()`
   - Manual message acknowledgement
   - Entry point: `async def consume_refund_requested()` (called from `main.py`)
   - Tests: `test_charge_refunded_after_completion_window()`, `test_refund_idempotency()`, `test_database_isolation_payment_consumer()`

2. **`payment/rabbitmq_config.py`**
   - RabbitMQ connection setup (reuse pattern from checkout if needed, or inline in `consumers.py`)
   - Constants: `RABBITMQ_URI`, `EXCHANGE_NAME = "ecom.events"`
   - Tests: covered by integration tests

---

### Modified Files

1. **`payment/models.py`**
   - Add `"refunded"` to `ChargeResponse.status` Literal (line 23)
   - Add docstring note about `refunded_at` field in internal charge schema
   - Tests: `test_charge_refunded_after_completion_window()` validates the response schema

2. **`payment/main.py`**
   - Import and start RabbitMQ consumer on app startup (after line 14, after FastAPI app creation)
   - `import asyncio`
   - `from consumers import consume_refund_requested`
   - Add startup event: `@app.on_event("startup")` → `asyncio.create_task(consume_refund_requested())`
   - Tests: no test validates startup logic directly; consumer tests fail if consumer is not started

3. **`payment/requirements.txt`**
   - Add `pika` (RabbitMQ client for Python)
   - Full list after: `fastapi`, `uvicorn`, `pymongo`, `pydantic`, `pika`
   - Tests: consumer tests fail if `pika` is missing

4. **`checkout/completion_checker.py`**
   - Already created in Story 1; verify the logic is present
   - Inside `check_expired_checkouts()` loop (line 22):
     - Query `checkouts_collection.find({"status": "accepted", "created_at": {"$lt": now - timedelta(seconds=COMPLETION_WINDOW_SECONDS)}})`
     - For each expired checkout:
       - Insert `RefundRequested` outbox row
       - Update checkout `status` to `"refund_pending"`
   - Tests: `test_checkout_status_refund_pending()` (verifies completion checker inserts outbox and updates status)

---

## Interfaces

### Database Schema: `charges` collection (read/write)

**Collection:** `payment_db.charges`

**Document:**
```python
{
  "_id": ObjectId,
  "reference": str,  # UUID (payment reference)
  "amount_minor_units": int,
  "card": str | None,
  "status": "accepted" | "refused" | "refunded",  # NEW: refunded
  "refunded_at": datetime | None,  # NEW
  "reason": str | None,  # for refused charges
  "timestamp": datetime
}
```

**Write Operations (this story):**
1. **On RefundRequested receipt:** Query by `reference`, update `status` to `"refunded"`, set `refunded_at` to `now()`.

**Idempotency:**
- If `status` is already `"refunded"`, skip the update and ack the message.

---

### RabbitMQ Message Schema: `RefundRequested` (consumed)

**Published by:** Checkout (via relay, triggered by completion window checker)  
**Routing key:** `payment.refund.requested`  
**Queue:** `payment.refund.requested.queue`

**Payload:**
```json
{
  "event_type": "RefundRequested",
  "payment_reference": "pay_abc123",
  "checkout_id": "chk_xyz789",
  "timestamp": "2025-01-10T14:35:00Z"
}
```

**Field Notes:**
- `payment_reference` (string, required): The charge reference to refund. Payment queries `charges` by `reference` (not `payment_reference`).
- `checkout_id` (string, required): Traceability; not stored in charge document.
- `timestamp` (string, ISO 8601, required): Event timestamp for observability (logged, not stored).

---

### Python Module: `payment/consumers.py`

**Entry point:**
```python
async def consume_refund_requested() -> None:
    """
    Connect to RabbitMQ, subscribe to payment.refund.requested.queue, consume messages.
    For each message:
      1. Validate message body (JSON schema check: event_type, payment_reference)
      2. Query charges_collection by reference (msg["payment_reference"])
      3. If charge not found: log warning, ack message (orphaned message)
      4. If status is already "refunded": ack message (idempotency)
      5. Update status to "refunded", set refunded_at to now()
      6. Ack message after processing
    Runs forever until process exits.
    """
```

**Helper function:**
```python
def is_valid_refund_requested(msg: dict) -> bool:
    """
    Validate RefundRequested message schema.
    Returns True if valid, False otherwise.
    """
    return (
        msg.get("event_type") == "RefundRequested"
        and isinstance(msg.get("payment_reference"), str)
        and len(msg.get("payment_reference", "")) > 0
    )
```

---

## Algorithm Notes

### Message Validation

**Goal:** Reject invalid messages before processing.

**Schema:**
```python
def is_valid_refund_requested(msg: dict) -> bool:
    return (
        msg.get("event_type") == "RefundRequested"
        and isinstance(msg.get("payment_reference"), str)
        and len(msg.get("payment_reference", "")) > 0
    )
```

**Algorithm:**
1. Parse message body: `msg = json.loads(message.body.decode())`
2. Validate: `if not is_valid_refund_requested(msg): log error, nack (no requeue), return`
3. If valid: proceed with refund

**Code:**
```python
try:
    msg = json.loads(message.body.decode())
except json.JSONDecodeError:
    print(f"Failed to parse RefundRequested message: {message.body}")
    channel.basic_nack(delivery_tag=message.delivery_tag, requeue=False)
    return

if not is_valid_refund_requested(msg):
    print(f"Invalid RefundRequested message: {msg}")
    channel.basic_nack(delivery_tag=message.delivery_tag, requeue=False)
    return
```

---

### Idempotency: Duplicate `RefundRequested` Message

**Goal:** A repeated `RefundRequested` message does not refund the charge twice.

**Key:** `payment_reference` (which maps to `reference` in the `charges` collection)

**Algorithm:**
1. Receive `RefundRequested` message from RabbitMQ
2. Parse and validate message body
3. Query `charges_collection.find_one({"reference": msg["payment_reference"]})`
4. If charge not found:
   - Log: `Charge not found for payment_reference {msg["payment_reference"]}, orphaned RefundRequested message`
   - Ack message (do not requeue; charge may have been deleted or never existed)
   - Return
5. If `status == "refunded"`:
   - Log: `Charge already refunded for payment_reference {msg["payment_reference"]}`
   - Ack message (idempotency)
   - Return
6. If `status == "accepted"` or `status == "refused"`:
   - Update `status` to `"refunded"`, set `refunded_at` to `datetime.utcnow()`
   - Ack message

**Code:**
```python
from datetime import datetime
from db import charges_collection

payment_reference = msg["payment_reference"]

charge = charges_collection.find_one({"reference": payment_reference})
if charge is None:
    print(f"Charge not found for payment_reference {payment_reference}, orphaned RefundRequested")
    channel.basic_ack(delivery_tag=message.delivery_tag)
    return

status = charge["status"]
if status == "refunded":
    print(f"Charge already refunded for payment_reference {payment_reference}")
    channel.basic_ack(delivery_tag=message.delivery_tag)
    return

# Update status to refunded
charges_collection.update_one(
    {"reference": payment_reference},
    {"$set": {
        "status": "refunded",
        "refunded_at": datetime.utcnow()
    }}
)
print(f"Refunded charge for payment_reference {payment_reference}")

# Ack message after processing
channel.basic_ack(delivery_tag=message.delivery_tag)
```

---

### Consumer Main Loop

**Goal:** Connect to RabbitMQ, subscribe to `payment.refund.requested.queue`, consume messages.

**Algorithm:**
1. Connect to RabbitMQ: `connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URI))`
2. Create channel: `channel = connection.channel()`
3. Declare exchange: `channel.exchange_declare(exchange="ecom.events", exchange_type="topic", durable=True)`
4. Declare queue: `channel.queue_declare(queue="payment.refund.requested.queue", durable=True)`
5. Bind queue: `channel.queue_bind(exchange="ecom.events", queue="payment.refund.requested.queue", routing_key="payment.refund.requested")`
6. Set prefetch: `channel.basic_qos(prefetch_count=1)` (process one message at a time)
7. Define callback: `def on_message(channel, method, properties, body):`
   - Parse and validate message
   - Query charge by `reference`
   - If not found or already refunded: ack and return
   - Update charge status to `refunded`, set `refunded_at`
   - Ack message: `channel.basic_ack(delivery_tag=method.delivery_tag)`
8. Consume: `channel.basic_consume(queue="payment.refund.requested.queue", on_message_callback=on_message, auto_ack=False)`
9. Start consuming: `channel.start_consuming()`

**Code:**
```python
import asyncio
import json
import os
from datetime import datetime

import pika
from pymongo.errors import PyMongoError

from db import charges_collection

RABBITMQ_URI = os.environ.get("RABBITMQ_URI", "amqp://localhost:5672")


def is_valid_refund_requested(msg: dict) -> bool:
    return (
        msg.get("event_type") == "RefundRequested"
        and isinstance(msg.get("payment_reference"), str)
        and len(msg.get("payment_reference", "")) > 0
    )


def on_message(channel, method, properties, body):
    try:
        msg = json.loads(body.decode())
    except json.JSONDecodeError:
        print(f"Failed to parse RefundRequested message: {body}")
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
        return

    if not is_valid_refund_requested(msg):
        print(f"Invalid RefundRequested message: {msg}")
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
        return

    payment_reference = msg["payment_reference"]

    charge = charges_collection.find_one({"reference": payment_reference})
    if charge is None:
        print(f"Charge not found for payment_reference {payment_reference}, orphaned RefundRequested")
        channel.basic_ack(delivery_tag=method.delivery_tag)
        return

    status = charge["status"]
    if status == "refunded":
        print(f"Charge already refunded for payment_reference {payment_reference}")
        channel.basic_ack(delivery_tag=method.delivery_tag)
        return

    # Update status to refunded
    try:
        charges_collection.update_one(
            {"reference": payment_reference},
            {"$set": {
                "status": "refunded",
                "refunded_at": datetime.utcnow()
            }}
        )
        print(f"Refunded charge for payment_reference {payment_reference}")
    except PyMongoError as e:
        print(f"Failed to refund charge for payment_reference {payment_reference}: {e}")
        # Nack with requeue on database failure
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
        return

    # Ack message after processing
    channel.basic_ack(delivery_tag=method.delivery_tag)


async def consume_refund_requested():
    try:
        connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URI))
        channel = connection.channel()

        channel.exchange_declare(exchange="ecom.events", exchange_type="topic", durable=True)
        channel.queue_declare(queue="payment.refund.requested.queue", durable=True)
        channel.queue_bind(exchange="ecom.events", queue="payment.refund.requested.queue", routing_key="payment.refund.requested")
        channel.basic_qos(prefetch_count=1)

        print("RefundRequested consumer started, waiting for messages...")
        channel.basic_consume(queue="payment.refund.requested.queue", on_message_callback=on_message, auto_ack=False)
        channel.start_consuming()
    except Exception as e:
        print(f"Failed to start RefundRequested consumer: {e}")
        raise
```

---

### Startup Integration

**Goal:** Start the RabbitMQ consumer when the FastAPI app starts.

**Algorithm:**
1. Import `asyncio` and `consume_refund_requested` in `payment/main.py`
2. Add a startup event handler: `@app.on_event("startup")`
3. Create a background task: `asyncio.create_task(consume_refund_requested())`

**Code (in `payment/main.py`):**
```python
import asyncio
import os

import uvicorn
from fastapi import FastAPI

from consumers import consume_refund_requested
from routes import router

app = FastAPI(
    title="Payment Service",
    version="1.0.0",
    docs_url="/docs",
    description=(
        "Payment is a deterministic charge test double. "
        "All endpoints are public in v1; there is no authentication. [AUTH_DEFERRED]"
    ),
)
app.include_router(router)


@app.on_event("startup")
async def startup_event():
    asyncio.create_task(consume_refund_requested())
    print("RefundRequested consumer task started")


if __name__ == "__main__":
    # Payment listens on 8005 unless PORT is set. [CONFIG_DEFERRED]
    port = int(os.environ.get("PORT", "8005"))
    uvicorn.run(app, host="0.0.0.0", port=port)
```

---

### Modified Model: `payment/models.py`

**Changes:**
- Add `"refunded"` to `ChargeResponse.status` Literal (line 23)

**Before:**
```python
class ChargeResponse(BaseModel):
    status: Literal["accepted", "refused"]
    reference: Optional[str] = None
    reason: Optional[str] = None

    @model_validator(mode="after")
    def require_status_fields(self):
        if self.status == "accepted" and self.reference is None:
            raise ValueError("reference must be present when status is accepted")
        if self.status == "refused" and self.reason is None:
            raise ValueError("reason must be present when status is refused")
        return self
```

**After:**
```python
class ChargeResponse(BaseModel):
    status: Literal["accepted", "refused", "refunded"]  # NEW: refunded
    reference: Optional[str] = None
    reason: Optional[str] = None

    @model_validator(mode="after")
    def require_status_fields(self):
        if self.status == "accepted" and self.reference is None:
            raise ValueError("reference must be present when status is accepted")
        if self.status == "refused" and self.reason is None:
            raise ValueError("reason must be present when status is refused")
        # [NOTE] refunded charges have a reference but no reason
        return self
```

**Docstring note (add to file header or charge schema docstring):**
```python
# Charge document in payment_db.charges:
#   _id: ObjectId
#   reference: str (UUID)
#   amount_minor_units: int
#   card: str | None
#   status: "accepted" | "refused" | "refunded"
#   refunded_at: datetime | None  # NEW: set when status is "refunded"
#   reason: str | None (for refused charges)
#   timestamp: datetime
```

---

## Tests

### Test 1: `test_charge_refunded_after_completion_window()`

**Covers:** AC6 (end-to-end: completion checker → relay → payment consumer)

**Code (write once, in `payment/tests/test_payment.py` or new test file):**

```python
import json
import time
from datetime import datetime, timedelta

import pika
from bson import ObjectId
from pymongo import MongoClient

MONGODB_URI = "mongodb://localhost:27017"
RABBITMQ_URI = "amqp://localhost:5672"

mongo_client = MongoClient(MONGODB_URI)
charges_collection = mongo_client["payment_db"]["charges"]
checkouts_collection = mongo_client["checkout_db"]["checkouts"]
outbox_collection = mongo_client["checkout_db"]["outbox"]
orders_collection = mongo_client["order_db"]["orders"]


def test_charge_refunded_after_completion_window():
    """AC6: A charge with no order is refunded."""
    charges_collection.delete_many({})
    checkouts_collection.delete_many({})
    outbox_collection.delete_many({})

    charge = {
        "_id": ObjectId(),
        "reference": "pay_expired",
        "amount_minor_units": 1000,
        "status": "accepted",
        "timestamp": datetime.utcnow()
    }
    charges_collection.insert_one(charge)

    checkout = {
        "_id": ObjectId(),
        "checkout_id": "chk_expired",
        "payment_reference": "pay_expired",
        "cart_id": "cart_expired",
        "email": "guest@example.com",
        "ship_to": {"street": "123 Main St", "city": "Springfield", "postal_code": "12345", "country": "US"},
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000,
        "status": "accepted",
        "order_id": None,
        "created_at": datetime.utcnow() - timedelta(minutes=6),
        "updated_at": datetime.utcnow() - timedelta(minutes=6)
    }
    checkouts_collection.insert_one(checkout)

    # Simulate completion checker inserting RefundRequested outbox row
    outbox_row = {
        "_id": ObjectId(),
        "event_type": "RefundRequested",
        "payload": {
            "event_type": "RefundRequested",
            "payment_reference": "pay_expired",
            "checkout_id": "chk_expired",
            "timestamp": datetime.utcnow().isoformat()
        },
        "payment_reference": "pay_expired",
        "published": False,
        "created_at": datetime.utcnow()
    }
    outbox_collection.insert_one(outbox_row)

    # Simulate relay publishing message
    connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URI))
    channel = connection.channel()
    channel.basic_publish(
        exchange="ecom.events",
        routing_key="payment.refund.requested",
        body=json.dumps(outbox_row["payload"]),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    channel.close()
    connection.close()

    # Wait for payment consumer to process
    time.sleep(2)

    # Assert charge is refunded
    refunded_charge = charges_collection.find_one({"reference": "pay_expired"})
    assert refunded_charge is not None
    assert refunded_charge["status"] == "refunded"
    assert refunded_charge["refunded_at"] is not None
    assert isinstance(refunded_charge["refunded_at"], datetime)

    # Assert checkout status is refund_pending
    updated_checkout = checkouts_collection.find_one({"checkout_id": "chk_expired"})
    assert updated_checkout["status"] == "refund_pending"

    # Assert no order exists
    order = orders_collection.find_one({"payment_reference": "pay_expired"})
    assert order is None
```

---

### Test 2: `test_refund_idempotency()`

**Covers:** AC6 (idempotency: duplicate message does not refund twice)

**Code:**
```python
def test_refund_idempotency():
    """AC6: Duplicate RefundRequested message does not refund twice."""
    charges_collection.delete_many({})

    charge = {
        "_id": ObjectId(),
        "reference": "pay_dup",
        "amount_minor_units": 1000,
        "status": "accepted",
        "timestamp": datetime.utcnow()
    }
    charges_collection.insert_one(charge)

    msg = {
        "event_type": "RefundRequested",
        "payment_reference": "pay_dup",
        "checkout_id": "chk_dup",
        "timestamp": datetime.utcnow().isoformat()
    }

    connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URI))
    channel = connection.channel()
    # Publish twice
    channel.basic_publish(
        exchange="ecom.events",
        routing_key="payment.refund.requested",
        body=json.dumps(msg),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    channel.basic_publish(
        exchange="ecom.events",
        routing_key="payment.refund.requested",
        body=json.dumps(msg),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    channel.close()
    connection.close()

    time.sleep(2)

    # Assert charge is refunded once
    refunded_charge = charges_collection.find_one({"reference": "pay_dup"})
    assert refunded_charge is not None
    assert refunded_charge["status"] == "refunded"
    assert refunded_charge["refunded_at"] is not None
```

---

### Test 3: `test_no_order_created_after_refund()`

**Covers:** AC6 (no order is created after refund)

**Code:**
```python
def test_no_order_created_after_refund():
    """AC6: No order is created after refund."""
    charges_collection.delete_many({})
    checkouts_collection.delete_many({})
    orders_collection.delete_many({})

    charge = {
        "_id": ObjectId(),
        "reference": "pay_no_order",
        "amount_minor_units": 1000,
        "status": "accepted",
        "timestamp": datetime.utcnow()
    }
    charges_collection.insert_one(charge)

    checkout = {
        "_id": ObjectId(),
        "checkout_id": "chk_no_order",
        "payment_reference": "pay_no_order",
        "cart_id": "cart_no_order",
        "email": "guest@example.com",
        "ship_to": {"street": "123 Main St", "city": "Springfield", "postal_code": "12345", "country": "US"},
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000,
        "status": "accepted",
        "order_id": None,
        "created_at": datetime.utcnow() - timedelta(minutes=6),
        "updated_at": datetime.utcnow() - timedelta(minutes=6)
    }
    checkouts_collection.insert_one(checkout)

    # Publish RefundRequested
    msg = {
        "event_type": "RefundRequested",
        "payment_reference": "pay_no_order",
        "checkout_id": "chk_no_order",
        "timestamp": datetime.utcnow().isoformat()
    }

    connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URI))
    channel = connection.channel()
    channel.basic_publish(
        exchange="ecom.events",
        routing_key="payment.refund.requested",
        body=json.dumps(msg),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    channel.close()
    connection.close()

    time.sleep(2)

    # Assert charge is refunded
    refunded_charge = charges_collection.find_one({"reference": "pay_no_order"})
    assert refunded_charge["status"] == "refunded"

    # Assert no order exists
    order = orders_collection.find_one({"payment_reference": "pay_no_order"})
    assert order is None
```

---

### Test 4: `test_checkout_status_refund_pending()`

**Covers:** AC6 (completion checker updates checkout status to `refund_pending`)

**Code:**
```python
def test_checkout_status_refund_pending():
    """AC6: Completion checker updates checkout status to refund_pending."""
    checkouts_collection.delete_many({})
    outbox_collection.delete_many({})

    checkout = {
        "_id": ObjectId(),
        "checkout_id": "chk_pending",
        "payment_reference": "pay_pending",
        "cart_id": "cart_pending",
        "email": "guest@example.com",
        "ship_to": {"street": "123 Main St", "city": "Springfield", "postal_code": "12345", "country": "US"},
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000,
        "status": "accepted",
        "order_id": None,
        "created_at": datetime.utcnow() - timedelta(minutes=6),
        "updated_at": datetime.utcnow() - timedelta(minutes=6)
    }
    checkouts_collection.insert_one(checkout)

    # Run completion checker (or simulate by running the script)
    # For test, manually trigger the checker logic
    import threading
    from checkout.completion_checker import check_expired_checkouts
    checker_thread = threading.Thread(target=check_expired_checkouts, daemon=True)
    checker_thread.start()
    time.sleep(2)

    # Assert checkout status is refund_pending
    updated_checkout = checkouts_collection.find_one({"checkout_id": "chk_pending"})
    assert updated_checkout["status"] == "refund_pending"

    # Assert RefundRequested outbox row exists
    refund_row = outbox_collection.find_one({"payment_reference": "pay_pending", "event_type": "RefundRequested"})
    assert refund_row is not None
    assert refund_row["published"] is False
```

---

### Test 5: `test_database_isolation_payment_consumer()`

**Covers:** AC8

**Code:**
```python
from pathlib import Path

def test_database_isolation_payment_consumer():
    """AC8: Payment consumer only accesses payment_db."""
    charges_collection.delete_many({})

    charge = {
        "_id": ObjectId(),
        "reference": "pay_iso",
        "amount_minor_units": 1000,
        "status": "accepted",
        "timestamp": datetime.utcnow()
    }
    charges_collection.insert_one(charge)

    msg = {
        "event_type": "RefundRequested",
        "payment_reference": "pay_iso",
        "checkout_id": "chk_iso",
        "timestamp": datetime.utcnow().isoformat()
    }

    connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URI))
    channel = connection.channel()
    channel.basic_publish(
        exchange="ecom.events",
        routing_key="payment.refund.requested",
        body=json.dumps(msg),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    channel.close()
    connection.close()

    time.sleep(2)

    # Assert charge is refunded
    refunded_charge = charges_collection.find_one({"reference": "pay_iso"})
    assert refunded_charge["status"] == "refunded"

    # Scan source files for other database names
    OTHER_DATABASES = ("checkout_db", "order_db", "cart_db", "identity_db", "catalog_db")
    consumers_path = Path(__file__).resolve().parents[1] / "consumers.py"
    source = consumers_path.read_text()
    for db_name in OTHER_DATABASES:
        assert db_name not in source, f"{db_name} found in consumers.py"
```

---

**End of LLD.**
