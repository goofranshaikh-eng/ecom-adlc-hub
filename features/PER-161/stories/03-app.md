# LLD — Story 03-app: Cart Service — Add items, update quantity, compute totals

**Repo:** `app`

---

## Acceptance Criteria (from requirements and breakdown)

### AC-2: Unsellable product cannot be added

**Given** a product that cannot be sold

**When** the client fetches it

**Then** the price is still returned and add-to-cart is refused with a reason

*(Cart calls catalog to verify sellability; if `sellable: false`, the add operation is refused.)*

---

### AC-3: Cart total is the catalog price

**Given** a sellable product

**When** the client adds a quantity

**Then** the cart line shows that quantity and the total equals catalog price times quantity

---

### AC-4: Quantity change recomputes the total

**Given** a cart with a line

**When** the client changes the quantity, including to zero

**Then** the line updates or disappears and the total matches the remaining lines

---

### AC-10: Services do not share databases

**Given** a completed purchase

**When** each service is inspected

**Then** catalog, cart, checkout, payment, order, and identity each read and write only their own data

*(Cart owns `cart_db`; no other service reads or writes it.)*

---

### AC-11: Swagger for every service

**Given** a running service

**When** a person opens that service's Swagger UI

**Then** every endpoint for that service is listed and can be called from the page

*(Cart exposes `/docs`.)*

---

## File Plan

All paths under repo root `/home/goos/projects/ecom`. The repo tree is currently empty.

**New files:**

- `app/cart/main.py` — FastAPI application entry point, enables Swagger UI at `/docs`
- `app/cart/requirements.txt` — Python dependencies: `fastapi`, `uvicorn`, `pymongo`, `pydantic`, `httpx`
- `app/cart/models.py` — Pydantic models for request/response validation
- `app/cart/routes.py` — API route handlers for `/cart`, `/cart/{cart_id}`, `/cart/{cart_id}/items`, `/cart/{cart_id}/items/{product_id}`
- `app/cart/db.py` — MongoDB connection to `cart_db`, reference to `carts` collection

**Directory structure:**

```
app/
└── cart/
    ├── main.py
    ├── requirements.txt
    ├── models.py
    ├── routes.py
    └── db.py
```

---

## Interfaces

### Pydantic Models (`models.py`)

#### `CartLine`
**Fields:**
- `product_id: str` — product identifier
- `quantity: int` — number of items
- `price_snapshot_minor_units: int` — price per unit at add-time (from catalog)

**Validation:**
- `quantity` must be > 0 (zero quantity is handled separately in update logic)
- `price_snapshot_minor_units` must be >= 0

---

#### `CartResponse`
**Fields:**
- `cart_id: str` — cart identifier
- `lines: List[CartLine]` — list of cart lines
- `total_minor_units: int` — sum of `price_snapshot_minor_units * quantity` for all lines

**Validation:**
- `total_minor_units` must be >= 0

---

#### `AddItemRequest`
**Fields:**
- `product_id: str` — product to add
- `quantity: int` — number of items to add

**Validation:**
- `product_id` must be non-empty
- `quantity` must be > 0

---

#### `UpdateItemRequest`
**Fields:**
- `quantity: int` — new quantity (can be 0 to remove)

**Validation:**
- `quantity` must be >= 0

---

#### `ErrorResponse`
**Fields:**
- `error: str` — error message (e.g., "Product cannot be added")
- `reason: str` — detailed reason (e.g., "Product is not sellable")

---

### API Routes (`routes.py`)

#### `POST /cart`
**Request body:** None

**Response:** `{cart_id: str}` (201 Created)

**Logic:**
1. Generate a new `cart_id` (UUID or MongoDB ObjectId as string).
2. Insert document into `carts` collection: `{_id: cart_id, lines: [], total_minor_units: 0}`.
3. Return `{cart_id}`.

**Error cases:**
- Database insertion failure → 500 Internal Server Error

---

#### `GET /cart/{cart_id}`
**Path parameter:** `cart_id` (string)

**Response:** `CartResponse` (200 OK) or `404 Not Found`

**Logic:**
1. Query `carts` collection for `_id == cart_id`.
2. If not found, return `404 Not Found` with `{detail: "Cart not found"}`.
3. If found, convert to `CartResponse`: `cart_id=doc["_id"]`, `lines=[CartLine(**line) for line in doc["lines"]]`, `total_minor_units=doc["total_minor_units"]`.
4. Return `CartResponse`.

**Error cases:**
- Cart not found → 404

---

#### `POST /cart/{cart_id}/items`
**Path parameter:** `cart_id` (string)

**Request body:** `AddItemRequest`

**Response:** `CartResponse` (200 OK), `ErrorResponse` (400 Bad Request), or `404 Not Found`

**Logic:**
1. Query `carts` collection for `_id == cart_id`. If not found, return 404.
2. **Call catalog service:** `GET http://localhost:8002/products/{product_id}` (from `AddItemRequest`).
   - Use `httpx.AsyncClient` to make the HTTP call.
   - If catalog returns 404, return `400 Bad Request` with `{error: "Product not found", reason: "Product does not exist in catalog"}`.
   - If catalog returns non-200, return `500 Internal Server Error` with `{detail: "Catalog service unavailable"}`.
3. Parse catalog response to extract `sellable` and `price_minor_units`.
4. **If `sellable == false`:**
   - Return `400 Bad Request` with `ErrorResponse(error="Product cannot be added", reason="Product is not sellable")`.
   - Cart remains unchanged.
5. **If `sellable == true`:**
   - Check if a line with `product_id` already exists in `cart["lines"]`.
   - **If exists:** increment `quantity` by `AddItemRequest.quantity` (do NOT update `price_snapshot_minor_units`; it was set at first add).
   - **If not exists:** append new line: `{product_id, quantity: AddItemRequest.quantity, price_snapshot_minor_units: catalog.price_minor_units}`.
   - Recompute `total_minor_units` = sum of `line.price_snapshot_minor_units * line.quantity` for all lines.
   - Update `carts` collection with new `lines` and `total_minor_units`.
   - Return `CartResponse` with updated state.

**Error cases:**
- Cart not found → 404
- Product not found in catalog → 400 with reason
- Product not sellable → 400 with `ErrorResponse`
- Catalog service down → 500

---

#### `PATCH /cart/{cart_id}/items/{product_id}`
**Path parameters:** `cart_id` (string), `product_id` (string)

**Request body:** `UpdateItemRequest`

**Response:** `CartResponse` (200 OK) or `404 Not Found`

**Logic:**
1. Query `carts` collection for `_id == cart_id`. If not found, return 404.
2. Find line in `cart["lines"]` where `line["product_id"] == product_id`.
   - If not found, return `404 Not Found` with `{detail: "Product not in cart"}`.
3. **If `UpdateItemRequest.quantity == 0`:**
   - Remove the line from `cart["lines"]`.
4. **If `UpdateItemRequest.quantity > 0`:**
   - Update `line["quantity"]` to `UpdateItemRequest.quantity`.
   - `price_snapshot_minor_units` remains unchanged.
5. Recompute `total_minor_units` = sum of `line.price_snapshot_minor_units * line.quantity` for remaining lines.
6. Update `carts` collection with new `lines` and `total_minor_units`.
7. Return `CartResponse` with updated state.

**Error cases:**
- Cart not found → 404
- Product not in cart → 404
- Invalid quantity (negative) → 422 (Pydantic validation)

---

#### `DELETE /cart/{cart_id}`
**Path parameter:** `cart_id` (string)

**Response:** `204 No Content` or `404 Not Found`

**Logic:**
1. Delete document from `carts` collection where `_id == cart_id`.
2. If `deleted_count == 0`, return `404 Not Found` with `{detail: "Cart not found"}`.
3. Return `204 No Content`.

**Error cases:**
- Cart not found → 404

---

### MongoDB Schema (`db.py`)

**Database:** `cart_db`

**Collection:** `carts`

**Document structure:**
```json
{
  "_id": "string (cart_id, UUID or ObjectId)",
  "lines": [
    {
      "product_id": "string",
      "quantity": "integer (> 0)",
      "price_snapshot_minor_units": "integer (>= 0)"
    }
  ],
  "total_minor_units": "integer (>= 0)"
}
```

**Indexes:**
- Primary key on `_id` (default)
- No additional indexes in v1

**Connection:**
- MongoDB URI from environment variable `MONGODB_URI` (default: `mongodb://localhost:27017`)
- Database name: `cart_db`
- Collection reference: `db.carts`

---

## Algorithm Notes

### Create cart flow (`POST /cart`)

```
1. cart_id = str(uuid.uuid4())  # or str(ObjectId())
2. doc = {
     "_id": cart_id,
     "lines": [],
     "total_minor_units": 0
   }
3. carts.insert_one(doc)
4. RETURN {cart_id}
```

**Notes:**
- Use `uuid.uuid4()` for cart ID (simpler than ObjectId for a session-like entity).
- No error handling for duplicate `_id` (UUID collision is negligible).

---

### Get cart flow (`GET /cart/{cart_id}`)

```
1. doc = carts.find_one({"_id": cart_id})
2. IF doc is None:
     RETURN 404 Not Found
3. lines = [CartLine(**line) for line in doc["lines"]]
4. RETURN CartResponse(
     cart_id=doc["_id"],
     lines=lines,
     total_minor_units=doc["total_minor_units"]
   )
```

---

### Add item flow (`POST /cart/{cart_id}/items`)

```
1. doc = carts.find_one({"_id": cart_id})
2. IF doc is None:
     RETURN 404 Not Found

3. # Call catalog service
   ASYNC WITH httpx.AsyncClient() AS client:
     response = AWAIT client.get(f"http://localhost:8002/products/{product_id}")
   
4. IF response.status_code == 404:
     RETURN 400 Bad Request, ErrorResponse(
       error="Product not found",
       reason="Product does not exist in catalog"
     )
5. IF response.status_code != 200:
     RETURN 500 Internal Server Error

6. catalog_product = response.json()
7. IF catalog_product["sellable"] == False:
     RETURN 400 Bad Request, ErrorResponse(
       error="Product cannot be added",
       reason="Product is not sellable"
     )

8. # Update cart
   lines = doc["lines"]
   existing_line = FIND line IN lines WHERE line["product_id"] == product_id
   
9. IF existing_line:
     existing_line["quantity"] += AddItemRequest.quantity
   ELSE:
     lines.append({
       "product_id": product_id,
       "quantity": AddItemRequest.quantity,
       "price_snapshot_minor_units": catalog_product["price_minor_units"]
     })

10. total_minor_units = SUM(line["price_snapshot_minor_units"] * line["quantity"] FOR line IN lines)

11. carts.update_one(
      {"_id": cart_id},
      {"$set": {"lines": lines, "total_minor_units": total_minor_units}}
    )

12. RETURN CartResponse(cart_id, lines, total_minor_units)
```

**Notes:**
- **Catalog service URL:** Hardcoded to `http://localhost:8002` in v1. Mark with `[CATALOG_URL_HARDCODED]` for future config.
- **Price snapshot:** Set only when product is first added. Subsequent adds increment quantity but do NOT update price.
- **Sellability check:** Happens on EVERY add. If a product becomes unsellable after first add, subsequent adds will fail (cart keeps existing line).
- **Async HTTP call:** Use `httpx.AsyncClient` (FastAPI is async). Handle timeouts with default httpx timeout (5s).

---

### Update item quantity flow (`PATCH /cart/{cart_id}/items/{product_id}`)

```
1. doc = carts.find_one({"_id": cart_id})
2. IF doc is None:
     RETURN 404 Not Found

3. lines = doc["lines"]
4. existing_line = FIND line IN lines WHERE line["product_id"] == product_id
5. IF existing_line is None:
     RETURN 404 Not Found, {detail: "Product not in cart"}

6. IF UpdateItemRequest.quantity == 0:
     lines.remove(existing_line)
   ELSE:
     existing_line["quantity"] = UpdateItemRequest.quantity

7. total_minor_units = SUM(line["price_snapshot_minor_units"] * line["quantity"] FOR line IN lines)

8. carts.update_one(
     {"_id": cart_id},
     {"$set": {"lines": lines, "total_minor_units": total_minor_units}}
   )

9. RETURN CartResponse(cart_id, lines, total_minor_units)
```

**Notes:**
- **Quantity zero:** Removes the line entirely (not just set to zero).
- **No catalog call:** Update does NOT re-verify sellability or re-fetch price. Uses existing `price_snapshot_minor_units`.

---

### Delete cart flow (`DELETE /cart/{cart_id}`)

```
1. result = carts.delete_one({"_id": cart_id})
2. IF result.deleted_count == 0:
     RETURN 404 Not Found
3. RETURN 204 No Content
```

---

### Database initialization (`db.py`)

```
1. Read MONGODB_URI from environment (default: "mongodb://localhost:27017")
2. Create MongoClient(uri)
3. db = client["cart_db"]
4. carts_collection = db["carts"]
5. Export carts_collection for use in routes.py
```

**Connection handling:**
- No connection pooling config in v1 (use MongoClient defaults).
- No retry logic; fail fast on connection errors.

---

### FastAPI app setup (`main.py`)

```
1. app = FastAPI(title="Cart Service", version="1.0.0", docs_url="/docs")
2. Include routes from routes.py: app.include_router(router)
3. IF __name__ == "__main__":
     uvicorn.run(app, host="0.0.0.0", port=8003)
```

**Port allocation:**
- Cart service runs on port `8003` (identity on 8001, catalog on 8002, checkout on 8004, etc.).
- Configurable via environment variable `PORT` (default: 8003).

---

## Tests

*(Matches breakdown Verification Approach count: 9 tests)*

### Test 1: Create cart
**Setup:** Start cart service on port 8003.

**Action:** `POST http://localhost:8003/cart`.

**Assert:**
- Response status 201
- Response body contains `{cart_id: "<UUID>"}` (valid UUID format)

**Verification:** Cart service endpoint operational (prerequisite for all AC).

---

### Test 2: Add sellable item to cart
**Setup:**
1. Start catalog service on port 8002.
2. Seed `catalog_db.products` with `{_id: "prod1", name: "Widget", price_minor_units: 1000, sellable: true}`.
3. Create a cart via `POST /cart`, capture `cart_id`.

**Action:** `POST http://localhost:8003/cart/{cart_id}/items` with `{product_id: "prod1", quantity: 2}`.

**Assert:**
- Response status 200
- Response body `CartResponse`:
  - `cart_id` matches
  - `lines` contains one line: `{product_id: "prod1", quantity: 2, price_snapshot_minor_units: 1000}`
  - `total_minor_units == 2000` (1000 * 2)

**Verification:** AC-3 (cart total is catalog price times quantity).

---

### Test 3: Add unsellable item refused
**Setup:**
1. Start catalog service on port 8002.
2. Seed `catalog_db.products` with `{_id: "prod2", name: "Gadget", price_minor_units: 2000, sellable: false}`.
3. Create a cart via `POST /cart`, capture `cart_id`.

**Action:** `POST http://localhost:8003/cart/{cart_id}/items` with `{product_id: "prod2", quantity: 1}`.

**Assert:**
- Response status 400
- Response body `ErrorResponse`:
  - `error == "Product cannot be added"`
  - `reason == "Product is not sellable"`
- Cart unchanged: `GET /cart/{cart_id}` returns `{lines: [], total_minor_units: 0}`

**Verification:** AC-2 (unsellable product add refused with reason).

---

### Test 4: Update item quantity
**Setup:**
1. Start catalog and cart services.
2. Seed catalog with sellable product `prod1` (price 1000).
3. Create cart, add `prod1` with quantity 2 (total 2000).

**Action:** `PATCH http://localhost:8003/cart/{cart_id}/items/prod1` with `{quantity: 3}`.

**Assert:**
- Response status 200
- Response body `CartResponse`:
  - `lines` contains `{product_id: "prod1", quantity: 3, price_snapshot_minor_units: 1000}`
  - `total_minor_units == 3000` (1000 * 3)

**Verification:** AC-4 (quantity change recomputes total).

---

### Test 5: Remove line with zero quantity
**Setup:**
1. Start catalog and cart services.
2. Seed catalog with sellable product `prod1` (price 1000).
3. Create cart, add `prod1` with quantity 2 (total 2000).

**Action:** `PATCH http://localhost:8003/cart/{cart_id}/items/prod1` with `{quantity: 0}`.

**Assert:**
- Response status 200
- Response body `CartResponse`:
  - `lines == []` (line removed)
  - `total_minor_units == 0`

**Verification:** AC-4 (quantity change including zero: line disappears, total matches remaining lines).

---

### Test 6: Get cart returns current state
**Setup:**
1. Start catalog and cart services.
2. Seed catalog with two sellable products: `prod1` (price 1000), `prod3` (price 500).
3. Create cart, add `prod1` qty 2, add `prod3` qty 1.

**Action:** `GET http://localhost:8003/cart/{cart_id}`.

**Assert:**
- Response status 200
- Response body `CartResponse`:
  - `lines` contains two lines:
    - `{product_id: "prod1", quantity: 2, price_snapshot_minor_units: 1000}`
    - `{product_id: "prod3", quantity: 1, price_snapshot_minor_units: 500}`
  - `total_minor_units == 2500` (1000*2 + 500*1)

**Verification:** AC-3 and AC-4 (cart total matches sum of lines).

---

### Test 7: Delete cart
**Setup:** Create a cart via `POST /cart`, capture `cart_id`.

**Action:** `DELETE http://localhost:8003/cart/{cart_id}`.

**Assert:**
- Response status 204 (No Content)
- Subsequent `GET /cart/{cart_id}` returns 404

**Verification:** Cart deletion (required for AC-8: checkout empties cart).

---

### Test 8: Swagger UI accessible
**Setup:** Start cart service on port 8003.

**Action:** HTTP GET `http://localhost:8003/docs`.

**Assert:**
- Response status 200
- Response HTML contains "Cart Service" title
- Swagger UI lists endpoints:
  - `POST /cart`
  - `GET /cart/{cart_id}`
  - `POST /cart/{cart_id}/items`
  - `PATCH /cart/{cart_id}/items/{product_id}`
  - `DELETE /cart/{cart_id}`
- Each endpoint can be expanded and shows request/response schemas

**Verification:** AC-11 (Swagger for every service).

---

### Test 9: Database isolation
**Setup:**
1. Start cart service.
2. Create a cart and add items (catalog service must be running).

**Action:** Query MongoDB to list all databases and collections accessed by cart service.

**Assert:**
- Only `cart_db.carts` exists and contains data
- No other databases (`identity_db`, `catalog_db`, `checkout_db`, etc.) are created or accessed by this service
- Cart service calls catalog service via HTTP (verified by network logs or httpx mock), but does NOT read `catalog_db` directly

**Verification:** AC-10 (services do not share databases).

---

## HLD Outcomes Binding This Design

From `hld.md`:

- **AD-1 (Data isolation):** Cart service MUST use only `cart_db`. No access to `catalog_db`, `identity_db`, etc. Verified by Test 9 (database isolation check).
- **AD-2 (HTTP-only inter-service communication):** Cart calls catalog via HTTP `GET /products/{product_id}` to verify sellability and fetch price. No direct database access to catalog. Verified by Test 2 (catalog HTTP call) and Test 9 (no `catalog_db` access).
- **AD-3 (Price authority):** Catalog is the source of truth for product price. Cart snapshots `price_minor_units` at add-time and stores it in `price_snapshot_minor_units`. Cart does NOT recalculate price on quantity updates. Verified by Test 2 (price snapshot), Test 4 (quantity update does not change price).
- **AD-5 (Money representation):** All prices stored and returned as `price_snapshot_minor_units` and `total_minor_units` (integers). No floating-point arithmetic. Verified by all tests (amounts are integers).
- **AD-8 (Service stack split):** Cart is Python 3.12 + FastAPI (not Node.js). `requirements.txt` specifies dependencies including `httpx` for HTTP calls.
- **AD-9 (Swagger UI per service):** `/docs` endpoint enabled in `main.py`. Verified by Test 8.

**Database schema from HLD:**
```json
{
  "_id": "string (cart_id)",
  "lines": [
    {
      "product_id": "string",
      "quantity": "integer",
      "price_snapshot_minor_units": "integer"
    }
  ],
  "total_minor_units": "integer"
}
```

**No deviations.** The schema and API contracts match HLD exactly.

---

## Known Gaps and Future Work

- **Catalog service URL hardcoded:** `http://localhost:8002` is hardcoded in `routes.py`. Mark with `[CATALOG_URL_HARDCODED]`. Future: read from environment variable `CATALOG_SERVICE_URL`.
- **No timeout config:** httpx default timeout (5s) is used. Mark with `[HTTP_TIMEOUT_DEFAULT]`. Future: make configurable.
- **No retry logic:** If catalog service is down, cart returns 500. No retry or circuit breaker. Mark with `[NO_RETRY]`. Future: add retries with exponential backoff.
- **No authentication/authorization:** All endpoints are public. Any client can access any cart. Mark in Swagger docs as "public, no auth". Mark with `[AUTH_DEFERRED]`.
- **No cart expiry:** Carts never expire. Mark with `[CART_EXPIRY_DEFERRED]`. Future: add `created_at` timestamp and TTL index.
- **No cart ownership:** No `shopper_id` or session token linking cart to a user. Mark with `[CART_OWNERSHIP_DEFERRED]`. Future: add `shopper_id` field and enforce ownership.
- **Concurrent updates:** Last-write-wins. No optimistic locking or versioning. Mark with `[CONCURRENT_UPDATES_DEFERRED]`. Future: add `version` field and compare-and-swap.
- **Price drift detection:** Cart does not detect or warn if catalog price changes after add. Mark with `[PRICE_DRIFT_DEFERRED]`. Future: on checkout, compare cart `price_snapshot_minor_units` to current catalog price and warn user.
- **No idempotency:** Duplicate `POST /cart/{cart_id}/items` requests add the product multiple times (increment quantity each time). Mark with `[IDEMPOTENCY_DEFERRED]`. Future: add idempotency key header and dedup.
- **No cart item metadata:** No product name or image stored in cart (only `product_id`). Client must call catalog to fetch display details. Mark with `[CART_DISPLAY_DEFERRED]`. Acceptable for v1 (cart is data only, no UI).
- **No maximum quantity:** Cart allows unlimited quantity per line. Mark with `[MAX_QUANTITY_DEFERRED]`. Future: enforce max quantity per product or per cart total.
- **Logging:** No structured logging (print statements only in v1). Mark with `[LOGGING_DEFERRED]`.
- **Environment config:** `MONGODB_URI`, `PORT`, and `CATALOG_SERVICE_URL` read from environment; no `.env` file loading in v1 (manual setup required). Mark with `[CONFIG_DEFERRED]`.
- **Error detail exposure:** 500 errors may expose stack traces in debug mode. Mark with `[ERROR_DETAIL_DEFERRED]`. Future: sanitize errors in production.

---

**End of LLD.**
