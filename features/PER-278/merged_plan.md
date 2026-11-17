# Merged plan — PER-278

Execution order (6 ticket(s)):

1. `1`
2. `2`
3. `3`
4. `4`
5. `5`
6. `6`

Specs (6): `specs/1.md`, `specs/2.md`, `specs/3.md`, `specs/4.md`, `specs/5.md`, `specs/6.md`

---

## Spec `specs/1.md`

```markdown
# Tech Stub — PER-278-1

**Ticket:** PER-278-1  
**Title:** Checkout outbox and OrderRequested publish via RabbitMQ  
**Feature:** A paid checkout still becomes one order when order or cart is briefly down  
**Initiative:** `epic-distributed-checkout/`  
**Covers AC:** AC2, AC4 (checkout side), AC8

---

## Stack

- **Language:** Python 3.12
- **Framework:** FastAPI (checkout service already uses it)
- **Database:** MongoDB (checkout service already has `checkout_db`)
- **Message Broker:** RabbitMQ (new dependency)
- **RabbitMQ Client:** `pika` (Python AMQP client)
- **Deployment:** Relay and completion checker run as separate Python processes (systemd service, Docker container, or Kubernetes pod per deployment config)

---

## Schema and API Contracts

### Database Schema

**Collection:** `checkouts` (new collection in `checkout_db`)

```python
{
  "_id": ObjectId,
  "checkout_id": str,  # UUID
  "payment_reference": str,  # idempotency key
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

**Collection:** `outbox` (new collection in `checkout_db`)

```python
{
  "_id": ObjectId,
  "event_type": "OrderRequested" | "RefundRequested",
  "payload": dict,  # message body
  "payment_reference": str,  # for idempotency
  "published": bool,
  "published_at": datetime | None,
  "created_at": datetime
}
```

---

### RabbitMQ Message Schema

**Message:** `OrderRequested`  
**Routing Key:** `order.requested`  
**Exchange:** `ecom.events` (type: `topic`, durable)

```json
{
  "event_type": "OrderRequested",
  "payment_reference": "pay_abc123",
  "checkout_id": "chk_xyz789",
  "shopper_id": "507f1f77bcf86cd799439011",
  "email": "shopper@example.com",
  "ship_to": {
    "street": "123 Main St",
    "city": "Springfield",
    "postal_code": "12345",
    "country": "US"
  },
  "lines": [
    {
      "product_id": "prod_001",
      "quantity": 2,
      "price_snapshot_minor_units": 1500
    }
  ],
  "total_minor_units": 3000,
  "timestamp": "2025-01-10T14:30:00Z"
}
```

**Queue:** `order.requested.queue` (durable, binds to `order.requested`)

---

### API Changes

**Modified Endpoint:** `POST /checkout`

**Before (synchronous):**
```json
{
  "order_id": "ord_def456",
  "status": "paid"
}
```

**After (async):**
```json
{
  "checkout_id": "chk_xyz789",
  "status": "accepted"
}
```

**New Behavior:**
- Returns immediately after inserting checkout + outbox row (atomic write).
- `order_id` is not present in the response (order not created yet).
- `status` is `"accepted"` when payment accepts; `"refused"` when payment refuses (unchanged).

---

## File Touch List

**New Files:**

1. `checkout/outbox.py` — Outbox model, `insert_outbox()`, `mark_published()` helpers
2. `checkout/relay.py` — Relay process main loop, RabbitMQ publisher
3. `checkout/completion_checker.py` — Completion window checker main loop
4. `checkout/rabbitmq_config.py` — RabbitMQ connection and channel setup

**Modified Files:**

1. `checkout/db.py` — Add `checkouts_collection` and `outbox_collection` references
2. `checkout/models.py` — Add `CheckoutRecord`, `OutboxRow`, `CheckoutCreateResponse` models
3. `checkout/routes.py` — Modify `POST /checkout` to insert checkout + outbox atomically
4. `checkout/requirements.txt` — Add `pika`

**Verification Needed:**
- `checkout/main.py` — verify startup logic (file exists per `repo-outline/app`)
- Deployment config (verify in `repo-context/app.md` or separate deployment docs) — how relay and completion checker are deployed

---

## File-by-File Changes

### `checkout/db.py`

**Changes:**
- Add `checkouts_collection = db["checkouts"]`
- Add `outbox_collection = db["outbox"]`

**Before:**
```python
MONGODB_URI = os.environ.get("MONGODB_URI", "mongodb://localhost:27017")

client = MongoClient(MONGODB_URI)
db = client["checkout_db"]
```

**After:**
```python
MONGODB_URI = os.environ.get("MONGODB_URI", "mongodb://localhost:27017")

client = MongoClient(MONGODB_URI)
db = client["checkout_db"]
checkouts_collection = db["checkouts"]
outbox_collection = db["outbox"]
```

---

### `checkout/models.py`

**New Models:**

```python
from datetime import datetime
from typing import Literal, Optional, List
from pydantic import BaseModel, EmailStr, Field


class CheckoutLine(BaseModel):
    product_id: str
    quantity: int
    price_snapshot_minor_units: int


class CheckoutRecord(BaseModel):
    checkout_id: str
    payment_reference: str
    cart_id: str
    shopper_id: Optional[str] = None
    email: Optional[str] = None
    ship_to: ShipTo
    lines: List[CheckoutLine]
    total_minor_units: int
    status: Literal["accepted", "order_created", "cart_cleared", "refunded"]
    order_id: Optional[str] = None
    created_at: datetime
    updated_at: datetime


class OutboxRow(BaseModel):
    event_type: Literal["OrderRequested", "RefundRequested"]
    payload: dict
    payment_reference: str
    published: bool = False
    published_at: Optional[datetime] = None
    created_at: datetime


class CheckoutCreateResponse(BaseModel):
    checkout_id: str
    status: Literal["accepted", "refused"]
    reason: Optional[str] = None

    @model_validator(mode="after")
    def require_status_fields(self):
        if self.status == "refused" and self.reason is None:
            raise ValueError("reason must be present when status is refused")
        return self
```

**Existing `CheckoutResponse` is replaced by `CheckoutCreateResponse`** (new name, no `order_id` field).

---

### `checkout/routes.py`

**Changes:**
- Import `CheckoutCreateResponse`, `CheckoutRecord`, `OutboxRow`, `CheckoutLine` from `models`
- Import `checkouts_collection`, `outbox_collection` from `db`
- Import `uuid` and `datetime`
- Modify `POST /checkout` to:
  1. Generate `checkout_id` (UUID)
  2. After payment accepts, build `CheckoutRecord` and `OutboxRow` documents
  3. Insert both in one atomic write (MongoDB transaction or two sequential inserts in one function scope — verify MongoDB transaction support in `repo-outline/app`; if no transaction, insert checkout first, then outbox, and handle rollback manually)
  4. Return `CheckoutCreateResponse` with `checkout_id` and `status: "accepted"` (no `order_id`)

**Before (payment success block):**
```python
order_payload = _order_payload(
    body, cart_lines, cart_total, payment_data["reference"]
)
try:
    async with httpx.AsyncClient() as client:
        order_response = await client.post(
            f"{ORDER_SERVICE_URL}/orders",
            json=order_payload,
        )
except httpx.RequestError:
    raise HTTPException(
        status_code=500, detail="Order service unavailable"
    ) from None

if order_response.status_code != 201:
    raise HTTPException(status_code=500, detail="Order service unavailable")

order_id = order_response.json()["order_id"]

try:
    async with httpx.AsyncClient() as client:
        delete_response = await client.delete(cart_url)
except httpx.RequestError:
    _log_cart_delete_failure(body.cart_id)
else:
    if delete_response.status_code != 204:
        _log_cart_delete_failure(body.cart_id)

return CheckoutResponse(order_id=order_id, status="paid")
```

**After:**
```python
checkout_id = str(uuid.uuid4())
checkout_record = CheckoutRecord(
    checkout_id=checkout_id,
    payment_reference=payment_data["reference"],
    cart_id=body.cart_id,
    shopper_id=body.shopper_id,
    email=str(body.email) if body.email else None,
    ship_to=body.ship_to,
    lines=[
        CheckoutLine(
            product_id=line["product_id"],
            quantity=line["quantity"],
            price_snapshot_minor_units=line["price_snapshot_minor_units"]
        )
        for line in cart_lines
    ],
    total_minor_units=cart_total,
    status="accepted",
    order_id=None,
    created_at=datetime.utcnow(),
    updated_at=datetime.utcnow()
)

outbox_payload = {
    "event_type": "OrderRequested",
    "payment_reference": payment_data["reference"],
    "checkout_id": checkout_id,
    "shopper_id": body.shopper_id,
    "email": str(body.email) if body.email else None,
    "ship_to": body.ship_to.model_dump(),
    "lines": [
        {
            "product_id": line["product_id"],
            "quantity": line["quantity"],
            "price_snapshot_minor_units": line["price_snapshot_minor_units"]
        }
        for line in cart_lines
    ],
    "total_minor_units": cart_total,
    "timestamp": datetime.utcnow().isoformat()
}

outbox_row = OutboxRow(
    event_type="OrderRequested",
    payload=outbox_payload,
    payment_reference=payment_data["reference"],
    published=False,
    published_at=None,
    created_at=datetime.utcnow()
)

# [ATOMIC_WRITE] Insert checkout + outbox in one write (verify MongoDB transaction support)
try:
    checkouts_collection.insert_one(checkout_record.model_dump(by_alias=True))
    outbox_collection.insert_one(outbox_row.model_dump(by_alias=True))
except PyMongoError:
    raise HTTPException(status_code=500, detail="Failed to store checkout") from None

return CheckoutCreateResponse(checkout_id=checkout_id, status="accepted")
```

**Idempotency Check (before payment call):**
```python
existing_checkout = checkouts_collection.find_one({"payment_reference": body.cart_id})  # [ASSUMPTION] payment_reference is derived from cart_id; verify in HLD
if existing_checkout:
    if existing_checkout["status"] in ["accepted", "order_created", "cart_cleared"]:
        return CheckoutCreateResponse(
            checkout_id=existing_checkout["checkout_id"],
            status="accepted"
        )
    elif existing_checkout["status"] == "refunded":
        raise HTTPException(status_code=400, detail="Checkout was refunded, cannot retry")
```

**Open Question:** How is `payment_reference` generated? HLD says "payment reference is the idempotency key"; is it derived from `cart_id` or generated by checkout before calling payment? Verify in `repo-file/app/payment/routes.py` — payment generates `reference` with `uuid.uuid4()`. Checkout must generate its own stable reference before calling payment, or reuse the payment response's reference. **Assumption:** Checkout generates `payment_reference = str(uuid.uuid4())` before calling payment, and passes it as an idempotency key to payment (but payment does not accept an idempotency key in v1; see `repo-file/app/payment/routes.py`). **Resolution:** Generate `payment_reference` after payment accepts, using the payment response's `reference`. Idempotency is checked by `payment_reference` after payment call.

**Revised Idempotency Check (after payment call):**
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

---

### `checkout/outbox.py` (new file)

```python
from datetime import datetime
from bson import ObjectId
from pymongo.errors import PyMongoError

from db import outbox_collection
from models import OutboxRow


def insert_outbox(row: OutboxRow) -> None:
    """Insert an outbox row. Raises PyMongoError on failure."""
    doc = row.model_dump(by_alias=True)
    doc["_id"] = ObjectId()
    outbox_collection.insert_one(doc)


def mark_published(row_id: ObjectId) -> None:
    """Mark an outbox row as published. Raises PyMongoError on failure."""
    outbox_collection.update_one(
        {"_id": row_id},
        {"$set": {"published": True, "published_at": datetime.utcnow()}}
    )
```

---

### `checkout/rabbitmq_config.py` (new file)

```python
import os
import pika


RABBITMQ_HOST = os.environ.get("RABBITMQ_HOST", "localhost")
RABBITMQ_PORT = int(os.environ.get("RABBITMQ_PORT", "5672"))
RABBITMQ_USER = os.environ.get("RABBITMQ_USER", "guest")
RABBITMQ_PASSWORD = os.environ.get("RABBITMQ_PASSWORD", "guest")
EXCHANGE_NAME = "ecom.events"


def get_rabbitmq_connection():
    """Return a RabbitMQ connection."""
    credentials = pika.PlainCredentials(RABBITMQ_USER, RABBITMQ_PASSWORD)
    parameters = pika.ConnectionParameters(
        host=RABBITMQ_HOST,
        port=RABBITMQ_PORT,
        credentials=credentials
    )
    return pika.BlockingConnection(parameters)


def setup_exchange_and_queue(channel, routing_key: str, queue_name: str):
    """Declare exchange and queue, bind queue to routing key."""
    channel.exchange_declare(
        exchange=EXCHANGE_NAME,
        exchange_type="topic",
        durable=True
    )
    channel.queue_declare(queue=queue_name, durable=True)
    channel.queue_bind(
        exchange=EXCHANGE_NAME,
        queue=queue_name,
        routing_key=routing_key
    )
```

---

### `checkout/relay.py` (new file)

```python
import json
import time
from datetime import datetime

from db import outbox_collection
from rabbitmq_config import get_rabbitmq_connection, setup_exchange_and_queue, EXCHANGE_NAME


POLL_INTERVAL_SECONDS = int(os.environ.get("RELAY_POLL_INTERVAL", "1"))


def publish_outbox_messages():
    """Poll unpublished outbox rows and publish to RabbitMQ."""
    connection = get_rabbitmq_connection()
    channel = connection.channel()
    
    # Ensure exchange exists
    channel.exchange_declare(
        exchange=EXCHANGE_NAME,
        exchange_type="topic",
        durable=True
    )
    
    while True:
        rows = outbox_collection.find({"published": False}).limit(100)
        
        for row in rows:
            event_type = row["event_type"]
            payload = row["payload"]
            row_id = row["_id"]
            
            routing_key = {
                "OrderRequested": "order.requested",
                "RefundRequested": "payment.refund.requested"
            }.get(event_type)
            
            if routing_key is None:
                print(f"Unknown event_type: {event_type}, skipping row {row_id}")
                continue
            
            try:
                channel.basic_publish(
                    exchange=EXCHANGE_NAME,
                    routing_key=routing_key,
                    body=json.dumps(payload),
                    properties=pika.BasicProperties(
                        delivery_mode=2,  # persistent
                        content_type="application/json"
                    )
                )
                # Mark published after RabbitMQ acks
                outbox_collection.update_one(
                    {"_id": row_id},
                    {"$set": {"published": True, "published_at": datetime.utcnow()}}
                )
                print(f"Published {event_type} for payment_reference {row['payment_reference']}")
            except Exception as e:
                print(f"Failed to publish row {row_id}: {e}")
                # Do not mark published; retry on next poll
        
        time.sleep(POLL_INTERVAL_SECONDS)


if __name__ == "__main__":
    publish_outbox_messages()
```

---

### `checkout/completion_checker.py` (new file)

```python
import os
import time
from datetime import datetime, timedelta
from bson import ObjectId

from db import checkouts_collection, outbox_collection


COMPLETION_WINDOW_SECONDS = int(os.environ.get("COMPLETION_WINDOW_SECONDS", "300"))  # 5 minutes
POLL_INTERVAL_SECONDS = int(os.environ.get("COMPLETION_CHECKER_POLL_INTERVAL", "30"))


def check_expired_checkouts():
    """Poll checkouts for expired accepted checkouts and insert RefundRequested outbox rows."""
    while True:
        cutoff = datetime.utcnow() - timedelta(seconds=COMPLETION_WINDOW_SECONDS)
        expired = checkouts_collection.find({
            "status": "accepted",
            "created_at": {"$lt": cutoff}
        })
        
        for checkout in expired:
            checkout_id = checkout["checkout_id"]
            payment_reference = checkout["payment_reference"]
            
            # Insert RefundRequested outbox row
            outbox_row = {
                "_id": ObjectId(),
                "event_type": "RefundRequested",
                "payload": {
                    "event_type": "RefundRequested",
                    "payment_reference": payment_reference,
                    "checkout_id": checkout_id,
                    "timestamp": datetime.utcnow().isoformat()
                },
                "payment_reference": payment_reference,
                "published": False,
                "published_at": None,
                "created_at": datetime.utcnow()
            }
            outbox_collection.insert_one(outbox_row)
            
            # Update checkout status to refund_pending
            checkouts_collection.update_one(
                {"_id": checkout["_id"]},
                {"$set": {"status": "refund_pending", "updated_at": datetime.utcnow()}}
            )
            
            print(f"Inserted RefundRequested for checkout {checkout_id}, payment_reference {payment_reference}")
        
        time.sleep(POLL_INTERVAL_SECONDS)


if __name__ == "__main__":
    check_expired_checkouts()
```

---

### `checkout/requirements.txt`

**Add:**
```
pika
```

**Full file after:**
```
fastapi
uvicorn
pymongo
pydantic
httpx
email-validator
pika
```

---

## Risks

### R1: Outbox relay fails to publish a message

**Risk:** Relay crashes or is killed before marking an outbox row as published.  
**Impact:** Message is published twice. Order service creates one order (idempotency via `payment_reference` in Ticket 2). Low risk.  
**Mitigation:** Order service handles duplicates via `payment_reference` (Ticket 2).

---

### R2: RabbitMQ is down

**Risk:** Relay cannot publish messages; messages pile up in the outbox.  
**Impact:** Orders are not created until RabbitMQ recovers. Completion window may expire, triggering refunds.  
**Mitigation:** Monitor RabbitMQ health. Set completion window to 5 minutes (longer than typical RabbitMQ downtime). Alert on outbox backlog (`outbox.pending.count` metric).

---

### R3: Checkout and outbox insert is not atomic

**Risk:** Checkout record is inserted, but outbox insert fails (or vice versa).  
**Impact:** Accepted charge with no message to send, or message with no checkout record.  
**Mitigation:** Use MongoDB transaction if supported (verify in `repo-outline/app`). If not supported, insert checkout first, then outbox; on outbox failure, delete the checkout record (rollback). Log the failure.

**Verification Needed:** Does `pymongo` in the existing stack support transactions? Check `repo-outline/app` for `MongoClient` usage. If `checkout/db.py` uses `MongoClient` without replica set, transactions are not supported (transactions require replica set or sharded cluster). **Assumption:** No transaction support in v1. Insert checkout first, then outbox; on outbox failure, attempt to delete the checkout record.

**Revised Insert Logic in `checkout/routes.py`:**
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

---

### R4: Duplicate checkout submission charges twice

**Risk:** Client retries checkout with a new payment reference (payment generates a new reference per charge).  
**Impact:** Two charges, two orders.  
**Mitigation:** Client-side idempotency: clients should generate a stable `payment_reference` per checkout attempt (e.g. hash of `cart_id` + `shopper_id` + timestamp). Server-side: checkout detects duplicates by `payment_reference` after payment call. **But** payment does not accept an idempotency key in v1 (see `repo-file/app/payment/routes.py`). **Known gap:** A repeated checkout with the same cart can charge twice. Mark as `[IDEMPOTENCY_DEFERRED]`.

**Resolution:** Add a note in the checkout route's docstring: `[IDEMPOTENCY_DEFERRED] a repeated checkout can charge twice if the payment reference is different`.

---

### R5: Completion window too short

**Risk:** Order service is slow; completion window expires before the order is created.  
**Impact:** Checkout refunds the charge; order appears later (orphaned order).  
**Mitigation:** Set completion window to 5 minutes (configurable via `COMPLETION_WINDOW_SECONDS`). Monitor order creation latency. If p99 latency approaches 5 minutes, increase the window.

---

### R6: Relay or completion checker deployment failure

**Risk:** Relay or completion checker is not deployed, or crashes on startup.  
**Impact:** Outbox rows are never published; orders are never created; all checkouts are refunded after 5 minutes.  
**Mitigation:** Monitor relay and completion checker process health. Alert on process down. Include relay and completion checker in deployment smoke tests.

**Verification Needed:** How are relay and completion checker deployed? Verify in `repo-context/app.md` or deployment config. **Assumption:** Deployed as separate systemd services or Docker containers. Add startup scripts and health checks.

---

## Open Questions

1. **MongoDB Transaction Support:** Does the existing `pymongo` setup support transactions (requires replica set or sharded cluster)? If not, the checkout + outbox insert is not atomic, and rollback is manual.
2. **Payment Idempotency:** Payment does not accept an idempotency key in v1. A repeated checkout with the same cart can charge twice. Should payment be modified to accept an idempotency key, or is this deferred?
3. **Relay and Completion Checker Deployment:** How are these processes deployed (systemd, Docker, Kubernetes)? Are they in the same repo or separate? Add startup scripts to `checkout/` or root?
4. **RabbitMQ Connection Pooling:** Should the relay use connection pooling, or one connection per instance?
5. **Completion Window Checker Refund Status:** After inserting `RefundRequested`, does the completion checker poll payment status to update checkout to `refunded`, or does payment publish a `RefundCompleted` event? (HLD says this is an open question.)

---

## Notes

- **[ATOMIC_WRITE]** Checkout + outbox insert is not atomic in v1 (no MongoDB transaction support assumed). Rollback is manual.
- **[IDEMPOTENCY_DEFERRED]** A repeated checkout can charge twice if the payment reference is different.
- **[CONFIG_DEFERRED]** RabbitMQ host, port, user, password, and poll intervals are environment variables with defaults.
- **[LOGGING_DEFERRED]** Print statements only in v1.
- **[ERROR_DETAIL_DEFERRED]** Database failures return a fixed 500 detail.
- **[DEPLOYMENT_DEFERRED]** Relay and completion checker deployment scripts are not in this ticket.
```

## Spec `specs/2.md`

# Tech Stub — PER-278-2

**Ticket:** Order service consumes OrderRequested and publishes OrderCreated  
**Repository:** app  
**Covers AC:** AC3, AC4 (order side), AC8

---

## Stack

- **Language:** Node.js (existing order service runtime)
- **Framework:** Express (existing order service HTTP framework)
- **Database:** MongoDB (existing order service database: `order_db`)
- **Message Broker:** RabbitMQ (new dependency for async communication)
- **RabbitMQ Client:** `amqplib` (Node.js AMQP 0-9-1 client library)

---

## Schema/API Contracts

### Message Schema: OrderRequested (consumed)

**Exchange:** `ecom.events` (type: `topic`)  
**Routing Key:** `order.requested`  
**Queue:** `order.requested.queue`  
**Published By:** Checkout (via relay)  
**Consumed By:** Order

**Message Body:**
```json
{
  "event_type": "OrderRequested",
  "payment_reference": "pay_abc123",
  "checkout_id": "chk_xyz789",
  "shopper_id": "507f1f77bcf86cd799439011",
  "email": "shopper@example.com",
  "ship_to": {
    "street": "123 Main St",
    "city": "Springfield",
    "postal_code": "12345",
    "country": "US"
  },
  "lines": [
    {
      "product_id": "prod_001",
      "quantity": 2,
      "price_snapshot_minor_units": 1500
    }
  ],
  "total_minor_units": 3000,
  "timestamp": "2025-01-10T14:30:00Z"
}
```

**Field Notes:**
- `payment_reference` (string, required): Idempotency key. Order checks for existing order by this field before inserting.
- `checkout_id` (string, required): Traceability; not stored in order document.
- `shopper_id` (string, optional): Present for signed-in checkout.
- `email` (string, optional): Present for guest checkout.
- `lines[].price_snapshot_minor_units` (integer, required): Mapped to `price_minor_units` in the order document on insert (see `repo-file/app/order/models.js` and `repo-file/app/order/routes.js`).
- `total_minor_units` (integer, required): Trusted from checkout; no recalculation.
- `timestamp` (string, ISO 8601, required): Event timestamp for observability.

---

### Message Schema: OrderCreated (published)

**Exchange:** `ecom.events` (type: `topic`)  
**Routing Key:** `order.created`  
**Queue:** `order.created.queue`  
**Published By:** Order  
**Consumed By:** Checkout

**Message Body:**
```json
{
  "event_type": "OrderCreated",
  "payment_reference": "pay_abc123",
  "checkout_id": "chk_xyz789",
  "order_id": "ord_def456",
  "timestamp": "2025-01-10T14:30:05Z"
}
```

**Field Notes:**
- `payment_reference` (string, required): Same as `OrderRequested`.
- `checkout_id` (string, required): Echoed from `OrderRequested` for correlation.
- `order_id` (string, required): MongoDB ObjectId as hex string.
- `timestamp` (string, ISO 8601, required): Order creation timestamp.

---

### MongoDB Schema: orders collection (modified)

**New Field:**
- `payment_reference` (string, required, unique): Idempotency key. Prevents duplicate orders for the same payment.

**Existing Fields (from `repo-file/app/order/models.js`):**
- `_id` (ObjectId, required): MongoDB-generated order id.
- `shopper_id` (string, optional): Present when the order is for a signed-in shopper.
- `email` (string, optional): Present when the order is for a guest.
- `ship_to` (object, required): `{street, city, postal_code, country}`.
- `lines` (array, required): `[{product_id, quantity, price_minor_units}]`. Renamed from `price_snapshot_minor_units` in the message to `price_minor_units` in the database (see HLD).
- `total_minor_units` (integer, required): Total charge amount.
- `created_at` (Date, required): Order creation timestamp.

**Updated Document Example:**
```javascript
{
  _id: ObjectId("507f1f77bcf86cd799439011"),
  payment_reference: "pay_abc123",  // NEW
  shopper_id: "507f1f77bcf86cd799439012",
  email: null,
  ship_to: {
    street: "123 Main St",
    city: "Springfield",
    postal_code: "12345",
    country: "US"
  },
  lines: [
    {
      product_id: "prod_001",
      quantity: 2,
      price_minor_units: 1500  // renamed from price_snapshot_minor_units
    }
  ],
  total_minor_units: 3000,
  created_at: ISODate("2025-01-10T14:30:05Z")
}
```

**Index:**
- Create a unique index on `payment_reference` to enforce idempotency and prevent duplicate orders.

---

### HTTP API (unchanged)

No changes to existing HTTP endpoints:
- `POST /orders` (verify in `repo-file/app/order/routes.js`)
- `GET /orders/{order_id}` (verify in `repo-file/app/order/routes.js`)

The consumer creates orders via RabbitMQ, not HTTP. The HTTP `POST /orders` endpoint remains for direct order creation (not used by checkout in this feature).

---

## File Touch List

### New Files

1. **`order/consumer.js`**
   - RabbitMQ consumer for `OrderRequested`.
   - Idempotency check: query `orders` collection by `payment_reference`.
   - Insert order if not exists.
   - Publish `OrderCreated` to RabbitMQ.
   - Manual message acknowledgement.

2. **`order/rabbitmq.js`**
   - RabbitMQ connection setup.
   - Channel creation.
   - Exchange and queue declaration.
   - Queue binding to routing key.
   - Publisher and consumer helpers.

---

### Modified Files

1. **`order/models.js`**
   - Add `payment_reference` to `createOrderSchema` (required).
   - Validation: `payment_reference` must be a non-empty trimmed string.

2. **`order/db.js`**
   - Add index creation on `payment_reference` (unique).
   - Run index creation in `connectDB()` before resolving.

3. **`order/index.js`**
   - Import `order/consumer.js`.
   - Start RabbitMQ consumer after MongoDB connection is established.
   - Add graceful shutdown for RabbitMQ connection and channel.

4. **`order/package.json`**
   - Add `amqplib` dependency: `"amqplib": "^0.10.3"` (verify current version on npm before writing).

---

## Risks

### R1: Duplicate message delivery creates duplicate orders

**Risk:** RabbitMQ delivers `OrderRequested` twice (network retry, relay crash after publish but before marking published).  
**Impact:** Two orders for the same payment reference.  
**Mitigation:** Query `orders` by `payment_reference` before inserting. If exists, skip insertion and publish `OrderCreated` with the existing order id. The unique index on `payment_reference` prevents duplicate inserts even if the query race-conditions.  
**Severity:** Medium (mitigated by idempotency logic and unique index).

---

### R2: RabbitMQ connection failure on startup

**Risk:** Order service starts but cannot connect to RabbitMQ (RabbitMQ down, network issue, wrong credentials).  
**Impact:** Consumer does not start; messages pile up in the queue.  
**Mitigation:** Retry connection with exponential backoff (5s, 10s, 20s, max 1 minute). Log connection errors. Fail the service startup if RabbitMQ is unreachable after retries (crash-loop backoff in deployment orchestrator).  
**Severity:** High (blocks order creation; must be detected and alerted immediately).

---

### R3: Message parsing failure (malformed OrderRequested)

**Risk:** `OrderRequested` message body does not match the schema (missing required field, wrong type).  
**Impact:** Consumer crashes or sends a nack, message goes to DLQ.  
**Mitigation:** Validate message body with Joi schema before processing. If invalid, log the error, nack the message (no requeue), and move on. The message goes to DLQ for manual review.  
**Severity:** Medium (isolated failure; does not block other messages).

---

### R4: MongoDB insertion failure (network, disk full, replica set unavailable)

**Risk:** `insertOne()` throws an error.  
**Impact:** Message is nacked, redelivered, and retried. If the failure is transient (network blip), the retry succeeds. If persistent (disk full), the message is retried until DLQ threshold.  
**Mitigation:** Nack the message without requeue on insertion failure (let RabbitMQ retry policy handle it). Log the error with the payment reference for traceability. Monitor DLQ depth and alert on buildup.  
**Severity:** High (blocks order creation; must be detected and resolved quickly).

---

### R5: OrderCreated publish failure (RabbitMQ down after order insertion)

**Risk:** Order inserted successfully, but `OrderCreated` publish fails.  
**Impact:** Checkout never receives `OrderCreated`; cart is not emptied; refund is triggered after completion window.  
**Mitigation:** Use RabbitMQ publisher confirms (`channel.waitForConfirms()`). If publish fails, log the error but **do not nack the original message** (order already exists; nack would reprocess and create a duplicate). Instead, rely on a background job or manual intervention to republish `OrderCreated` for orphaned orders (future work; logged as `[ORPHANED_ORDER_REPUBLISH_DEFERRED]`).  
**Severity:** Medium (manual intervention required; order exists but checkout state is stale).

---

### R6: Message ordering (OrderCreated arrives before OrderRequested)

**Risk:** Network reordering or relay delay causes `OrderCreated` to arrive at checkout before `OrderRequested` is processed.  
**Impact:** Checkout receives `OrderCreated` for a payment reference it has not stored yet.  
**Mitigation:** Checkout ignores `OrderCreated` for unknown payment references (logs a warning). When `OrderRequested` is processed later, the order already exists, and `OrderCreated` is published again (idempotency).  
**Severity:** Low (self-healing; logged for observability).

---

### R7: Unique index violation on payment_reference

**Risk:** Race condition: two consumer instances query for the same payment reference, find nothing, and both insert.  
**Impact:** One insert succeeds, the other fails with a duplicate key error.  
**Mitigation:** Catch duplicate key error (`MongoError` with code 11000), log it as a duplicate detection (not an error), and query the existing order to publish `OrderCreated`. The failed insert is a no-op; the message is acked.  
**Severity:** Low (expected behavior; handled gracefully).

---

## Open Questions

1. **RabbitMQ connection URI:** Is `RABBITMQ_URI` an environment variable, or is it hardcoded to `amqp://localhost`? (Answer: environment variable `RABBITMQ_URI`, default `amqp://localhost`; see HLD.)
2. **Dead Letter Queue configuration:** Who creates the DLQ, and what is the retry policy (max retries, TTL)? (Answer: DLQ is created by RabbitMQ config, not the consumer; retry policy is 3 retries with exponential backoff; see HLD.)
3. **Trace ID propagation:** Is `trace_id` in the message headers, and should it be logged? (Answer: yes, `trace_id` is in headers; log it for distributed tracing; see HLD Observability section.)
4. **Order creation latency SLA:** What is the acceptable p99 latency for order creation? (Answer: <1 second; see HLD Performance section.)
5. **Background job for orphaned OrderCreated republish:** Is this in scope, or logged as `[DEFERRED]`? (Answer: deferred; see R5 mitigation.)

---

## Notes

- **Idempotency is mandatory:** Every `OrderRequested` message may be delivered more than once. The unique index on `payment_reference` and the query-before-insert pattern ensure exactly-once order creation semantics.
- **Manual message acknowledgement:** Messages are acked only after the order is inserted and `OrderCreated` is published (or skipped due to idempotency). If the process crashes before acking, RabbitMQ redelivers the message.
- **Field name mismatch:** The message uses `price_snapshot_minor_units`, but the database schema uses `price_minor_units`. The consumer must map `price_snapshot_minor_units` to `price_minor_units` when inserting (verify existing schema in `repo-file/app/order/models.js`).
- **No HTTP API changes:** The consumer creates orders via RabbitMQ, not HTTP. The existing `POST /orders` endpoint is unchanged and remains public (verify in `repo-file/app/order/routes.js`).
- **Graceful shutdown:** The consumer must close the RabbitMQ channel and connection on `SIGTERM` or `SIGINT` to avoid message loss.
- **Logging:** Log every order creation (payment reference, order id, timestamp) and every duplicate detection (payment reference, existing order id). Use structured logging (JSON) for observability.
- **Testing:** Integration tests must cover: order creation from `OrderRequested`, idempotency (duplicate message does not create two orders), `OrderCreated` publish, and duplicate key error handling.

## Spec `specs/3.md`

```markdown
# Tech Stub — Ticket 3

**Ticket:** PER-278-3  
**Title:** Checkout empties the cart on OrderCreated with retry  
**Epic:** PER-278 — A paid checkout still becomes one order when order or cart is briefly down  
**Type:** Story  
**Covers AC:** AC5, AC8

---

## Stack

**Language & Framework:**
- Python 3.12
- FastAPI

**Database:**
- MongoDB (checkout service owns `checkout_db`)

**Message Broker:**
- RabbitMQ (exchange: `ecom.events`, queue: `order.created.queue`)

**Dependencies:**
- `pika` (RabbitMQ Python client)
- `httpx` (async HTTP client for cart deletion)
- `pymongo` (MongoDB driver)
- `fastapi`, `uvicorn`, `pydantic` (existing)

---

## Schema and API Contracts

### Message Schema: OrderCreated (consumed)

**Published by:** Order service  
**Routing key:** `order.created`  
**Queue:** `order.created.queue`

```json
{
  "event_type": "OrderCreated",
  "payment_reference": "pay_abc123",
  "checkout_id": "chk_xyz789",
  "order_id": "ord_def456",
  "timestamp": "2025-01-10T14:30:05Z"
}
```

**Consumer behavior:**
- On receipt of `OrderCreated`, the checkout service:
  1. Updates the checkout record (verify schema in HLD): set `status` to `"order_created"`, set `order_id` field.
  2. Attempts `DELETE /cart/{cart_id}` (verify endpoint in `repo-outline/app`).
  3. On success, updates checkout `status` to `"cart_cleared"` and acks the message.
  4. On failure, logs the failure, schedules a retry (exponential backoff: 5s, 10s, 20s, max 3 retries), and acks the message immediately (do not block other checkouts).

---

### HTTP API: Cart Deletion (called by checkout)

**Endpoint:** `DELETE /cart/{cart_id}`  
**Service:** Cart (port 8003, verify URL in `repo-file/app/checkout/routes.py`)  
**URL:** `http://localhost:8003/cart/{cart_id}`  
**Expected Response:** `204 No Content`  
**Failure Handling:** Network error, 500, or timeout triggers retry.

---

### Database Schema: Checkouts Collection (updated)

**Collection:** `checkout_db.checkouts`  
**New Fields:**
- `order_id: str | None` — set when `OrderCreated` is consumed
- `status: "accepted" | "order_created" | "cart_cleared" | "refunded"` — updated as the checkout progresses

**Write Operations:**
1. **On OrderCreated receipt:** Update by `payment_reference`, set `status = "order_created"`, `order_id = <value>`, `updated_at = now()`.
2. **On cart delete success:** Update by `payment_reference`, set `status = "cart_cleared"`, `updated_at = now()`.

**Idempotency:**
- If `status` is already `"order_created"` or `"cart_cleared"` when `OrderCreated` arrives, skip the update and ack the message.

---

## RabbitMQ Topology

**Exchange:** `ecom.events` (type: `topic`, durable)  
**Queue:** `order.created.queue` (durable, binds to routing key `order.created`)  
**Message Acknowledgement:** Manual ack after processing (update checkout record and attempt cart delete).  
**Retry Strategy:** Consumer-side exponential backoff for cart delete failures. RabbitMQ message is acked immediately after updating the checkout record (do not hold the message hostage on cart service downtime).

---

## File Touch List

### New Files

1. **`checkout/consumers.py`**
   - RabbitMQ consumer for `OrderCreated`.
   - Implements retry logic for cart deletion (exponential backoff, max 3 retries over 1 minute).
   - Logs cart delete failures (verify logging pattern in `repo-file/app/checkout/routes.py`).
   - Updates `checkouts` collection (set `order_id`, update `status`).

---

### Modified Files

1. **`checkout/main.py`**
   - Start the RabbitMQ consumer on app startup.
   - Add a background task or thread to run the consumer loop (verify async startup pattern in `repo-file/app/checkout/main.py`).

2. **`checkout/db.py`**
   - Add `checkouts_collection` reference (verify collection naming in `repo-file/app/checkout/db.py`).
   - **Note:** The outbox pattern and `outbox_collection` are added in Ticket 1; this ticket uses the `checkouts` collection only.

3. **`checkout/models.py`**
   - **No changes.** The `CheckoutResponse` model already exists (verify in `repo-file/app/checkout/models.py`). The `checkouts` collection schema is not a Pydantic model in v1 (see HLD).

4. **`checkout/requirements.txt`**
   - Add `pika` (RabbitMQ client) if not already present (verify in `repo-file/app/checkout/requirements.txt`).

---

### Files Read (No Writes)

1. **`checkout/routes.py`**
   - Read to verify `CART_SERVICE_URL` (currently `http://localhost:8003`, hardcoded).
   - Read to verify cart delete failure logging pattern (`_log_cart_delete_failure`).

2. **`cart/routes.py`**
   - Read to verify `DELETE /cart/{cart_id}` endpoint signature and expected response (verify in `repo-outline/app`).

---

## Risks

### R1: Cart service down during all retry attempts

**Risk:** Cart delete fails after 3 retries over 1 minute.  
**Impact:** Order exists, cart not emptied. Shopper sees a non-empty cart after checkout.  
**Mitigation:** Log a warning (verify logging in `repo-outline/app`). Manual intervention (ops deletes the cart). Future: add a background job to retry cart deletes indefinitely (see HLD Deferred section).  
**Acceptance:** AC5 covers this scenario; the test verifies the cart is eventually removed on a later attempt.

---

### R2: Duplicate `OrderCreated` message delivery

**Risk:** RabbitMQ redelivers the message after a consumer crash.  
**Impact:** Checkout receives the same `OrderCreated` twice.  
**Mitigation:** Idempotency: before updating the checkout record, check the current `status`. If already `"order_created"` or `"cart_cleared"`, skip the update and ack the message.

---

### R3: Checkout record does not exist for the payment reference

**Risk:** `OrderCreated` arrives before the checkout record is written (network reordering, relay delay).  
**Impact:** Checkout cannot find the record to update.  
**Mitigation:** Log a warning (verify logging in `repo-outline/app`). When the checkout record is written later, it will not trigger another cart delete (the order already exists). Future: add a retry queue for orphaned `OrderCreated` messages (see HLD Open Questions).

---

### R4: RabbitMQ connection failure

**Risk:** Checkout consumer cannot connect to RabbitMQ on startup.  
**Impact:** `OrderCreated` messages are not consumed; carts are not emptied.  
**Mitigation:** Retry connection with exponential backoff (5s, 10s, 20s, 40s, max 2 minutes). Log connection failures (verify logging in `repo-outline/app`). If RabbitMQ is down for >2 minutes, the consumer crashes and must be restarted manually.

---

### R5: Order service publishes `OrderCreated` before checkout writes the checkout record

**Risk:** Race condition if order service is very fast.  
**Impact:** Checkout consumer cannot find the checkout record to update.  
**Mitigation:** Log a warning. The checkout record will be written shortly (checkout service returns the response after writing). When the record appears, the order already exists, so no second cart delete is triggered. See R3.

---

## Open Questions

1. **Retry queue for orphaned `OrderCreated` messages:** Should checkout add a retry mechanism for messages that arrive before the checkout record exists, or is logging sufficient?
2. **Infinite cart delete retry:** Should cart deletion be retried indefinitely in a background job, or is the 3-retry limit acceptable for v1?
3. **RabbitMQ connection pooling:** Should the consumer use connection pooling, or one connection per checkout service instance?
4. **Consumer concurrency:** Should the `OrderCreated` consumer process messages serially or in parallel?
5. **Dead Letter Queue handling:** How are DLQ messages processed (manual review, automated retry, alert)?

---

## Notes

- **Assumption:** The `checkouts` collection and `outbox` collection are added in Ticket 1 (Checkout outbox and OrderRequested publish via RabbitMQ). This ticket depends on Ticket 1.
- **Assumption:** The order service publishes `OrderCreated` messages in Ticket 2 (Order service consumes OrderRequested and publishes OrderCreated). This ticket depends on Ticket 2.
- **Assumption:** The cart service's `DELETE /cart/{cart_id}` endpoint is unchanged (verify in `repo-outline/app`).
- **Assumption:** The checkout service already has access to `checkouts_collection` (added in Ticket 1).
- **Assumption:** The completion window checker and refund logic are added in Ticket 4; this ticket does not handle refunds.
- **Idempotency key:** `payment_reference` is used to find the checkout record and to detect duplicate `OrderCreated` messages.
- **Logging:** Use the same logging pattern as `checkout/routes.py` (`_log_cart_delete_failure`).
- **Error handling:** Cart delete failures are logged and retried; they do not block message acknowledgement.
- **Database isolation (AC8):** Checkout reads and writes only `checkout_db`. No cross-database queries.

---

## End of Tech Stub
```

## Spec `specs/4.md`

```markdown
# Tech Stub — PER-278-4

**Epic:** PER-278 — A paid checkout still becomes one order when order or cart is briefly down  
**Ticket:** 4 — Refund a charge when the order never appears within the completion window  
**Type:** Story  
**Covers:** AC6, AC8

---

## Stack

- **Language:** Python 3.12
- **Framework:** FastAPI (checkout, payment)
- **Database:** MongoDB (one database per service: `checkout_db`, `payment_db`)
- **Message Broker:** RabbitMQ (exchange: `ecom.events`, durable queues)
- **RabbitMQ Client:** `pika` (Python)
- **Deployment:** Standalone processes (checkout service, payment service, completion window checker, relay)

---

## Schema and API Contracts

### Checkout Service Data Schema

**`checkouts` collection** (MongoDB, `checkout_db`):

```python
{
  "_id": ObjectId,
  "checkout_id": str,  # UUID
  "payment_reference": str,  # idempotency key
  "cart_id": str,
  "shopper_id": str | None,
  "email": str,
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
  "status": "accepted" | "order_created" | "cart_cleared" | "refunded" | "refund_pending",
  "order_id": str | None,
  "created_at": datetime,
  "updated_at": datetime
}
```

**`outbox` collection** (MongoDB, `checkout_db`):

```python
{
  "_id": ObjectId,
  "event_type": "OrderRequested" | "RefundRequested",
  "payload": dict,  # message body (JSON)
  "payment_reference": str,  # for idempotency
  "published": bool,
  "published_at": datetime | None,
  "created_at": datetime
}
```

---

### Payment Service Data Schema

**`charges` collection** (MongoDB, `payment_db`):

```python
{
  "_id": ObjectId,
  "reference": str,  # payment reference (UUID)
  "amount_minor_units": int,
  "card": str | None,
  "status": "accepted" | "refused" | "refunded",  # NEW: refunded
  "refunded_at": datetime | None,  # NEW
  "timestamp": datetime
}
```

---

### RabbitMQ Message Schema: RefundRequested

**Published by:** Checkout (via relay, triggered by completion window checker)  
**Consumed by:** Payment  
**Routing key:** `payment.refund.requested`

```json
{
  "event_type": "RefundRequested",
  "payment_reference": "pay_abc123",
  "checkout_id": "chk_xyz789",
  "timestamp": "2025-01-10T14:35:00Z"
}
```

---

### RabbitMQ Topology (New Queue)

**Exchange:** `ecom.events` (type: `topic`, durable)

**New Queue:**
- **Queue name:** `payment.refund.requested.queue`
- **Binding key:** `payment.refund.requested`
- **Durability:** durable
- **Messages:** persistent (`delivery_mode=2`)
- **Acknowledgements:** manual acks (ack after processing)
- **DLQ:** `payment.refund.requested.dlq` (for messages that fail after 3 retries)

---

### API: Payment Service (No HTTP Changes)

No new HTTP endpoints. Refunds are triggered by RabbitMQ messages only.

**Existing endpoint (unchanged):**
- `POST /charge` → `{status, reference?, reason?}`

---

## File Touch List

### New Files

1. **`payment/consumers.py`**
   - RabbitMQ consumer for `RefundRequested`
   - Refund logic: query charge by `payment_reference`, update `status` to `refunded`, set `refunded_at`
   - Idempotency: skip if already refunded
   - Manual ack after processing
   - DLQ on failure after retries

2. **`checkout/completion_checker.py`**
   - Standalone Python script (runs as a background process)
   - Polls `checkouts` collection every 30 seconds (configurable via `POLL_INTERVAL_SECONDS`)
   - Finds checkouts with `status == "accepted"` and `created_at < now - COMPLETION_WINDOW_SECONDS`
   - For each expired checkout:
     - Insert an `outbox` row with `event_type: "RefundRequested"`
     - Update checkout `status` to `refund_pending`
   - Logs expired checkouts and inserted refund requests

---

### Modified Files

1. **`payment/models.py`**
   - Add `"refunded"` to `ChargeResponse.status` Literal
   - Add `refunded_at: Optional[datetime]` field to internal charge document schema (docstring or type hint)

2. **`payment/main.py`**
   - Import and start RabbitMQ consumer (`consumers.py`) on app startup
   - Use `asyncio.create_task()` or threading to run the consumer in the background

3. **`payment/requirements.txt`**
   - Add `pika` (RabbitMQ client for Python)

4. **`checkout/completion_checker.py`** (new, listed above)
   - Entry point: `if __name__ == "__main__": main()`
   - Deployment: run as a systemd service, cron job, or Docker container

5. **`checkout/db.py`**
   - Add `outbox_collection` reference (already exists from previous tickets, verify)

6. **`checkout/models.py`**
   - Add `"refund_pending"` and `"refunded"` to `CheckoutResponse.status` Literal (if not already present)

---

## Risks

### R1: Completion window too short

**Risk:** Order service is slow; completion window expires before the order is created.  
**Impact:** Checkout refunds the charge; order appears later (orphaned order).  
**Mitigation:**
- Default completion window: 5 minutes (configurable via `COMPLETION_WINDOW_SECONDS`)
- Monitor order creation latency (p99)
- If p99 latency approaches 5 minutes, increase the window

---

### R2: Completion window checker crashes after inserting outbox, before updating checkout status

**Risk:** Outbox row for `RefundRequested` exists, but checkout status is still `accepted`.  
**Impact:** Relay publishes the refund request, but checkout status is not updated to `refund_pending`.  
**Mitigation:**
- Completion window checker re-polls and finds the same checkout again (still `accepted`)
- Inserts a duplicate outbox row (same `payment_reference`)
- Relay publishes the duplicate message
- Payment service handles the duplicate via idempotency (`status == "refunded"` check)
- Completion window checker should insert outbox and update checkout in a single MongoDB transaction (verify MongoDB transaction support)

---

### R3: Payment service consumer crashes after refunding, before acking the message

**Risk:** Charge is refunded, message is not acked, RabbitMQ redelivers the message.  
**Impact:** Payment service receives the same `RefundRequested` message again.  
**Mitigation:**
- Payment service idempotency: check `status == "refunded"` before refunding
- If already refunded, skip the refund and ack the message

---

### R4: Checkout status is never updated to `refunded`

**Risk:** Completion window checker inserts `RefundRequested` outbox, payment service refunds, but checkout status remains `refund_pending`.  
**Impact:** Checkout status endpoint returns `refund_pending` instead of `refunded`.  
**Mitigation:**
- **Open Question (from HLD):** Should payment service publish a `RefundCompleted` event, or should the completion window checker poll payment status to update checkout to `refunded`?
- **For this ticket:** Checkout status is updated to `refund_pending` by the completion window checker. The final update to `refunded` is deferred to a future ticket (or handled by a new consumer for `RefundCompleted`).
- **Alternative:** Completion window checker polls payment service (HTTP `GET /charges/{payment_reference}`) to check refund status, then updates checkout to `refunded`. (Requires a new payment service endpoint.)

---

### R5: RabbitMQ is down when the completion window checker runs

**Risk:** Completion window checker inserts `RefundRequested` outbox, but relay cannot publish to RabbitMQ.  
**Impact:** Refund is delayed until RabbitMQ recovers. If RabbitMQ is down for longer than the completion window, multiple checkouts may expire.  
**Mitigation:**
- Monitor RabbitMQ health
- Relay retries publishing until RabbitMQ is available
- Completion window is 5 minutes (longer than typical RabbitMQ downtime)

---

### R6: Duplicate refund requests published (relay crash before marking outbox row as published)

**Risk:** Relay publishes `RefundRequested` to RabbitMQ, crashes before marking the outbox row as published, re-publishes on restart.  
**Impact:** Payment service receives the same `RefundRequested` message twice.  
**Mitigation:**
- Payment service idempotency: check `status == "refunded"` before refunding
- RabbitMQ delivers the duplicate; payment service skips the refund and acks

---

## Open Questions

1. **Completion window duration:** Default is 5 minutes. Should this be configurable per checkout, or is a global environment variable acceptable?
2. **Checkout status update to `refunded`:** Should payment service publish a `RefundCompleted` event, or should the completion window checker poll payment status via HTTP to update checkout to `refunded`?
3. **Completion window checker deployment:** Should this run as a systemd service, cron job, Kubernetes CronJob, or background worker thread inside the checkout service?
4. **MongoDB transactions:** Should the completion window checker insert the outbox row and update the checkout status in a single transaction? (MongoDB supports multi-document transactions; verify support in the current MongoDB version.)
5. **Refund idempotency at payment service:** Is it sufficient to check `status == "refunded"` before refunding, or should payment service also check the `refunded_at` timestamp to prevent race conditions?
6. **DLQ handling:** How are dead-letter messages in `payment.refund.requested.dlq` processed? Manual review, automated retry, or alert?

---

## Deferred

- **RefundCompleted event:** Payment service does not publish `RefundCompleted` in this ticket. Checkout status update to `refunded` is deferred.
- **HTTP refund endpoint:** No `POST /refund` endpoint in v1. Refunds are triggered by RabbitMQ messages only.
- **Charge lookup endpoint:** No `GET /charges/{charge_id}` or `GET /charges?reference={payment_reference}` endpoint in v1. Payment service does not expose charge status via HTTP.
- **Human review queue for refunds:** No admin UI or approval workflow for refunds in v1.
- **Refund webhook to external payment provider:** Payment service is a test double; no real payment provider is called for refunds.

---

## Notes

- **Completion window default:** 5 minutes (configurable via `COMPLETION_WINDOW_SECONDS` environment variable).
- **Completion window checker poll interval:** 30 seconds (configurable via `POLL_INTERVAL_SECONDS` environment variable).
- **Payment service test double:** The payment service remains a stub. Refunding a charge updates `status` to `refunded` in the `charges` collection. No external payment provider is called.
- **Idempotency key:** `payment_reference` is the idempotency key across checkout, payment, and order. Payment service checks `status == "refunded"` before refunding to handle duplicate `RefundRequested` messages.
- **Atomic insert and update:** Completion window checker should insert the `RefundRequested` outbox row and update the checkout status to `refund_pending` in a single MongoDB transaction (if MongoDB supports transactions; verify).

---

## End of Tech Stub
```

## Spec `specs/5.md`

# specs/5.md

**Ticket:** 5  
**Title:** Checkout status endpoint returns accepted, order created, cart cleared, or refunded with order id when present  
**Epic:** PER-278  
**Type:** Story  
**Covers:** AC7

---

## Stack

- **Language:** Python 3.12
- **Framework:** FastAPI
- **Database:** MongoDB (checkout service's own `checkout_db`)
- **Service:** Checkout service (runs on port 8004)
- **Existing collections:** `checkouts` (added in ticket 1)

---

## Schema and API Contracts

### Checkout Status Response

**New model** (add to `checkout/models.py`):

```python
class CheckoutStatusResponse(BaseModel):
    checkout_id: str
    status: Literal["accepted", "order_created", "cart_cleared", "refunded"]
    order_id: Optional[str] = None
    created_at: str  # ISO 8601 timestamp

    @model_validator(mode="after")
    def order_id_present_when_order_exists(self):
        if self.status in ("order_created", "cart_cleared") and self.order_id is None:
            raise ValueError("order_id must be present when status is order_created or cart_cleared")
        return self
```

### New Endpoint

**Route:** `GET /checkout/{checkout_id}`  
**Service:** Checkout (port 8004)  
**Summary:** Fetch checkout status  
**Description:**  
Public endpoint (no authorization in v1). [AUTH_DEFERRED] Returns the checkout status, order id (when the order has been created), and created timestamp. The status is one of: `accepted` (checkout stored, order not yet created), `order_created` (order exists but cart not yet cleared), `cart_cleared` (happy path complete), or `refunded` (order never appeared, charge refunded).

**Path Parameter:**
- `checkout_id` (string, required): The checkout id returned by `POST /checkout`

**Response (200):**
```json
{
  "checkout_id": "chk_xyz789",
  "status": "order_created",
  "order_id": "ord_def456",
  "created_at": "2025-01-10T14:30:00Z"
}
```

**Response (404):**
```json
{
  "detail": "Checkout not found"
}
```

**Response (500):**
```json
{
  "detail": "Database query failed"
}
```

---

## File Touch List

### Modified Files

1. **`checkout/models.py`**
   - Add `CheckoutStatusResponse` model (see Schema and API Contracts above)

2. **`checkout/routes.py`**
   - Add `GET /checkout/{checkout_id}` endpoint
   - Import `CheckoutStatusResponse` from `models`
   - Query `checkouts_collection` by `checkout_id`
   - Return 404 when checkout not found
   - Return 500 on database query failure
   - Convert `created_at` datetime to ISO 8601 string

### No New Files

This ticket adds one endpoint to the existing checkout service. No new Python modules are required.

---

## Risks

### R1: checkout_id is not a UUID

**Risk:** The `checkout_id` field in the `checkouts` collection may be a MongoDB ObjectId string instead of a UUID.  
**Impact:** The endpoint's path parameter validation may need to accept ObjectId hex strings.  
**Mitigation:** Inspect the `checkouts` collection schema added in ticket 1. If `checkout_id` is an ObjectId, the route must accept a 24-character hex string. If it's a UUID, validate as UUID4.  
**Status:** Verify the `checkout_id` field type in the `checkouts` collection DDL from ticket 1 before implementing this endpoint.

---

### R2: Status field may not exist in all checkout records

**Risk:** Checkouts created before the status field was added may have `status == None`.  
**Impact:** The endpoint returns incomplete data or crashes.  
**Mitigation:** Default to `"accepted"` when the status field is missing. Add a migration script to backfill `status` for existing checkouts (deferred).  
**Status:** Handle missing status gracefully in the endpoint code.

---

### R3: created_at may be absent in older records

**Risk:** Checkouts created before the `created_at` field was added have no timestamp.  
**Impact:** The endpoint cannot return a valid ISO 8601 string.  
**Mitigation:** Return `None` or a placeholder timestamp (e.g. `"1970-01-01T00:00:00Z"`) when `created_at` is missing. Add a migration script to backfill timestamps (deferred).  
**Status:** Handle missing `created_at` gracefully in the endpoint code.

---

### R4: Database query failure

**Risk:** MongoDB is unreachable or the query times out.  
**Impact:** The endpoint returns a 500 error.  
**Mitigation:** Return a fixed error message (no stack trace). [ERROR_DETAIL_DEFERRED] Log the failure for ops. [LOGGING_DEFERRED]  
**Status:** Accepted. Fail fast on database errors.

---

### R5: No authentication

**Risk:** Any client can query any checkout's status.  
**Impact:** Shoppers can see each other's checkout statuses if they know the checkout id.  
**Mitigation:** [AUTH_DEFERRED] Add authentication in a future ticket.  
**Status:** Accepted for v1. Noted as [AUTH_DEFERRED].

---

## Open Questions

1. **What is the `checkout_id` field type?** UUID or ObjectId? (See R1)
2. **Are `status` and `created_at` always present?** Or do we need to handle missing fields? (See R2, R3)
3. **Should the endpoint return `payment_reference`?** The ticket does not mention it, so it's excluded.
4. **Should refunded checkouts include a `refunded_at` timestamp?** The ticket does not require it, so it's excluded.

---

## Acceptance Criteria

### AC7: The client can see how the checkout finished

**Given** a checkout id  
**When** the client fetches that checkout  
**Then** the status is one of `accepted`, `order_created`, `cart_cleared`, or `refunded`, and an order id is present once the order exists

**Verification:**
1. Accept a checkout and obtain a checkout id (call `POST /checkout` with a valid cart id)
2. Immediately call `GET /checkout/{checkout_id}` and assert the response status is `accepted` and no `order_id` is present
3. Wait for the order to be created (order service processes `OrderRequested`)
4. Call `GET /checkout/{checkout_id}` again and assert the status is `order_created` and the `order_id` is present
5. Wait for the cart to be cleared (checkout service processes `OrderCreated` and deletes the cart)
6. Call `GET /checkout/{checkout_id}` again and assert the status is `cart_cleared` and the `order_id` is still present
7. Accept a checkout, prevent the order service from processing the message, wait for the completion window to pass
8. Call `GET /checkout/{checkout_id}` and assert the status is `refunded` and no `order_id` is present
9. Call `GET /checkout/{invalid_id}` and assert the response is 404

---

## Notes

- **[AUTH_DEFERRED]:** No authentication in v1. Any client can query any checkout.
- **[ERROR_DETAIL_DEFERRED]:** Database failures return a fixed message with no stack trace.
- **[LOGGING_DEFERRED]:** No structured logging in v1. Console output only.
- **Migration deferred:** Backfilling `status` and `created_at` for existing checkouts is not in scope for this ticket.

---

## Dependencies

- **Ticket 1:** Checkout outbox and `OrderRequested` publish via RabbitMQ (adds `checkouts` collection)
- **Ticket 3:** Checkout empties the cart on `OrderCreated` with retry (updates `status` to `cart_cleared`)
- **Ticket 4:** Refund a charge when the order never appears within the completion window (updates `status` to `refunded`)

The `checkouts` collection schema (DDL, field names, field types) must be verified in ticket 1's implementation before implementing this endpoint.

---

**End of spec.**

## Spec `specs/6.md`

# specs/6.md

**Ticket:** PER-278-6  
**Title:** Integration test suite for distributed checkout acceptance criteria  
**Parent Feature:** PER-278 — A paid checkout still becomes one order when order or cart is briefly down  
**Type:** Story  
**Covers:** AC1, AC2, AC3, AC4, AC5, AC6, AC7, AC8

---

## Job

Write an integration test suite that verifies all eight acceptance criteria of the distributed checkout feature. The suite must run against live instances of checkout, order, payment, and cart services, with RabbitMQ running, and demonstrate that payment refused stays immediate, accepted checkout returns before the order exists, orders are created after recovery, duplicate checkouts and messages are idempotent, cart deletion is retried, charges are refunded when orders never appear, the checkout status endpoint works, and no service shares another service's database.

---

## Stack

- **Language:** Python 3.12
- **Framework:** pytest for the test suite
- **Dependencies:** `httpx` (async HTTP client), `pymongo` (MongoDB client), `pika` (RabbitMQ client), `pytest-asyncio` (async test support)
- **Test file location:** `checkout/tests/test_distributed_checkout.py`
- **Existing test pattern:** See `checkout/tests/test_checkout.py`, `cart/tests/test_cart.py`, `catalog/tests/test_catalog.py`, `payment/tests/test_payment.py` for the established pytest style, MongoDB cleanup in `setup_function()`, and service URL constants

---

## Schema and API Contracts

### Checkout Service

**Endpoint:** `POST http://localhost:8004/checkout`

**Request:**
```json
{
  "cart_id": "6789...",
  "shopper_id": "507f...",
  "email": "shopper@example.com",
  "ship_to": {
    "street": "123 Main St",
    "city": "Springfield",
    "postal_code": "12345",
    "country": "US"
  }
}
```

**Response (accepted, before order exists):**
```json
{
  "checkout_id": "chk_xyz789",
  "status": "accepted"
}
```

**Response (refused):**
```json
{
  "status": "refused",
  "reason": "Card declined"
}
```

**New endpoint:** `GET http://localhost:8004/checkout/{checkout_id}`

**Response:**
```json
{
  "checkout_id": "chk_xyz789",
  "status": "accepted" | "order_created" | "cart_cleared" | "refunded",
  "order_id": "ord_def456",
  "created_at": "2025-01-10T14:30:00Z"
}
```

### Checkout Database (MongoDB)

**Database:** `checkout_db`

**Collections:**
- `checkouts` — holds checkout records with `payment_reference`, `status`, `order_id`, `created_at`, `updated_at`
- `outbox` — holds outbox rows with `event_type`, `payload`, `payment_reference`, `published`, `published_at`, `created_at`

**Schema (checkouts):**
```python
{
  "_id": ObjectId,
  "checkout_id": str,
  "payment_reference": str,
  "cart_id": str,
  "shopper_id": str | None,
  "email": str,
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

**Schema (outbox):**
```python
{
  "_id": ObjectId,
  "event_type": "OrderRequested" | "RefundRequested",
  "payload": dict,
  "payment_reference": str,
  "published": bool,
  "published_at": datetime | None,
  "created_at": datetime
}
```

### Order Service

**Endpoint:** `GET http://localhost:8006/orders/{order_id}`

**Response:**
```json
{
  "order_id": "ord_def456",
  "payment_reference": "pay_abc123",
  "shopper_id": "507f...",
  "email": "shopper@example.com",
  "ship_to": {
    "street": "123 Main St",
    "city": "Springfield",
    "postal_code": "12345",
    "country": "US"
  },
  "lines": [
    {
      "product_id": "prod_001",
      "quantity": 2,
      "price_minor_units": 1500
    }
  ],
  "total_minor_units": 3000,
  "created_at": "2025-01-10T14:30:05Z"
}
```

### Order Database (MongoDB)

**Database:** `order_db`

**Collection:** `orders`

**Schema:**
```javascript
{
  _id: ObjectId,
  order_id: string,
  payment_reference: string,
  shopper_id: string | null,
  email: string,
  ship_to: {
    street: string,
    city: string,
    postal_code: string,
    country: string
  },
  lines: [
    {
      product_id: string,
      quantity: number,
      price_minor_units: number
    }
  ],
  total_minor_units: number,
  created_at: Date
}
```

### Payment Service

**Endpoint:** `POST http://localhost:8005/charge`

**Request:**
```json
{
  "amount_minor_units": 3000,
  "card": "4111111111111111"
}
```

**Response (accepted):**
```json
{
  "status": "accepted",
  "reference": "pay_abc123"
}
```

**Response (refused):**
```json
{
  "status": "refused",
  "reason": "Card declined"
}
```

### Payment Database (MongoDB)

**Database:** `payment_db`

**Collection:** `charges`

**Schema:**
```python
{
  "_id": ObjectId,
  "reference": str,
  "amount_minor_units": int,
  "card": str | None,
  "status": "accepted" | "refused" | "refunded",
  "refunded_at": datetime | None,
  "timestamp": datetime
}
```

### Cart Service

**Endpoint:** `GET http://localhost:8003/cart/{cart_id}`

**Response:**
```json
{
  "cart_id": "6789...",
  "lines": [
    {
      "product_id": "prod_001",
      "quantity": 2,
      "price_snapshot_minor_units": 1500
    }
  ],
  "total_minor_units": 3000
}
```

**Endpoint:** `DELETE http://localhost:8003/cart/{cart_id}`

**Response:** 204 No Content

### Cart Database (MongoDB)

**Database:** `cart_db`

**Collection:** `carts`

**Schema:**
```python
{
  "_id": ObjectId,
  "cart_id": str,
  "lines": [
    {
      "product_id": str,
      "quantity": int,
      "price_snapshot_minor_units": int
    }
  ],
  "total_minor_units": int
}
```

### RabbitMQ Messages

**Exchange:** `ecom.events` (type: `topic`, durable)

**Message: OrderRequested**

**Routing key:** `order.requested`

**Payload:**
```json
{
  "event_type": "OrderRequested",
  "payment_reference": "pay_abc123",
  "checkout_id": "chk_xyz789",
  "shopper_id": "507f1f77bcf86cd799439011",
  "email": "shopper@example.com",
  "ship_to": {
    "street": "123 Main St",
    "city": "Springfield",
    "postal_code": "12345",
    "country": "US"
  },
  "lines": [
    {
      "product_id": "prod_001",
      "quantity": 2,
      "price_snapshot_minor_units": 1500
    }
  ],
  "total_minor_units": 3000,
  "timestamp": "2025-01-10T14:30:00Z"
}
```

**Message: OrderCreated**

**Routing key:** `order.created`

**Payload:**
```json
{
  "event_type": "OrderCreated",
  "payment_reference": "pay_abc123",
  "checkout_id": "chk_xyz789",
  "order_id": "ord_def456",
  "timestamp": "2025-01-10T14:30:05Z"
}
```

**Message: RefundRequested**

**Routing key:** `payment.refund.requested`

**Payload:**
```json
{
  "event_type": "RefundRequested",
  "payment_reference": "pay_abc123",
  "checkout_id": "chk_xyz789",
  "timestamp": "2025-01-10T14:35:00Z"
}
```

---

## File Touch List

### New Files

- `checkout/tests/test_distributed_checkout.py` — integration test suite for AC1–AC8

### Modified Files

- `checkout/requirements.txt` — add `pytest-asyncio` if not already present (verify in `repo-file/app/checkout/requirements.txt`)

### Files to Read (verify existing code)

- `checkout/tests/conftest.py` — pytest fixtures and setup patterns (see `repo-file/app/checkout/tests/conftest.py` — does not exist yet, so create if needed or inline setup in test file)
- `checkout/tests/test_checkout.py` — existing test patterns, MongoDB cleanup in `setup_function()`, service URL constants (see `repo-file/app/checkout/tests/test_checkout.py` — does not exist yet, so derive patterns from `cart/tests/test_cart.py`)
- `cart/tests/test_cart.py` — `setup_function()` clears `cart_db.carts`, service URL constants like `CATALOG_BASE_URL` (see `repo-outline/app`)
- `payment/tests/test_payment.py` — `setup_function()` clears `payment_db.charges`, database isolation test pattern (see `repo-outline/app`)
- `checkout/db.py` — `MONGODB_URI`, `client`, `db` references (see `repo-file/app/checkout/db.py`)
- `checkout/models.py` — `CheckoutRequest`, `CheckoutResponse`, `ShipTo` (see `repo-file/app/checkout/models.py`)
- `checkout/routes.py` — `CART_SERVICE_URL`, `PAYMENT_SERVICE_URL`, `ORDER_SERVICE_URL` (see `repo-file/app/checkout/routes.py`)
- `order/routes.js` — `POST /orders`, `GET /orders/{order_id}` (see `repo-file/app/order/routes.js`)
- `order/models.js` — `createOrderSchema`, `shipToSchema`, `orderLineSchema` (see `repo-file/app/order/models.js`)
- `payment/routes.py` — `POST /charge`, `AMOUNT_LIMIT_MINOR_UNITS` (see `repo-file/app/payment/routes.py`)
- `payment/models.py` — `ChargeRequest`, `ChargeResponse` (see `repo-file/app/payment/models.py`)

---

## Risks

### R1: RabbitMQ connection and message delivery timing

**Risk:** Tests depend on RabbitMQ being up and messages being delivered promptly. If the relay process is slow or RabbitMQ is not running, tests may fail or hang.

**Mitigation:** Document that RabbitMQ must be running on `localhost:5672` before running the test suite. Add a timeout and retry loop when polling for order creation or cart deletion (e.g. poll every 0.5s for up to 10s). If the order does not appear within the timeout, fail the test with a clear message.

### R2: Order service down scenario requires process control

**Risk:** AC3 requires stopping and restarting the order service. pytest cannot stop and start external processes unless the test harness has process control (e.g. subprocess spawning).

**Mitigation:** Document that AC3 requires manual setup: the test suite assumes the order service is not running when `test_order_created_after_recovery()` starts, and the test will instruct the operator to start it. Alternatively, use `subprocess.Popen()` to start the order service within the test, or skip AC3 in automated CI and mark it as a manual test.

**Decision for this spec:** The test suite will document that the order service must be stopped manually before running `test_order_created_after_recovery()`, and the test will poll for the order after the service is restarted. Mark this test with `@pytest.mark.manual` or a similar marker so it can be skipped in CI.

### R3: Completion window timeout requires waiting 5 minutes

**Risk:** AC6 tests that a charge is refunded when the order is not created within the completion window (default 5 minutes). Waiting 5 minutes in a test is impractical.

**Mitigation:** The completion window must be configurable via an environment variable (`COMPLETION_WINDOW_SECONDS`). Set it to 5 seconds for the test suite. Document that the test requires `COMPLETION_WINDOW_SECONDS=5` in the checkout service's environment before running the suite.

**Decision for this spec:** The test suite will document that the completion window must be set to 5 seconds for AC6 (`COMPLETION_WINDOW_SECONDS=5`), and the test will wait up to 10 seconds for the refund to appear.

### R4: Cart delete retry timing

**Risk:** AC5 tests that cart deletion is retried on failure. The retry logic (exponential backoff: 5s, 10s, 20s) means the test must wait up to 35 seconds for the cart to be deleted.

**Mitigation:** The retry backoff must be configurable via environment variables for testing (e.g. `CART_DELETE_RETRY_DELAYS=1,2,3`). Set it to `1,2,3` for the test suite (total wait: 6 seconds). Document this requirement.

**Decision for this spec:** The test suite will document that the cart delete retry delays must be set to `1,2,3` seconds for AC5 (`CART_DELETE_RETRY_DELAYS=1,2,3`), and the test will poll for the cart deletion every 1s for up to 10s.

### R5: Database isolation across services

**Risk:** AC8 tests that no service reads or writes another service's database. This requires inspecting database connections or queries, which is not feasible in a black-box integration test.

**Mitigation:** AC8 is a code inspection and monitoring test, not a runtime integration test. The test suite will verify isolation by checking that each service's database contains only the expected collections after a completed or refunded checkout. For example, after a completed checkout, `checkout_db` has `checkouts` and `outbox`, `order_db` has `orders`, `payment_db` has `charges`, and `cart_db` is empty (cart deleted). No other collections exist in any database.

**Decision for this spec:** `test_database_isolation()` will inspect the collections in each database after a completed checkout and assert that only the expected collections exist, and no cross-database writes occurred.

### R6: Duplicate message delivery simulation

**Risk:** AC4 tests that duplicate message delivery does not create two orders. Simulating duplicate RabbitMQ message delivery in a test is complex and requires either manually publishing the same message twice or mocking the RabbitMQ consumer's ack behavior.

**Mitigation:** The test will manually publish the same `OrderRequested` message twice to the RabbitMQ queue using `pika`, then poll the order service to verify that only one order exists for the payment reference.

**Decision for this spec:** `test_duplicate_message_does_not_create_two_orders()` will use `pika` to publish the same `OrderRequested` message twice, then assert that only one order exists after both messages are consumed.

---

## Open Questions

1. **How is the relay process deployed?** The test suite assumes the relay is running and polling the outbox. Should the test suite start the relay as a subprocess, or should it be a prerequisite?
   - **Decision for this spec:** Document that the relay process must be running before the test suite starts. Do not spawn it in the test.

2. **How is the completion window checker deployed?** The test suite assumes the completion window checker is running. Should the test suite start it as a subprocess, or should it be a prerequisite?
   - **Decision for this spec:** Document that the completion window checker must be running before the test suite starts (for AC6). Do not spawn it in the test.

3. **What is the MongoDB URI for the test suite?** The existing tests assume `mongodb://localhost:27017`. Should the test suite use the same URI, or a separate test database?
   - **Decision for this spec:** Use `mongodb://localhost:27017` and clear all databases (`checkout_db`, `order_db`, `payment_db`, `cart_db`, `identity_db`, `catalog_db`) in `setup_function()`, matching the existing pattern in `cart/tests/test_cart.py`.

4. **What is the RabbitMQ URI for the test suite?** The relay and consumers assume `amqp://localhost:5672`. Should the test suite use the same URI?
   - **Decision for this spec:** Yes, use `amqp://localhost:5672`.

5. **How are environment variables set for the test suite?** The test suite requires `COMPLETION_WINDOW_SECONDS=5` and `CART_DELETE_RETRY_DELAYS=1,2,3` for AC6 and AC5. Should these be set in a `.env` file, in the test suite itself, or as pytest fixtures?
   - **Decision for this spec:** Document that these environment variables must be set before running the test suite. The test suite will not set them itself. Add a check in `setup_function()` to verify they are set, and skip the relevant tests if they are not.

---

## Tests

### AC1: Payment refused stays immediate

**Test:** `test_payment_refused_stays_immediate()`

**Given:** A cart at checkout

**When:** Payment refuses the charge

**Then:** The response says refused, no order is created, the cart is unchanged, and nothing is published

**Steps:**
1. Create a cart with one line (product_id: `prod_001`, quantity: 2, price: 1500 minor units, total: 3000)
2. Call `POST /checkout` with `card: "4111111111110000"` (card ending in 0000, which payment refuses per `repo-file/app/payment/routes.py`)
3. Assert the response status is `refused` and reason is `"Card declined"`
4. Assert no checkout record exists in `checkout_db.checkouts`
5. Assert no outbox row exists in `checkout_db.outbox`
6. Assert the cart still exists in `cart_db.carts` with the same lines and total
7. Assert no order exists in `order_db.orders`
8. Assert no `OrderRequested` message was published to RabbitMQ (verify by consuming the `order.requested.queue` and asserting it is empty)

---

### AC2: Accepted checkout returns before the order exists

**Test:** `test_accepted_checkout_returns_before_order_exists()`

**Given:** A cart and a payment service that accepts

**When:** The client checks out

**Then:** The response is accepted with a checkout id while the order may not exist yet, and checkout has stored that checkout and an unpublished outbox row for it in one write

**Steps:**
1. Create a cart with one line (product_id: `prod_001`, quantity: 2, price: 1500 minor units, total: 3000)
2. Stop the relay process (or mock it to not poll the outbox) so the outbox row is not published yet
3. Call `POST /checkout` with a valid cart id and ship-to address
4. Assert the response status is `accepted` and contains a `checkout_id`
5. Assert the response does not contain an `order_id` yet (order service has not consumed the message)
6. Query `checkout_db.checkouts` by `checkout_id` and assert the checkout record exists with `status: "accepted"`, `payment_reference`, and the lines from the cart
7. Query `checkout_db.outbox` by `payment_reference` and assert an outbox row exists with `event_type: "OrderRequested"`, `published: false`, and the same `payment_reference`
8. Assert both the checkout record and the outbox row were written in a single atomic operation (verify they both exist or both are absent — this is implicitly true if both queries succeed; no explicit atomicity check is needed in the test)
9. Assert no order exists in `order_db.orders` yet

---

### AC3: Order is created after the order service recovers

**Test:** `test_order_created_after_recovery()`

**Given:** An accepted checkout whose `OrderRequested` message could not be handled because the order service was down

**When:** The order service is running again

**Then:** Exactly one order exists with the lines, total, ship-to address, and payment reference from that checkout

**Steps:**
1. Stop the order service (manual step or subprocess control)
2. Create a cart with one line (product_id: `prod_001`, quantity: 2, price: 1500 minor units, total: 3000)
3. Call `POST /checkout` with a valid cart id and ship-to address
4. Assert the response status is `accepted` and contains a `checkout_id`
5. Wait for the relay to publish the `OrderRequested` message (poll `checkout_db.outbox` until `published: true`)
6. Assert no order exists in `order_db.orders` yet (order service is down)
7. Start the order service (manual step or subprocess control)
8. Poll `order_db.orders` for up to 10 seconds until an order appears with the `payment_reference` from the checkout
9. Assert exactly one order exists for that `payment_reference`
10. Assert the order's `lines`, `total_minor_units`, `ship_to`, and `email` match the checkout's cart contents and provided address

**Marker:** `@pytest.mark.manual` (requires manual order service stop/start)

---

### AC4: A repeated checkout does not charge or order twice

**Test:** `test_repeated_checkout_does_not_charge_twice()`

**Given:** An accepted checkout for a payment reference

**When:** The same checkout is submitted again, or the same message is delivered again

**Then:** There is still one charge and one order for that reference

**Steps:**
1. Create a cart with one line (product_id: `prod_001`, quantity: 2, price: 1500 minor units, total: 3000)
2. Call `POST /checkout` with a valid cart id and ship-to address (checkout 1)
3. Assert the response status is `accepted` and contains a `checkout_id` and `payment_reference`
4. Wait for the order to be created (poll `order_db.orders` until an order appears with the `payment_reference`)
5. Submit the same checkout request again with the same `cart_id`, `email`, and `ship_to` (checkout 2)
6. Assert the response status is `accepted` and contains the same `checkout_id` (idempotency: checkout service detects duplicate by `payment_reference` and returns the existing checkout)
7. Query `payment_db.charges` by `reference` and assert only one charge exists for that `payment_reference`
8. Query `order_db.orders` by `payment_reference` and assert only one order exists for that `payment_reference`

**Test:** `test_duplicate_message_does_not_create_two_orders()`

**Given:** An accepted checkout whose `OrderRequested` message is delivered twice

**When:** The order service consumes both messages

**Then:** There is still one order for that reference

**Steps:**
1. Create a cart with one line (product_id: `prod_001`, quantity: 2, price: 1500 minor units, total: 3000)
2. Call `POST /checkout` with a valid cart id and ship-to address
3. Wait for the relay to publish the `OrderRequested` message (poll `checkout_db.outbox` until `published: true`)
4. Use `pika` to manually publish the same `OrderRequested` message to the `order.requested.queue` again (duplicate)
5. Wait for the order service to consume both messages (poll `order_db.orders` for up to 10 seconds)
6. Query `order_db.orders` by `payment_reference` and assert only one order exists for that `payment_reference`

---

### AC5: The cart is emptied, and a failed delete is retried

**Test:** `test_cart_emptied_on_order_created()`

**Given:** An order created for a checkout

**When:** Deleting the cart succeeds

**Then:** The cart is removed

**Steps:**
1. Create a cart with one line (product_id: `prod_001`, quantity: 2, price: 1500 minor units, total: 3000)
2. Call `POST /checkout` with a valid cart id and ship-to address
3. Wait for the order to be created (poll `order_db.orders` until an order appears)
4. Wait for the cart to be deleted (poll `cart_db.carts` for up to 10 seconds until the cart no longer exists)
5. Assert `GET /cart/{cart_id}` returns 404

**Test:** `test_cart_delete_retried_on_failure()`

**Given:** An order created for a checkout

**When:** Deleting the cart fails (cart service is down or returns 500)

**Then:** The cart is still removed on a later attempt, and the order is not created again

**Steps:**
1. Create a cart with one line (product_id: `prod_001`, quantity: 2, price: 1500 minor units, total: 3000)
2. Call `POST /checkout` with a valid cart id and ship-to address
3. Wait for the order to be created (poll `order_db.orders` until an order appears)
4. Stop the cart service (manual step or subprocess control) before the first cart delete attempt
5. Wait 2 seconds (first retry delay)
6. Assert the cart still exists in `cart_db.carts`
7. Start the cart service (manual step or subprocess control)
8. Poll `cart_db.carts` for up to 10 seconds until the cart no longer exists (retry succeeds)
9. Assert `GET /cart/{cart_id}` returns 404
10. Assert only one order exists for the `payment_reference` (order is not created again)

**Marker:** `@pytest.mark.manual` (requires manual cart service stop/start)

**Alternative (without stopping the cart service):** Mock the cart service to return 500 on the first `DELETE /cart/{cart_id}` call, then 204 on the second call. This requires starting a mock cart service on port 8003 or modifying the checkout service's `CART_SERVICE_URL` to point to the mock. For simplicity, mark this test as manual and document the cart service stop/start requirement.

---

### AC6: A charge with no order is refunded

**Test:** `test_charge_refunded_when_order_never_appears()`

**Given:** An accepted charge whose order has not been created within the completion window

**When:** That window passes

**Then:** The charge is refunded, no order is created, and the checkout status says refunded

**Steps:**
1. Set `COMPLETION_WINDOW_SECONDS=5` in the checkout service's environment before running this test (verify in `setup_function()` and skip the test if not set)
2. Create a cart with one line (product_id: `prod_001`, quantity: 2, price: 1500 minor units, total: 3000)
3. Stop the order service (manual step or subprocess control) so the order is never created
4. Call `POST /checkout` with a valid cart id and ship-to address
5. Assert the response status is `accepted` and contains a `checkout_id` and `payment_reference`
6. Wait for the relay to publish the `OrderRequested` message (poll `checkout_db.outbox` until `published: true`)
7. Wait 10 seconds (5-second completion window + 5-second buffer for the completion window checker to run and publish `RefundRequested`)
8. Poll `checkout_db.outbox` for a `RefundRequested` row with the same `payment_reference` (up to 10 seconds)
9. Assert the `RefundRequested` outbox row exists and is published
10. Poll `payment_db.charges` for up to 10 seconds until the charge status is `refunded`
11. Call `GET /checkout/{checkout_id}` and assert the status is `refunded`
12. Assert no order exists in `order_db.orders` for that `payment_reference`

**Marker:** `@pytest.mark.manual` (requires `COMPLETION_WINDOW_SECONDS=5` and order service stopped)

---

### AC7: The client can see how the checkout finished

**Test:** `test_checkout_status_endpoint()`

**Given:** A checkout id

**When:** The client fetches that checkout

**Then:** The status is one of accepted, order created, cart cleared, or refunded, and an order id is present once the order exists

**Steps:**
1. Create a cart with one line (product_id: `prod_001`, quantity: 2, price: 1500 minor units, total: 3000)
2. Call `POST /checkout` with a valid cart id and ship-to address
3. Assert the response status is `accepted` and contains a `checkout_id`
4. Call `GET /checkout/{checkout_id}` immediately
5. Assert the response contains `status: "accepted"` and no `order_id` yet
6. Wait for the order to be created (poll `order_db.orders` until an order appears)
7. Call `GET /checkout/{checkout_id}` again
8. Assert the response contains `status: "order_created"` and the `order_id`
9. Wait for the cart to be deleted (poll `cart_db.carts` until the cart no longer exists)
10. Call `GET /checkout/{checkout_id}` again
11. Assert the response contains `status: "cart_cleared"` and the `order_id`

**Test:** `test_checkout_status_refunded()`

**Given:** A checkout that was refunded

**When:** The client fetches that checkout

**Then:** The status is refunded and no order id is present

**Steps:**
1. Create a cart with one line (product_id: `prod_001`, quantity: 2, price: 1500 minor units, total: 3000)
2. Stop the order service (manual step or subprocess control)
3. Call `POST /checkout` with a valid cart id and ship-to address
4. Wait for the completion window to pass and the refund to be processed (see AC6 steps 6–10)
5. Call `GET /checkout/{checkout_id}`
6. Assert the response contains `status: "refunded"` and no `order_id`

**Marker:** `@pytest.mark.manual` (requires `COMPLETION_WINDOW_SECONDS=5` and order service stopped)

---

### AC8: Services do not share databases

**Test:** `test_database_isolation()`

**Given:** A completed or refunded checkout

**When:** Each service is inspected

**Then:** Checkout, order, payment, and cart each read and write only their own data, and the only new cross-service path is RabbitMQ

**Steps:**
1. Create a cart with one line (product_id: `prod_001`, quantity: 2, price: 1500 minor units, total: 3000)
2. Call `POST /checkout` with a valid cart id and ship-to address
3. Wait for the order to be created and the cart to be deleted (poll until complete)
4. Inspect `checkout_db.checkouts` and assert it contains the checkout record
5. Inspect `checkout_db.outbox` and assert it contains the outbox rows (`OrderRequested`)
6. Assert no other collections exist in `checkout_db`
7. Inspect `order_db.orders` and assert it contains the order record
8. Assert no other collections exist in `order_db`
9. Inspect `payment_db.charges` and assert it contains the charge record
10. Assert no other collections exist in `payment_db`
11. Inspect `cart_db.carts` and assert the cart no longer exists (deleted)
12. Assert no other collections exist in `cart_db` (or only the empty `carts` collection)
13. Assert no service wrote to another service's database (this is implicitly verified by steps 6, 8, 10, 12 — each database contains only its own collections)

**Alternative (for a refunded checkout):**
1. Create a cart, stop the order service, call `POST /checkout`, wait for the refund (see AC6)
2. Inspect `checkout_db.checkouts` and assert it contains the checkout record with `status: "refunded"`
3. Inspect `checkout_db.outbox` and assert it contains the outbox rows (`OrderRequested`, `RefundRequested`)
4. Assert no other collections exist in `checkout_db`
5. Inspect `order_db.orders` and assert no order exists
6. Assert no other collections exist in `order_db`
7. Inspect `payment_db.charges` and assert the charge status is `refunded`
8. Assert no other collections exist in `payment_db`
9. Inspect `cart_db.carts` and assert the cart still exists (not deleted when refunded)
10. Assert no other collections exist in `cart_db`

---

## Example Test Code Structure

```python
# checkout/tests/test_distributed_checkout.py

import os
import time
from pathlib import Path

import httpx
import pika
import pytest
from bson import ObjectId
from pymongo import MongoClient

# Service URLs (match checkout/routes.py constants)
CHECKOUT_URL = "http://localhost:8004"
CART_URL = "http://localhost:8003"
PAYMENT_URL = "http://localhost:8005"
ORDER_URL = "http://localhost:8006"

# MongoDB connection
MONGODB_URI = os.environ.get("MONGODB_URI", "mongodb://localhost:27017")
client = MongoClient(MONGODB_URI)

# RabbitMQ connection
RABBITMQ_URI = "amqp://localhost:5672"

# Databases
checkout_db = client["checkout_db"]
order_db = client["order_db"]
payment_db = client["payment_db"]
cart_db = client["cart_db"]
identity_db = client["identity_db"]
catalog_db = client["catalog_db"]

OTHER_DATABASES = ("identity_db", "catalog_db")


def setup_function():
    """Clear all databases before each test."""
    for db_name in [
        "checkout_db",
        "order_db",
        "payment_db",
        "cart_db",
        "identity_db",
        "catalog_db",
    ]:
        db = client[db_name]
        for collection_name in db.list_collection_names():
            db[collection_name].delete_many({})

    # Verify environment variables for time-sensitive tests
    completion_window = os.environ.get("COMPLETION_WINDOW_SECONDS")
    retry_delays = os.environ.get("CART_DELETE_RETRY_DELAYS")
    if completion_window is None:
        pytest.skip("COMPLETION_WINDOW_SECONDS not set (required for AC6)")
    if retry_delays is None:
        pytest.skip("CART_DELETE_RETRY_DELAYS not set (required for AC5)")


def _create_cart(quantity: int = 2, price: int = 1500) -> str:
    """Create a cart with one line and return the cart_id."""
    # Seed a cart directly in cart_db (bypass cart service for simplicity)
    cart_id = str(ObjectId())
    cart_db.carts.insert_one(
        {
            "_id": ObjectId(),
            "cart_id": cart_id,
            "lines": [
                {
                    "product_id": "prod_001",
                    "quantity": quantity,
                    "price_snapshot_minor_units": price,
                }
            ],
            "total_minor_units": quantity * price,
        }
    )
    return cart_id


def _ship_to():
    """Return a valid ship_to address."""
    return {
        "street": "123 Main St",
        "city": "Springfield",
        "postal_code": "12345",
        "country": "US",
    }


def _poll_until(condition_fn, timeout_seconds=10, poll_interval=0.5):
    """Poll until condition_fn() returns True or timeout is reached."""
    start = time.time()
    while time.time() - start < timeout_seconds:
        if condition_fn():
            return True
        time.sleep(poll_interval)
    return False


@pytest.mark.asyncio
async def test_payment_refused_stays_immediate():
    """AC1: Payment refused stays immediate."""
    cart_id = _create_cart()

    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{CHECKOUT_URL}/checkout",
            json={
                "cart_id": cart_id,
                "email": "guest@example.com",
                "ship_to": _ship_to(),
                "card": "4111111111110000",  # Card ending in 0000 → refused
            },
        )

    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "refused"
    assert data["reason"] == "Card declined"
    assert "checkout_id" not in data

    # No checkout record
    assert checkout_db.checkouts.count_documents({}) == 0

    # No outbox row
    assert checkout_db.outbox.count_documents({}) == 0

    # Cart unchanged
    cart = cart_db.carts.find_one({"cart_id": cart_id})
    assert cart is not None
    assert cart["total_minor_units"] == 3000

    # No order
    assert order_db.orders.count_documents({}) == 0

    # No message published (verify queue is empty)
    # (This requires consuming the queue or checking RabbitMQ management API)
    # For simplicity, skip RabbitMQ queue check in this example


@pytest.mark.asyncio
async def test_accepted_checkout_returns_before_order_exists():
    """AC2: Accepted checkout returns before the order exists."""
    cart_id = _create_cart()

    # (Assume relay is not polling yet, or stop it manually)

    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{CHECKOUT_URL}/checkout",
            json={
                "cart_id": cart_id,
                "email": "guest@example.com",
                "ship_to": _ship_to(),
            },
        )

    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "accepted"
    assert "checkout_id" in data
    assert "order_id" not in data  # Order not created yet

    checkout_id = data["checkout_id"]

    # Checkout record exists
    checkout = checkout_db.checkouts.find_one({"checkout_id": checkout_id})
    assert checkout is not None
    assert checkout["status"] == "accepted"
    assert "payment_reference" in checkout

    payment_reference = checkout["payment_reference"]

    # Outbox row exists
    outbox_row = checkout_db.outbox.find_one({"payment_reference": payment_reference})
    assert outbox_row is not None
    assert outbox_row["event_type"] == "OrderRequested"
    assert outbox_row["published"] == False

    # No order yet
    assert order_db.orders.count_documents({"payment_reference": payment_reference}) == 0


# (Add remaining tests following the same pattern)
```

---

## End of Spec

---

# Story summaries

- `01-app`: LLD — Story 01-app
- `02-app`: LLD — Story 02-app
- `03-app`: LLD — Story 03-app
- `04-app`: LLD — Story 04-app
- `05-app`: LLD — Story 05-app
- `06-app`: LLD — Story 06-app

---

## Why this order

Follow breakdown.md dependencies; tickets listed above match scrum→Linear story order.
