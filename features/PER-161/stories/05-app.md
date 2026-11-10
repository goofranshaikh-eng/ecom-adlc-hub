# LLD — Story 05-app: Payment Service — Test double for charge acceptance/refusal

**Repo:** `app`

---

## Acceptance Criteria (from requirements and breakdown)

### AC-7: Payment refused

**Given** a cart at checkout

**When** payment refuses the charge

**Then** no order is created, the cart is unchanged, and the response says the payment was refused

*(Payment service returns `{status: "refused", reason: "..."}` when refusal conditions are met.)*

---

### AC-8: Payment accepted

**Given** a cart at checkout

**When** payment accepts the charge

**Then** an order exists with the lines, the total charged, a payment reference, and the ship-to address, and the cart is empty

*(Payment service returns `{status: "accepted", reference: "<UUID>"}` when acceptance conditions are met.)*

---

### AC-10: Services do not share databases

**Given** a completed purchase

**When** each service is inspected

**Then** catalog, cart, checkout, payment, order, and identity each read and write only their own data

*(Payment owns `payment_db`; no other service reads or writes it.)*

---

### AC-11: Swagger for every service

**Given** a running service

**When** a person opens that service's Swagger UI

**Then** every endpoint for that service is listed and can be called from the page

*(Payment exposes `/docs`.)*

---

## File Plan

All paths under repo root `/home/goos/projects/ecom`. The repo tree is currently empty.

**New files:**

- `app/payment/main.py` — FastAPI application entry point, enables Swagger UI at `/docs`
- `app/payment/requirements.txt` — Python dependencies: `fastapi`, `uvicorn`, `pymongo`, `pydantic`
- `app/payment/models.py` — Pydantic models for request/response validation
- `app/payment/routes.py` — API route handler for `/charge`
- `app/payment/db.py` — MongoDB connection to `payment_db`, reference to `charges` collection

**Directory structure:**

```
app/
└── payment/
    ├── main.py
    ├── requirements.txt
    ├── models.py
    ├── routes.py
    └── db.py
```

---

## Interfaces

### Pydantic Models (`models.py`)

#### `ChargeRequest`
**Fields:**
- `amount_minor_units: int` — charge amount in minor units (e.g., cents)
- `card: Optional[str]` — card number (optional; used to trigger refusal if ends in "0000")

**Validation:**
- `amount_minor_units` must be > 0

---

#### `ChargeResponse`
**Fields:**
- `status: str` — `"accepted"` or `"refused"`
- `reference: Optional[str]` — payment reference (UUID; present when `status == "accepted"`)
- `reason: Optional[str]` — refusal reason (present when `status == "refused"`)

**Validation:**
- `status` must be one of `["accepted", "refused"]`
- When `status == "accepted"`, `reference` must be present
- When `status == "refused"`, `reason` must be present

---

### API Routes (`routes.py`)

#### `POST /charge`
**Request body:** `ChargeRequest`

**Response:** `ChargeResponse` (200 OK) or `400 Bad Request`

**Logic:**

1. **Parse request body:** `ChargeRequest(amount_minor_units, card?)`

2. **Validate amount:**
   - If `amount_minor_units <= 0`, return `400 Bad Request` with `{detail: "Amount must be positive"}`. (Pydantic handles this.)

3. **Deterministic test double logic:**
   - **Refuse if:**
     - `amount_minor_units >= 100000` → `reason = "Amount exceeds limit"`
     - **OR** `card` is provided and ends with `"0000"` → `reason = "Card declined"`
   - **Accept otherwise.**

4. **If refused:**
   - Generate no reference.
   - Store charge record: `{_id: ObjectId(), amount_minor_units, status: "refused", reason, timestamp: datetime.utcnow()}`.
   - Return `ChargeResponse(status="refused", reason=reason)`.

5. **If accepted:**
   - Generate `reference` as UUID string: `str(uuid.uuid4())`.
   - Store charge record: `{_id: ObjectId(), amount_minor_units, status: "accepted", reference, timestamp: datetime.utcnow()}`.
   - Return `ChargeResponse(status="accepted", reference=reference)`.

**Error cases:**
- Invalid request body (non-positive amount) → 422 (Pydantic validation)
- Database insertion failure → 500 Internal Server Error

---

### MongoDB Schema (`db.py`)

**Database:** `payment_db`

**Collection:** `charges`

**Document structure:**
```json
{
  "_id": "ObjectId",
  "amount_minor_units": "integer (> 0)",
  "status": "accepted | refused",
  "reference": "string (UUID, only when accepted)",
  "reason": "string (only when refused)",
  "timestamp": "ISODate"
}
```

**Indexes:**
- Primary key on `_id` (default)
- Optional index on `reference` (for future lookups)
- Optional index on `timestamp` (for auditing)

**Connection:**
- MongoDB URI from environment variable `MONGODB_URI` (default: `mongodb://localhost:27017`)
- Database name: `payment_db`
- Collection reference: `db.charges`

---

## Algorithm Notes

### Charge flow (`POST /charge`)

```
1. Parse request body → ChargeRequest(amount_minor_units, card?)

2. # Validate amount (Pydantic handles this)
   # Proceed to test double logic

3. # Determine acceptance/refusal
   refused = False
   reason = None
   
4. IF amount_minor_units >= 100000:
     refused = True
     reason = "Amount exceeds limit"

5. ELSE IF card is not None AND card.endswith("0000"):
     refused = True
     reason = "Card declined"

6. IF refused:
     # Store refusal record
     doc = {
       "amount_minor_units": amount_minor_units,
       "status": "refused",
       "reason": reason,
       "timestamp": datetime.utcnow()
     }
     charges.insert_one(doc)
     RETURN ChargeResponse(status="refused", reason=reason)

7. ELSE:
     # Store acceptance record
     reference = str(uuid.uuid4())
     doc = {
       "amount_minor_units": amount_minor_units,
       "status": "accepted",
       "reference": reference,
       "timestamp": datetime.utcnow()
     }
     charges.insert_one(doc)
     RETURN ChargeResponse(status="accepted", reference=reference)
```

**Notes:**
- **Test double marker:** All charge logic is a stub. Mark with `[TEST_DOUBLE]` comment for future replacement with real payment provider integration.
- **Deterministic refusal conditions:**
  - Amount >= 100,000 minor units (e.g., $1,000.00 or more)
  - Card number ending in "0000"
- **No actual card validation:** Card number is treated as an opaque string. No Luhn algorithm, expiry check, or CVV validation.
- **UUID generation:** Use Python `uuid.uuid4()` for reference. Mark with `[UUID_REFERENCE]`.
- **Timestamp:** Use `datetime.utcnow()` for all charge records (UTC). Mark with `[UTC_TIMESTAMP]`.

---

### Database initialization (`db.py`)

```
1. Read MONGODB_URI from environment (default: "mongodb://localhost:27017")
2. Create MongoClient(uri)
3. db = client["payment_db"]
4. charges_collection = db["charges"]
5. (Optional) Create index on "reference": charges_collection.create_index("reference")
6. Export charges_collection for use in routes.py
```

**Connection handling:**
- No connection pooling config in v1 (use MongoClient defaults).
- No retry logic; fail fast on connection errors.

---

### FastAPI app setup (`main.py`)

```
1. app = FastAPI(title="Payment Service", version="1.0.0", docs_url="/docs")
2. Include routes from routes.py: app.include_router(router)
3. IF __name__ == "__main__":
     uvicorn.run(app, host="0.0.0.0", port=8005)
```

**Port allocation:**
- Payment service runs on port `8005` (identity on 8001, catalog on 8002, cart on 8003, checkout on 8004, order on 8006).
- Configurable via environment variable `PORT` (default: 8005).

---

## Tests

*(Matches breakdown Verification Approach count: 6 tests)*

### Test 1: Charge accepted (low amount)
**Setup:** Start payment service on port 8005.

**Action:** `POST http://localhost:8005/charge` with `{amount_minor_units: 5000}`.

**Assert:**
- Response status 200
- Response body: `{status: "accepted", reference: "<UUID>"}` (valid UUID format)
- MongoDB `payment_db.charges` contains one document:
  - `amount_minor_units == 5000`
  - `status == "accepted"`
  - `reference` matches response
  - `timestamp` is recent (within 1 second of test execution)

**Verification:** AC-8 (payment accepted, reference returned).

---

### Test 2: Charge refused (high amount)
**Setup:** Start payment service on port 8005.

**Action:** `POST http://localhost:8005/charge` with `{amount_minor_units: 150000}`.

**Assert:**
- Response status 200
- Response body: `{status: "refused", reason: "Amount exceeds limit"}`
- MongoDB `payment_db.charges` contains one document:
  - `amount_minor_units == 150000`
  - `status == "refused"`
  - `reason == "Amount exceeds limit"`
  - No `reference` field
  - `timestamp` is recent

**Verification:** AC-7 (payment refused, reason returned).

---

### Test 3: Charge refused (card ending in "0000")
**Setup:** Start payment service on port 8005.

**Action:** `POST http://localhost:8005/charge` with `{amount_minor_units: 5000, card: "1234567890000000"}`.

**Assert:**
- Response status 200
- Response body: `{status: "refused", reason: "Card declined"}`
- MongoDB `payment_db.charges` contains one document:
  - `amount_minor_units == 5000`
  - `status == "refused"`
  - `reason == "Card declined"`
  - No `reference` field
  - `timestamp` is recent

**Verification:** AC-7 (payment refused, alternate refusal condition).

---

### Test 4: Charge record stored in database
**Setup:** Start payment service on port 8005.

**Action:** 
1. `POST /charge` with `{amount_minor_units: 1000}` (accepted).
2. `POST /charge` with `{amount_minor_units: 200000}` (refused).

**Assert:**
- MongoDB `payment_db.charges` contains exactly 2 documents.
- First document: `status == "accepted"`, has `reference`.
- Second document: `status == "refused"`, has `reason`, no `reference`.

**Verification:** AC-10 (payment owns its database and stores charge records).

---

### Test 5: Swagger UI accessible
**Setup:** Start payment service on port 8005.

**Action:** HTTP GET `http://localhost:8005/docs`.

**Assert:**
- Response status 200
- Response HTML contains "Payment Service" title
- Swagger UI lists endpoint: `POST /charge`
- Endpoint can be expanded and shows:
  - Request schema: `ChargeRequest` with `amount_minor_units` (integer, required) and `card` (string, optional)
  - Response schema: `ChargeResponse` with `status`, `reference`, `reason`

**Verification:** AC-11 (Swagger for every service).

---

### Test 6: Database isolation
**Setup:** Start payment service. Execute several charge requests (accepted and refused).

**Action:** Query MongoDB to list all databases and collections accessed by payment service.

**Assert:**
- Only `payment_db.charges` exists and contains data
- No other databases (`identity_db`, `catalog_db`, `cart_db`, `checkout_db`, `order_db`) are created or accessed by this service
- Payment service does not call any other service's HTTP endpoints (verified by network logs or lack of httpx/requests usage in code)

**Verification:** AC-10 (services do not share databases).

---

## HLD Outcomes Binding This Design

From `hld.md`:

- **AD-1 (Data isolation):** Payment service MUST use only `payment_db`. No access to `cart_db`, `order_db`, `checkout_db`, etc. Verified by Test 6 (database isolation check).
- **AD-2 (HTTP-only inter-service communication):** Payment does not call other services. Checkout calls payment via HTTP `POST /charge`. Verified by algorithm (no cross-service calls in payment code).
- **AD-4 (Payment test double):** Payment service is a deterministic test double with hardcoded refusal logic (amount >= 100000, card ending in "0000"). Marked with `[TEST_DOUBLE]` comment. Verified by Tests 1-3.
- **AD-5 (Money representation):** All amounts stored and returned as `amount_minor_units` (integer). No floating-point arithmetic. Verified by all tests (amounts are integers).
- **AD-8 (Service stack split):** Payment is Python 3.12 + FastAPI (not Node.js). `requirements.txt` specifies dependencies.
- **AD-9 (Swagger UI per service):** `/docs` endpoint enabled in `main.py`. Verified by Test 5.

**Database schema from HLD:**
```json
{
  "_id": "ObjectId",
  "amount_minor_units": "integer",
  "status": "accepted | refused",
  "reference": "string (UUID or stub)",
  "reason": "string (only when refused)",
  "timestamp": "ISODate"
}
```

**No deviations.** The schema and API contracts match HLD exactly.

---

## Known Gaps and Future Work

- **Test double:** All payment logic is a stub. No real payment provider integration (Stripe, PayPal, etc.). Mark all charge logic with `[TEST_DOUBLE]`. Future: replace with real provider SDK, environment-specific API keys, webhook handling.
- **Card validation:** Card number is treated as an opaque string. No Luhn algorithm, expiry date, CVV, or billing address validation. Mark with `[CARD_VALIDATION_DEFERRED]`. Future: validate card format before calling provider.
- **Refusal reasons:** Only two hardcoded reasons in v1 ("Amount exceeds limit", "Card declined"). Real providers return detailed error codes (insufficient funds, fraud detection, expired card, etc.). Mark with `[REFUSAL_REASONS_STUB]`. Future: map provider error codes to user-friendly reasons.
- **No idempotency:** Duplicate `POST /charge` requests create multiple charge records. Mark with `[IDEMPOTENCY_DEFERRED]`. Future: add idempotency key header, store in `charges` with unique index.
- **No charge lookup:** No `GET /charges/{charge_id}` endpoint to retrieve charge history. Mark with `[CHARGE_LOOKUP_DEFERRED]`. Future: add endpoint for auditing and customer support.
- **No refunds:** No `POST /refund` endpoint. Mark with `[REFUNDS_DEFERRED]`. Future: add refund logic with partial/full refund support.
- **No webhook handling:** Real payment providers send webhooks for async charge updates (chargeback, dispute, delayed capture). Mark with `[WEBHOOKS_DEFERRED]`. Future: add webhook receiver endpoint.
- **No fraud detection:** Accepts all charges below threshold. No IP check, velocity check, or fraud scoring. Mark with `[FRAUD_DETECTION_DEFERRED]`. Future: integrate fraud detection service (Stripe Radar, Sift, etc.).
- **No currency:** Implicit (single merchant, single currency). No `currency` field in charge record. Mark with `[CURRENCY_DEFERRED]`. Future: add `currency` field (ISO 4217 code) for multi-currency support.
- **No merchant account:** Implicit single merchant. No `merchant_id` field. Mark with `[MERCHANT_DEFERRED]`. Future: add `merchant_id` for multi-tenant support.
- **No charge metadata:** No `metadata` field for storing checkout context (cart_id, order_id, shopper_id). Mark with `[CHARGE_METADATA_DEFERRED]`. Future: add `metadata: Dict[str, Any]` for traceability.
- **No authentication/authorization:** All endpoints are public. Any client can create charges. Mark in Swagger docs as "public, no auth". Mark with `[AUTH_DEFERRED]`. Future: require API key or service-to-service token.
- **No rate limiting:** No protection against charge flooding. Mark with `[RATE_LIMITING_DEFERRED]`. Future: add rate limiting middleware (per IP, per merchant, per card).
- **No logging:** No structured logging (print statements only in v1). Mark with `[LOGGING_DEFERRED]`. Future: log all charges with request ID, amount, status, execution time.
- **Environment config:** `MONGODB_URI` and `PORT` read from environment; no `.env` file loading in v1 (manual setup required). Mark with `[CONFIG_DEFERRED]`.
- **Error detail exposure:** 500 errors may expose stack traces in debug mode. Mark with `[ERROR_DETAIL_DEFERRED]`. Future: sanitize errors in production.
- **No charge expiry:** Charge records stored indefinitely. Mark with `[CHARGE_EXPIRY_DEFERRED]`. Future: add TTL index or archival process.

---

**End of LLD.**
