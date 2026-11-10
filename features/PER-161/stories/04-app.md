# LLD — Story 04-app: Checkout Service — Orchestrate cart → payment → order

**Repo:** `app`

---

## Acceptance Criteria (from requirements and breakdown)

### AC-5: Guest checkout

**Given** a cart and no account

**When** the client checks out with a contact email and a ship-to address

**Then** checkout asks payment for the cart total just returned

---

### AC-6: Signed-in checkout uses the same total

**Given** a signed-in shopper with a cart

**When** the client checks out

**Then** the amount sent to payment equals the cart total, and the order is stored against that shopper id

---

### AC-7: Payment refused

**Given** a cart at checkout

**When** payment refuses the charge

**Then** no order is created, the cart is unchanged, and the response says the payment was refused

---

### AC-8: Payment accepted

**Given** a cart at checkout

**When** payment accepts the charge

**Then** an order exists with the lines, the total charged, a payment reference, and the ship-to address, and the cart is empty

---

### AC-10: Services do not share databases

**Given** a completed purchase

**When** each service is inspected

**Then** catalog, cart, checkout, payment, order, and identity each read and write only their own data

*(Checkout owns `checkout_db`; no other service reads or writes it.)*

---

### AC-11: Swagger for every service

**Given** a running service

**When** a person opens that service's Swagger UI

**Then** every endpoint for that service is listed and can be called from the page

*(Checkout exposes `/docs`.)*

---

## File Plan

All paths under repo root `/home/goos/projects/ecom`. The repo tree is currently empty.

**New files:**

- `app/checkout/main.py` — FastAPI application entry point, enables Swagger UI at `/docs`
- `app/checkout/requirements.txt` — Python dependencies: `fastapi`, `uvicorn`, `pymongo`, `pydantic`, `httpx`
- `app/checkout/models.py` — Pydantic models for request/response validation
- `app/checkout/routes.py` — API route handler for `/checkout`
- `app/checkout/db.py` — MongoDB connection to `checkout_db`

**Directory structure:**

```
app/
└── checkout/
    ├── main.py
    ├── requirements.txt
    ├── models.py
    ├── routes.py
    └── db.py
```

---

## Interfaces

### Pydantic Models (`models.py`)

#### `ShipTo`
**Fields:**
- `street: str` — street address
- `city: str` — city name
- `postal_code: str` — postal/zip code
- `country: str` — country name

**Validation:**
- All fields must be non-empty strings

---

#### `CheckoutRequest`
**Fields:**
- `cart_id: str` — cart identifier (required)
- `shopper_id: Optional[str]` — shopper ID from identity service (for signed-in users)
- `email: Optional[str]` — contact email (required for guest checkout)
- `ship_to: ShipTo` — shipping address (required)

**Validation:**
- `cart_id` must be non-empty
- **Guest checkout:** when `shopper_id` is absent, `email` must be present and valid (EmailStr)
- **Signed-in checkout:** when `shopper_id` is present, `email` is optional
- `ship_to` is always required

---

#### `CheckoutResponse`
**Fields:**
- `order_id: Optional[str]` — order identifier (present when payment accepted)
- `status: str` — `"paid"` or `"refused"`
- `reason: Optional[str]` — refusal reason (present when payment refused)

**Validation:**
- `status` must be one of `["paid", "refused"]`
- When `status == "paid"`, `order_id` must be present
- When `status == "refused"`, `reason` must be present

---

### API Routes (`routes.py`)

#### `POST /checkout`
**Request body:** `CheckoutRequest`

**Response:** `CheckoutResponse` (200 OK), `400 Bad Request`, or `500 Internal Server Error`

**Logic:**

1. **Validate request:**
   - If `shopper_id` is absent and `email` is absent, return `400 Bad Request` with `{detail: "Email is required for guest checkout"}`.
   - If `ship_to` is missing or incomplete, return `422 Unprocessable Entity` (Pydantic validation).

2. **Get cart total and lines:**
   - Call `GET http://localhost:8003/cart/{cart_id}` using `httpx.AsyncClient`.
   - If cart service returns 404, return `400 Bad Request` with `{detail: "Cart not found"}`.
   - If cart service returns non-200, return `500 Internal Server Error` with `{detail: "Cart service unavailable"}`.
   - Parse response: `cart_response = {cart_id, lines, total_minor_units}`.

3. **Charge payment:**
   - Call `POST http://localhost:8005/charge` with `{amount_minor_units: cart_response["total_minor_units"]}`.
   - If payment service returns non-200, return `500 Internal Server Error` with `{detail: "Payment service unavailable"}`.
   - Parse response: `payment_response = {status, reference?, reason?}`.

4. **If payment refused (`payment_response["status"] == "refused"`):**
   - Return `CheckoutResponse(status="refused", reason=payment_response["reason"])`.
   - Cart remains unchanged (no delete call).

5. **If payment accepted (`payment_response["status"] == "accepted"`):**
   - **Create order:**
     - Call `POST http://localhost:8006/orders` with:
       ```json
       {
         "shopper_id": CheckoutRequest.shopper_id (if present),
         "email": CheckoutRequest.email (if present),
         "ship_to": CheckoutRequest.ship_to,
         "lines": cart_response["lines"],
         "total_minor_units": cart_response["total_minor_units"],
         "payment_reference": payment_response["reference"]
       }
       ```
     - If order service returns non-201, return `500 Internal Server Error` with `{detail: "Order service unavailable"}`. **Known gap:** payment succeeded but order not created. Mark with `[KNOWN_GAP]`.
     - Parse response: `order_response = {order_id}`.
   
   - **Empty cart:**
     - Call `DELETE http://localhost:8003/cart/{cart_id}`.
     - If cart delete fails (non-204), log the error but do NOT fail the checkout (order is already created). Mark with `[KNOWN_GAP]`.
   
   - Return `CheckoutResponse(order_id=order_response["order_id"], status="paid")`.

**Error cases:**
- Cart not found → 400
- Invalid checkout request (missing email for guest) → 400 or 422
- Cart, payment, or order service unavailable → 500
- Payment refused → 200 with `{status: "refused", reason: "..."}`

---

### MongoDB Schema (`db.py`)

**Database:** `checkout_db`

**Collection:** None required in v1 (checkout is stateless).

**Future:** May use `checkout_db.sessions` for idempotency keys or audit logs.

**Connection:**
- MongoDB URI from environment variable `MONGODB_URI` (default: `mongodb://localhost:27017`)
- Database name: `checkout_db`
- No collections referenced in v1 (db object exported for future use)

---

## Algorithm Notes

### Checkout flow (`POST /checkout`)

```
1. Parse request body → CheckoutRequest(cart_id, shopper_id?, email?, ship_to)

2. # Validate guest vs. signed-in
   IF shopper_id is None AND email is None:
     RETURN 400 Bad Request, {detail: "Email is required for guest checkout"}

3. # Get cart total and lines
   ASYNC WITH httpx.AsyncClient() AS client:
     cart_response = AWAIT client.get(f"http://localhost:8003/cart/{cart_id}")
   
4. IF cart_response.status_code == 404:
     RETURN 400 Bad Request, {detail: "Cart not found"}
5. IF cart_response.status_code != 200:
     RETURN 500 Internal Server Error, {detail: "Cart service unavailable"}

6. cart_data = cart_response.json()
   cart_total = cart_data["total_minor_units"]
   cart_lines = cart_data["lines"]

7. # Charge payment
   ASYNC WITH httpx.AsyncClient() AS client:
     payment_response = AWAIT client.post(
       "http://localhost:8005/charge",
       json={"amount_minor_units": cart_total}
     )
   
8. IF payment_response.status_code != 200:
     RETURN 500 Internal Server Error, {detail: "Payment service unavailable"}

9. payment_data = payment_response.json()
   
10. # Payment refused
    IF payment_data["status"] == "refused":
      RETURN CheckoutResponse(status="refused", reason=payment_data["reason"])

11. # Payment accepted → create order
    order_payload = {
      "shopper_id": shopper_id (if present),
      "email": email (if present),
      "ship_to": ship_to,
      "lines": cart_lines,
      "total_minor_units": cart_total,
      "payment_reference": payment_data["reference"]
    }
    
    ASYNC WITH httpx.AsyncClient() AS client:
      order_response = AWAIT client.post(
        "http://localhost:8006/orders",
        json=order_payload
      )
    
12. IF order_response.status_code != 201:
      # [KNOWN_GAP]: Payment succeeded but order not created
      RETURN 500 Internal Server Error, {detail: "Order service unavailable"}

13. order_data = order_response.json()
    order_id = order_data["order_id"]

14. # Empty cart
    ASYNC WITH httpx.AsyncClient() AS client:
      delete_response = AWAIT client.delete(f"http://localhost:8003/cart/{cart_id}")
    
15. IF delete_response.status_code != 204:
      # [KNOWN_GAP]: Order created but cart not emptied
      LOG ERROR "Failed to delete cart {cart_id} after checkout"

16. RETURN CheckoutResponse(order_id=order_id, status="paid")
```

**Notes:**
- **Service URLs hardcoded:** `http://localhost:8003` (cart), `http://localhost:8005` (payment), `http://localhost:8006` (order). Mark with `[SERVICE_URLS_HARDCODED]` for future config.
- **No retries:** If a service is down, checkout fails immediately. Mark with `[NO_RETRY]`.
- **Known gaps:**
  - Payment succeeds but order creation fails → client charged but no order. Mark with `[KNOWN_GAP]`.
  - Order created but cart delete fails → client has order but cart not emptied. Mark with `[KNOWN_GAP]`.
  - Future: add idempotency keys, retry logic, and compensating transactions (sagas or 2PC).
- **No idempotency:** Duplicate checkout requests will create multiple orders. Mark with `[IDEMPOTENCY_DEFERRED]`.
- **Async HTTP calls:** Use `httpx.AsyncClient` (FastAPI is async). Handle timeouts with default httpx timeout (5s). Mark with `[HTTP_TIMEOUT_DEFAULT]`.

---

### Database initialization (`db.py`)

```
1. Read MONGODB_URI from environment (default: "mongodb://localhost:27017")
2. Create MongoClient(uri)
3. db = client["checkout_db"]
4. Export db for future use (no collections in v1)
```

**Connection handling:**
- No connection pooling config in v1 (use MongoClient defaults).
- No retry logic; fail fast on connection errors.
- Database created on first write (if future idempotency logs are added).

---

### FastAPI app setup (`main.py`)

```
1. app = FastAPI(title="Checkout Service", version="1.0.0", docs_url="/docs")
2. Include routes from routes.py: app.include_router(router)
3. IF __name__ == "__main__":
     uvicorn.run(app, host="0.0.0.0", port=8004)
```

**Port allocation:**
- Checkout service runs on port `8004` (identity on 8001, catalog on 8002, cart on 8003, payment on 8005, order on 8006).
- Configurable via environment variable `PORT` (default: 8004).

---

## Tests

*(Matches breakdown Verification Approach count: 7 tests mapping to AC-5, AC-6, AC-7, AC-8, AC-11)*

### Test 1: Guest checkout success
**Setup:**
1. Start all services (identity on 8001, catalog on 8002, cart on 8003, checkout on 8004, payment on 8005, order on 8006).
2. Seed catalog with sellable product `prod1` (price 1000).
3. Create cart via `POST http://localhost:8003/cart`, capture `cart_id`.
4. Add `prod1` qty 2 to cart → cart total is 2000.

**Action:** `POST http://localhost:8004/checkout` with:
```json
{
  "cart_id": "<cart_id>",
  "email": "guest@example.com",
  "ship_to": {
    "street": "123 Main St",
    "city": "Springfield",
    "postal_code": "12345",
    "country": "USA"
  }
}
```

**Assert:**
- Response status 200
- Response body: `{order_id: "<order_id>", status: "paid"}`
- Cart is empty: `GET http://localhost:8003/cart/{cart_id}` returns 404
- Order exists: `GET http://localhost:8006/orders/{order_id}` returns order with `email: "guest@example.com"`, `ship_to`, `lines: [{product_id: "prod1", quantity: 2, price_minor_units: 1000}]`, `total_minor_units: 2000`, `payment_reference`, `created_at`

**Verification:** AC-5 (guest checkout asks payment for cart total), AC-8 (order created with lines, total, reference, address; cart emptied).

---

### Test 2: Signed-in checkout success
**Setup:**
1. Start all services.
2. Seed catalog with sellable product `prod1` (price 1000).
3. Seed identity with shopper `{email: "user@example.com", password: "password123"}`, capture `shopper_id`.
4. Create cart, add `prod1` qty 1 → cart total is 1000.

**Action:** `POST http://localhost:8004/checkout` with:
```json
{
  "cart_id": "<cart_id>",
  "shopper_id": "<shopper_id>",
  "ship_to": {
    "street": "456 Elm St",
    "city": "Shelbyville",
    "postal_code": "67890",
    "country": "USA"
  }
}
```

**Assert:**
- Response status 200
- Response body: `{order_id: "<order_id>", status: "paid"}`
- Cart is empty: `GET http://localhost:8003/cart/{cart_id}` returns 404
- Order includes `shopper_id`: `GET http://localhost:8006/orders/{order_id}` returns order with `shopper_id: "<shopper_id>"`, `ship_to`, `lines: [{product_id: "prod1", quantity: 1, price_minor_units: 1000}]`, `total_minor_units: 1000`, `payment_reference`, `created_at`

**Verification:** AC-6 (amount sent to payment equals cart total, order stored against shopper_id), AC-8 (order created, cart emptied).

---

### Test 3: Payment refused
**Setup:**
1. Start all services.
2. Seed catalog with sellable product `prod1` (price 1000).
3. Create cart, add `prod1` qty 200 → cart total is 200000 (triggers payment refusal: amount >= 100000).

**Action:** `POST http://localhost:8004/checkout` with:
```json
{
  "cart_id": "<cart_id>",
  "email": "guest@example.com",
  "ship_to": {
    "street": "123 Main St",
    "city": "Springfield",
    "postal_code": "12345",
    "country": "USA"
  }
}
```

**Assert:**
- Response status 200
- Response body: `{status: "refused", reason: "Amount exceeds limit"}`
- Cart unchanged: `GET http://localhost:8003/cart/{cart_id}` returns cart with 200 items and total 200000
- No order created: query `order_db.orders` → no documents match this checkout attempt

**Verification:** AC-7 (payment refused, no order created, cart unchanged, response says payment refused).

---

### Test 4: Cart unchanged on payment refusal
**Setup:** Same as Test 3.

**Action:** After checkout returns `{status: "refused"}`, call `GET http://localhost:8003/cart/{cart_id}`.

**Assert:**
- Cart still exists (status 200)
- Cart lines and total unchanged from before checkout

**Verification:** AC-7 (cart unchanged when payment refused).

---

### Test 5: Cart emptied on payment success
**Setup:** Same as Test 1 (guest checkout success).

**Action:** After checkout returns `{status: "paid"}`, call `GET http://localhost:8003/cart/{cart_id}`.

**Assert:**
- Response status 404 (cart deleted)

**Verification:** AC-8 (cart is empty after payment accepted).

---

### Test 6: Order created on payment success
**Setup:** Same as Test 1 (guest checkout success).

**Action:** After checkout returns `{order_id: "<order_id>"}`, call `GET http://localhost:8006/orders/{order_id}`.

**Assert:**
- Response status 200
- Order document contains all fields from checkout request: `email`, `ship_to`, `lines`, `total_minor_units`, `payment_reference`

**Verification:** AC-8 (order exists with lines, total, payment reference, ship-to address).

---

### Test 7: Swagger UI accessible
**Setup:** Start checkout service on port 8004.

**Action:** HTTP GET `http://localhost:8004/docs`.

**Assert:**
- Response status 200
- Response HTML contains "Checkout Service" title
- Swagger UI lists endpoint: `POST /checkout`
- Endpoint can be expanded and shows request schema (CheckoutRequest) and response schema (CheckoutResponse)

**Verification:** AC-11 (Swagger for every service).

---

## HLD Outcomes Binding This Design

From `hld.md`:

- **AD-1 (Data isolation):** Checkout service MUST use only `checkout_db`. No access to `cart_db`, `payment_db`, `order_db`, etc. Verified by inspecting MongoDB (no cross-database reads/writes). In v1, `checkout_db` is created but empty (no collections used).
- **AD-2 (HTTP-only inter-service communication):** Checkout calls cart, payment, and order services via HTTP. No direct database access to their data. Verified by algorithm (all cross-service calls use `httpx`).
- **AD-3 (Price authority):** Checkout uses the cart total without recalculating. Cart has already snapshotted catalog prices. Verified by algorithm step 6 (uses `cart_total` from cart response).
- **AD-6 (Guest vs. signed-in checkout):** Checkout accepts either `shopper_id` (signed-in) or `email` (guest). Validates that guest checkout includes `email`. Verified by algorithm steps 1-2 and Tests 1-2.
- **AD-7 (Cart lifecycle):** Checkout empties cart (calls `DELETE /cart/{cart_id}`) only after payment accepts and order is created. Cart unchanged on payment refusal. Verified by algorithm steps 10, 14-15 and Tests 3-5.
- **AD-8 (Service stack split):** Checkout is Python 3.12 + FastAPI (not Node.js). `requirements.txt` specifies dependencies including `httpx` for HTTP calls.
- **AD-9 (Swagger UI per service):** `/docs` endpoint enabled in `main.py`. Verified by Test 7.

**Database schema from HLD:**
- Checkout owns `checkout_db` but uses no collections in v1 (stateless orchestration).
- Future: may add `checkout_db.sessions` for idempotency keys.

**No deviations.** The orchestration flow and API contracts match HLD exactly.

---

## Known Gaps and Future Work

- **Service URLs hardcoded:** Cart (`http://localhost:8003`), payment (`http://localhost:8005`), order (`http://localhost:8006`) are hardcoded in `routes.py`. Mark with `[SERVICE_URLS_HARDCODED]`. Future: read from environment variables (`CART_SERVICE_URL`, `PAYMENT_SERVICE_URL`, `ORDER_SERVICE_URL`).
- **No timeout config:** httpx default timeout (5s) is used for all service calls. Mark with `[HTTP_TIMEOUT_DEFAULT]`. Future: make configurable per service.
- **No retry logic:** If cart, payment, or order service is temporarily down, checkout fails immediately. Mark with `[NO_RETRY]`. Future: add retries with exponential backoff and circuit breaker.
- **No idempotency:** Duplicate `POST /checkout` requests with the same `cart_id` will attempt to charge again and create multiple orders. Mark with `[IDEMPOTENCY_DEFERRED]`. Future: add idempotency key header, store in `checkout_db.sessions`.
- **Known gap: payment succeeds but order creation fails:** If order service returns 500 after payment accepts, the client is charged but no order exists. Checkout returns 500, but the charge is not reversed. Mark with `[KNOWN_GAP]`. Future: implement compensating transaction (refund) or saga pattern.
- **Known gap: order created but cart not emptied:** If cart delete fails after order is created, the client has a paid order but the cart still exists. Checkout logs the error but does not fail. Mark with `[KNOWN_GAP]`. Future: retry cart delete or accept eventual consistency (cart expires).
- **No authentication/authorization:** All endpoints are public. Any client can checkout any cart. Mark in Swagger docs as "public, no auth". Mark with `[AUTH_DEFERRED]`. Future: validate shopper_id against identity service, require auth token.
- **No cart ownership validation:** Checkout does not verify that the `cart_id` belongs to the `shopper_id` (if provided). Mark with `[CART_OWNERSHIP_DEFERRED]`. Future: cart service should store and enforce ownership.
- **No database transactions:** Checkout orchestration is not atomic. Partial failures leave inconsistent state (e.g., order created but cart not deleted). Mark with `[NO_TRANSACTIONS]`. Future: use sagas, 2PC, or Temporal workflows.
- **No audit log:** Checkout attempts are not logged. Mark with `[AUDIT_LOG_DEFERRED]`. Future: store checkout attempts in `checkout_db.audit_log` for debugging and analytics.
- **No order confirmation email:** Checkout does not send email to guest or shopper. Mark with `[EMAIL_DEFERRED]`. Future: call notification service after order creation.
- **No stock reservation:** Checkout does not reserve inventory before payment. Products may become unavailable between cart view and checkout. Mark with `[STOCK_RESERVATION_DEFERRED]`. Future: cart service reserves stock, checkout releases on refusal.
- **No fraud detection:** Payment is accepted/refused based on simple rules (amount, card number). No fraud checks. Mark with `[FRAUD_DETECTION_DEFERRED]`. Future: integrate fraud detection service before payment.
- **Logging:** No structured logging (print statements only in v1). Mark with `[LOGGING_DEFERRED]`.
- **Environment config:** `MONGODB_URI`, `PORT`, and service URLs read from environment; no `.env` file loading in v1 (manual setup required). Mark with `[CONFIG_DEFERRED]`.
- **Error detail exposure:** 500 errors may expose stack traces in debug mode. Mark with `[ERROR_DETAIL_DEFERRED]`. Future: sanitize errors in production.

---

**End of LLD.**
