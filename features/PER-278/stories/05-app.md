# LLD — Story 05-app

**Story:** Checkout status endpoint returns accepted, order created, cart cleared, or refunded with order id when present  
**Repo:** `app`  
**Type:** Implementation  
**Covers AC:** AC7

---

## Acceptance Criteria

### AC7: The client can see how the checkout finished

**Given** a checkout id  
**When** the client fetches that checkout  
**Then** the status is one of `accepted`, `order_created`, `cart_cleared`, or `refunded`, and an order id is present once the order exists

---

## File Plan

### New Files

None.

---

### Modified Files

1. **`checkout/models.py`**
   - Add `CheckoutStatusResponse` model (after line 32, after `CheckoutResponse`)
   - Fields: `checkout_id: str`, `status: Literal["accepted", "order_created", "cart_cleared", "refunded"]`, `order_id: Optional[str]`, `created_at: str` (ISO 8601 timestamp)
   - Validation: `@model_validator` to enforce `order_id` is present when status is `order_created` or `cart_cleared`
   - Tests: `test_checkout_status_accepted()`, `test_checkout_status_order_created()`, `test_checkout_status_cart_cleared()`, `test_checkout_status_refunded()`

2. **`checkout/routes.py`**
   - Add `GET /checkout/{checkout_id}` endpoint (after line 120, after the `POST /checkout` endpoint)
   - Import `CheckoutStatusResponse` from `models`
   - Import `checkouts_collection` from `db` (verify this was added in Story 1; if not, add `from db import checkouts_collection` at the top)
   - Query `checkouts_collection.find_one({"checkout_id": checkout_id})`
   - Return 404 if checkout not found
   - Return 500 on database query failure (catch `PyMongoError`)
   - Convert `created_at` datetime to ISO 8601 string (`checkout["created_at"].isoformat()`)
   - Return `CheckoutStatusResponse` with `checkout_id`, `status`, `order_id` (if present), and `created_at`
   - Tests: `test_checkout_status_accepted()`, `test_checkout_status_order_created()`, `test_checkout_status_cart_cleared()`, `test_checkout_status_refunded()`

---

## Interfaces

### Database Schema: `checkouts` collection (read-only for this story)

**Collection:** `checkout_db.checkouts`

**Document:**
```python
{
  "_id": ObjectId,
  "checkout_id": str,  # UUID
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

**Read Operation (this story):**
- Query by `checkout_id`, return `checkout_id`, `status`, `order_id` (if present), and `created_at`

---

### HTTP API: `GET /checkout/{checkout_id}` (new endpoint)

**Route:** `GET /checkout/{checkout_id}`  
**Service:** Checkout (port 8004)  
**Summary:** Fetch checkout status  
**Description:**  
Public endpoint (no authorization in v1). [AUTH_DEFERRED] Returns the checkout status, order id (when the order has been created), and created timestamp. The status is one of: `accepted` (checkout stored, order not yet created), `order_created` (order exists but cart not yet cleared), `cart_cleared` (happy path complete), or `refunded` (order never appeared, charge refunded).

**Path Parameter:**
- `checkout_id` (string, required): The checkout id returned by `POST /checkout` (UUID format)

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

### Python Model: `CheckoutStatusResponse`

**File:** `checkout/models.py`

**Definition:**
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

**Field Notes:**
- `checkout_id` (string, required): The checkout id (UUID).
- `status` (Literal, required): One of `accepted`, `order_created`, `cart_cleared`, or `refunded`.
- `order_id` (string, optional): Present when the order has been created (status is `order_created` or `cart_cleared`). Absent when status is `accepted` or `refunded`.
- `created_at` (string, required): ISO 8601 timestamp of when the checkout was created.

**Validation:**
- If `status` is `order_created` or `cart_cleared`, `order_id` must be present. Otherwise, raise a `ValueError`.

---

## Algorithm Notes

### Query Checkout by checkout_id

**Goal:** Fetch a checkout record from the database by `checkout_id` and return its status and order id (if present).

**Algorithm:**
1. Query `checkouts_collection.find_one({"checkout_id": checkout_id})`
2. If checkout not found (result is `None`):
   - Raise `HTTPException(status_code=404, detail="Checkout not found")`
3. If database query fails (catch `PyMongoError`):
   - Log the error (verify logging pattern in `repo-file/app/checkout/routes.py`)
   - Raise `HTTPException(status_code=500, detail="Database query failed")`
4. Extract fields from the checkout document:
   - `checkout_id = checkout["checkout_id"]`
   - `status = checkout.get("status", "accepted")` (default to `"accepted"` if missing)
   - `order_id = checkout.get("order_id")` (may be `None`)
   - `created_at = checkout.get("created_at")`
   - If `created_at` is `None`, use placeholder `"1970-01-01T00:00:00Z"`
   - Otherwise, convert to ISO 8601: `created_at_str = created_at.isoformat()`
5. Return `CheckoutStatusResponse(checkout_id=checkout_id, status=status, order_id=order_id, created_at=created_at_str)`

**Code:**
```python
from fastapi import HTTPException
from pymongo.errors import PyMongoError

from db import checkouts_collection
from models import CheckoutStatusResponse

@router.get(
    "/checkout/{checkout_id}",
    response_model=CheckoutStatusResponse,
    summary="Fetch checkout status",
    description=(
        "Public endpoint (no authorization in v1). [AUTH_DEFERRED] "
        "Returns the checkout status, order id (when the order has been created), and created timestamp. "
        "The status is one of: accepted (checkout stored, order not yet created), "
        "order_created (order exists but cart not yet cleared), "
        "cart_cleared (happy path complete), or refunded (order never appeared, charge refunded)."
    ),
    responses={
        404: {
            "description": "Checkout not found",
            "content": {
                "application/json": {"example": {"detail": "Checkout not found"}}
            },
        },
        500: {
            "description": "Database query failed",
            "content": {
                "application/json": {"example": {"detail": "Database query failed"}}
            },
        },
    },
)
def get_checkout_status(checkout_id: str) -> CheckoutStatusResponse:
    try:
        checkout = checkouts_collection.find_one({"checkout_id": checkout_id})
    except PyMongoError:
        # [ERROR_DETAIL_DEFERRED] fixed message; the response has no stack trace.
        print(f"[ERROR_DETAIL_DEFERRED] Database query failed for checkout_id {checkout_id}")
        raise HTTPException(status_code=500, detail="Database query failed") from None

    if checkout is None:
        raise HTTPException(status_code=404, detail="Checkout not found")

    status = checkout.get("status", "accepted")  # Default to "accepted" if missing
    
    created_at = checkout.get("created_at")
    if created_at is None:
        # [BACKFILL_DEFERRED] created_at is missing for checkouts created before Story 1
        created_at_str = "1970-01-01T00:00:00Z"
    else:
        created_at_str = created_at.isoformat()

    return CheckoutStatusResponse(
        checkout_id=checkout["checkout_id"],
        status=status,
        order_id=checkout.get("order_id"),
        created_at=created_at_str
    )
```

---

### Validation: order_id Required When Order Exists

**Goal:** Enforce that `order_id` is present in the response when the status indicates the order has been created.

**Algorithm:**
1. In `CheckoutStatusResponse`, define a `@model_validator(mode="after")` method
2. Check if `self.status` is in `("order_created", "cart_cleared")`
3. If yes, assert `self.order_id is not None`
4. If assertion fails, raise `ValueError("order_id must be present when status is order_created or cart_cleared")`
5. Return `self`

**Code:**
```python
from pydantic import model_validator

class CheckoutStatusResponse(BaseModel):
    checkout_id: str
    status: Literal["accepted", "order_created", "cart_cleared", "refunded"]
    order_id: Optional[str] = None
    created_at: str

    @model_validator(mode="after")
    def order_id_present_when_order_exists(self):
        if self.status in ("order_created", "cart_cleared") and self.order_id is None:
            raise ValueError("order_id must be present when status is order_created or cart_cleared")
        return self
```

---

### Edge Case: checkout_id Not a UUID

**Risk:** The `checkout_id` path parameter may not be a valid UUID (e.g. a MongoDB ObjectId hex string, or arbitrary text).

**Current Assumption:** Checkout generates `checkout_id` as a UUID (see Story 1 `checkout/routes.py` where `checkout_id = str(uuid.uuid4())`). The endpoint does not validate the UUID format; it queries the database by the provided `checkout_id` string. If the checkout is not found, a 404 is returned.

**Decision:** No UUID validation in v1. Any string is accepted as `checkout_id`. If it does not match a checkout in the database, a 404 is returned. This is consistent with the existing pattern in `cart/routes.py` where `cart_id` is a string and no validation is performed.

**Code (no change needed):**
```python
def get_checkout_status(checkout_id: str) -> CheckoutStatusResponse:
    # No UUID validation; query by the provided string
    checkout = checkouts_collection.find_one({"checkout_id": checkout_id})
    if checkout is None:
        raise HTTPException(status_code=404, detail="Checkout not found")
    # ...
```

---

### Edge Case: created_at Field Missing in Old Checkouts

**Risk:** Checkouts created before Story 1 may not have a `created_at` field.

**Current Assumption:** Story 1 adds `created_at` to all new checkouts. Old checkouts (if any exist) do not have this field.

**Mitigation:** Handle missing `created_at` gracefully by returning a placeholder timestamp (e.g. `"1970-01-01T00:00:00Z"`).

**Decision:** Return a placeholder timestamp if `created_at` is missing. Add a note that backfilling `created_at` for existing checkouts is deferred.

**Code:**
```python
created_at = checkout.get("created_at")
if created_at is None:
    # [BACKFILL_DEFERRED] created_at is missing for checkouts created before Story 1
    created_at_str = "1970-01-01T00:00:00Z"
else:
    created_at_str = created_at.isoformat()

return CheckoutStatusResponse(
    checkout_id=checkout["checkout_id"],
    status=checkout["status"],
    order_id=checkout.get("order_id"),
    created_at=created_at_str
)
```

---

### Edge Case: status Field Missing in Old Checkouts

**Risk:** Checkouts created before Story 1 may not have a `status` field.

**Current Assumption:** Story 1 adds `status` to all new checkouts. Old checkouts (if any exist) do not have this field.

**Mitigation:** Default to `"accepted"` if `status` is missing (most likely state for an old checkout).

**Decision:** Return `status = "accepted"` if the field is missing. Add a note that backfilling `status` for existing checkouts is deferred.

**Code:**
```python
status = checkout.get("status", "accepted")  # Default to "accepted" if missing

return CheckoutStatusResponse(
    checkout_id=checkout["checkout_id"],
    status=status,
    order_id=checkout.get("order_id"),
    created_at=created_at_str
)
```

---

## Tests

### Test 1: `test_checkout_status_accepted()`

**Covers:** AC7 (status is `accepted` before order is created)

**Setup:**
- Seed a checkout in `checkouts_collection` with `status: "accepted"` and no `order_id`

**Code:**
```python
from datetime import datetime
from bson import ObjectId
from fastapi.testclient import TestClient

from db import checkouts_collection
from main import app

client = TestClient(app)

def test_checkout_status_accepted():
    checkouts_collection.delete_many({})

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
        "created_at": datetime(2025, 1, 10, 14, 30, 0),
        "updated_at": datetime(2025, 1, 10, 14, 30, 0)
    }
    checkouts_collection.insert_one(checkout)

    response = client.get("/checkout/chk_test_1")

    assert response.status_code == 200
    body = response.json()
    assert body["checkout_id"] == "chk_test_1"
    assert body["status"] == "accepted"
    assert body["order_id"] is None
    assert body["created_at"] == "2025-01-10T14:30:00"
```

---

### Test 2: `test_checkout_status_order_created()`

**Covers:** AC7 (status is `order_created` and order id is present)

**Code:**
```python
def test_checkout_status_order_created():
    checkouts_collection.delete_many({})

    checkout = {
        "_id": ObjectId(),
        "checkout_id": "chk_test_2",
        "payment_reference": "pay_test_2",
        "cart_id": "cart_test_2",
        "email": "guest@example.com",
        "ship_to": {"street": "123 Main St", "city": "Springfield", "postal_code": "12345", "country": "US"},
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000,
        "status": "order_created",
        "order_id": "ord_test_1",
        "created_at": datetime(2025, 1, 10, 14, 30, 5),
        "updated_at": datetime(2025, 1, 10, 14, 30, 5)
    }
    checkouts_collection.insert_one(checkout)

    response = client.get("/checkout/chk_test_2")

    assert response.status_code == 200
    body = response.json()
    assert body["checkout_id"] == "chk_test_2"
    assert body["status"] == "order_created"
    assert body["order_id"] == "ord_test_1"
    assert body["created_at"] == "2025-01-10T14:30:05"
```

---

### Test 3: `test_checkout_status_cart_cleared()`

**Covers:** AC7 (status is `cart_cleared` and order id is present)

**Code:**
```python
def test_checkout_status_cart_cleared():
    checkouts_collection.delete_many({})

    checkout = {
        "_id": ObjectId(),
        "checkout_id": "chk_test_3",
        "payment_reference": "pay_test_3",
        "cart_id": "cart_test_3",
        "email": "guest@example.com",
        "ship_to": {"street": "123 Main St", "city": "Springfield", "postal_code": "12345", "country": "US"},
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000,
        "status": "cart_cleared",
        "order_id": "ord_test_2",
        "created_at": datetime(2025, 1, 10, 14, 30, 10),
        "updated_at": datetime(2025, 1, 10, 14, 30, 15)
    }
    checkouts_collection.insert_one(checkout)

    response = client.get("/checkout/chk_test_3")

    assert response.status_code == 200
    body = response.json()
    assert body["checkout_id"] == "chk_test_3"
    assert body["status"] == "cart_cleared"
    assert body["order_id"] == "ord_test_2"
    assert body["created_at"] == "2025-01-10T14:30:10"
```

---

### Test 4: `test_checkout_status_refunded()`

**Covers:** AC7 (status is `refunded` and no order id is present)

**Code:**
```python
def test_checkout_status_refunded():
    checkouts_collection.delete_many({})

    checkout = {
        "_id": ObjectId(),
        "checkout_id": "chk_test_4",
        "payment_reference": "pay_test_4",
        "cart_id": "cart_test_4",
        "email": "guest@example.com",
        "ship_to": {"street": "123 Main St", "city": "Springfield", "postal_code": "12345", "country": "US"},
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000,
        "status": "refunded",
        "order_id": None,
        "created_at": datetime(2025, 1, 10, 14, 35, 0),
        "updated_at": datetime(2025, 1, 10, 14, 40, 0)
    }
    checkouts_collection.insert_one(checkout)

    response = client.get("/checkout/chk_test_4")

    assert response.status_code == 200
    body = response.json()
    assert body["checkout_id"] == "chk_test_4"
    assert body["status"] == "refunded"
    assert body["order_id"] is None
    assert body["created_at"] == "2025-01-10T14:35:00"
```

---

### Test 5: `test_checkout_not_found()`

**Covers:** 404 error when checkout does not exist

**Code:**
```python
def test_checkout_not_found():
    checkouts_collection.delete_many({})

    response = client.get("/checkout/chk_missing")

    assert response.status_code == 404
    assert response.json() == {"detail": "Checkout not found"}
```

---

### Test 6: `test_checkout_status_missing_created_at()`

**Covers:** Edge case where `created_at` is missing (old checkouts)

**Code:**
```python
def test_checkout_status_missing_created_at():
    checkouts_collection.delete_many({})

    checkout = {
        "_id": ObjectId(),
        "checkout_id": "chk_old",
        "payment_reference": "pay_old",
        "cart_id": "cart_old",
        "email": "guest@example.com",
        "ship_to": {"street": "123 Main St", "city": "Springfield", "postal_code": "12345", "country": "US"},
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000,
        "status": "accepted",
        "order_id": None,
        "updated_at": datetime(2025, 1, 10, 14, 30, 0)
        # No created_at field
    }
    checkouts_collection.insert_one(checkout)

    response = client.get("/checkout/chk_old")

    assert response.status_code == 200
    body = response.json()
    assert body["checkout_id"] == "chk_old"
    assert body["status"] == "accepted"
    assert body["created_at"] == "1970-01-01T00:00:00Z"  # Placeholder
```

---

### Test 7: `test_checkout_status_missing_status_field()`

**Covers:** Edge case where `status` is missing (old checkouts)

**Code:**
```python
def test_checkout_status_missing_status_field():
    checkouts_collection.delete_many({})

    checkout = {
        "_id": ObjectId(),
        "checkout_id": "chk_old_status",
        "payment_reference": "pay_old_status",
        "cart_id": "cart_old_status",
        "email": "guest@example.com",
        "ship_to": {"street": "123 Main St", "city": "Springfield", "postal_code": "12345", "country": "US"},
        "lines": [{"product_id": "prod_001", "quantity": 1, "price_snapshot_minor_units": 1000}],
        "total_minor_units": 1000,
        "order_id": None,
        "created_at": datetime(2025, 1, 10, 14, 30, 0),
        "updated_at": datetime(2025, 1, 10, 14, 30, 0)
        # No status field
    }
    checkouts_collection.insert_one(checkout)

    response = client.get("/checkout/chk_old_status")

    assert response.status_code == 200
    body = response.json()
    assert body["checkout_id"] == "chk_old_status"
    assert body["status"] == "accepted"  # Default
    assert body["order_id"] is None
```

---

**End of LLD.**
