# LLD — Story 02-app: Catalog Service — Products, prices, and sellability

**Repo:** `app`

---

## Acceptance Criteria (from requirements and breakdown)

### AC-1: Find a product

**Given** the catalog has a sellable product

**When** the client searches for it and fetches it

**Then** the response includes its name, price, and that it can be added to the cart

---

### AC-2: Unsellable product cannot be added

**Given** a product that cannot be sold

**When** the client fetches it

**Then** the price is still returned and add-to-cart is refused with a reason

*(Catalog returns the sellable flag; cart service will enforce the refusal.)*

---

### AC-10: Services do not share databases

**Given** a completed purchase

**When** each service is inspected

**Then** catalog, cart, checkout, payment, order, and identity each read and write only their own data

*(Catalog owns `catalog_db`; no other service reads or writes it.)*

---

### AC-11: Swagger for every service

**Given** a running service

**When** a person opens that service's Swagger UI

**Then** every endpoint for that service is listed and can be called from the page

*(Catalog exposes `/docs`.)*

---

## File Plan

All paths under repo root `/home/goos/projects/ecom`. The repo tree is currently empty.

**New files:**

- `app/catalog/main.py` — FastAPI application entry point, enables Swagger UI at `/docs`
- `app/catalog/requirements.txt` — Python dependencies: `fastapi`, `uvicorn`, `pymongo`, `pydantic`
- `app/catalog/models.py` — Pydantic models for request/response validation
- `app/catalog/routes.py` — API route handlers for `/products/search` and `/products/{product_id}`
- `app/catalog/db.py` — MongoDB connection to `catalog_db`, reference to `products` collection

**Directory structure:**

```
app/
└── catalog/
    ├── main.py
    ├── requirements.txt
    ├── models.py
    ├── routes.py
    └── db.py
```

---

## Interfaces

### Pydantic Models (`models.py`)

#### `Product`
**Fields:**
- `id: str` — MongoDB `ObjectId` as string
- `name: str` — product name
- `price_minor_units: int` — price in minor units (e.g., cents)
- `sellable: bool` — whether the product can be added to a cart

**Validation:**
- `name` must be non-empty
- `price_minor_units` must be >= 0
- `sellable` is a boolean

---

#### `ProductSearchResponse`
**Fields:**
- `products: List[Product]` — list of matching products

---

### API Routes (`routes.py`)

#### `GET /products/search?q={query}`
**Query parameter:** `q` (string, required) — search term

**Response:** `ProductSearchResponse` (200 OK) or `400 Bad Request`

**Logic:**
1. If `q` is empty or missing, return `400 Bad Request` with `{detail: "Query parameter 'q' is required"}`.
2. Query `products` collection with case-insensitive regex match on `name` field: `{"name": {"$regex": q, "$options": "i"}}`.
3. Convert each document to `Product` model: `id=str(doc["_id"])`, `name=doc["name"]`, `price_minor_units=doc["price_minor_units"]`, `sellable=doc["sellable"]`.
4. Return `ProductSearchResponse(products=[...])`.

**Error cases:**
- Missing or empty `q` → 400
- No matching products → return empty list (200 OK with `{products: []}`)

---

#### `GET /products/{product_id}`
**Path parameter:** `product_id` (string)

**Response:** `Product` (200 OK) or `404 Not Found`

**Logic:**
1. Convert `product_id` string to MongoDB `ObjectId`.
2. Query `products` collection for `_id == ObjectId(product_id)`.
3. If not found, return `404 Not Found` with `{detail: "Product not found"}`.
4. If found, return `Product(id=str(doc["_id"]), name=doc["name"], price_minor_units=doc["price_minor_units"], sellable=doc["sellable"])`.

**Error cases:**
- Invalid `ObjectId` format → 400 Bad Request
- Product not found → 404

---

### MongoDB Schema (`db.py`)

**Database:** `catalog_db`

**Collection:** `products`

**Document structure:**
```json
{
  "_id": "ObjectId",
  "name": "string",
  "price_minor_units": "integer",
  "sellable": "boolean"
}
```

**Indexes:**
- Text index on `name` (optional; supports case-insensitive search via regex in v1)

**Connection:**
- MongoDB URI from environment variable `MONGODB_URI` (default: `mongodb://localhost:27017`)
- Database name: `catalog_db`
- Collection reference: `db.products`

---

## Algorithm Notes

### Product search flow (`GET /products/search`)

```
1. Parse query parameter q
2. IF q is empty or None:
     RETURN 400 Bad Request
3. query = {"name": {"$regex": q, "$options": "i"}}
4. cursor = products.find(query)
5. result = []
6. FOR doc IN cursor:
     product = Product(
       id=str(doc["_id"]),
       name=doc["name"],
       price_minor_units=doc["price_minor_units"],
       sellable=doc["sellable"]
     )
     result.append(product)
7. RETURN ProductSearchResponse(products=result)
```

**Notes:**
- Case-insensitive search uses MongoDB `$regex` with `$options: "i"`.
- No pagination in v1 (return all matches).
- Empty result set is valid (return `{products: []}`).

---

### Product fetch flow (`GET /products/{product_id}`)

```
1. TRY:
     oid = ObjectId(product_id)
   EXCEPT InvalidId:
     RETURN 400 Bad Request
2. doc = products.find_one({"_id": oid})
3. IF doc is None:
     RETURN 404 Not Found
4. product = Product(
     id=str(doc["_id"]),
     name=doc["name"],
     price_minor_units=doc["price_minor_units"],
     sellable=doc["sellable"]
   )
5. RETURN product
```

**Error handling:**
- Invalid `ObjectId` format → catch `bson.errors.InvalidId`, return 400.
- Missing document → return 404.

---

### Database initialization (`db.py`)

```
1. Read MONGODB_URI from environment (default: "mongodb://localhost:27017")
2. Create MongoClient(uri)
3. db = client["catalog_db"]
4. products_collection = db["products"]
5. (Optional) Create text index on "name": products_collection.create_index([("name", "text")])
6. Export products_collection for use in routes.py
```

**Connection handling:**
- No connection pooling config in v1 (use MongoClient defaults).
- No retry logic; fail fast on connection errors.

---

### FastAPI app setup (`main.py`)

```
1. app = FastAPI(title="Catalog Service", version="1.0.0", docs_url="/docs")
2. Include routes from routes.py: app.include_router(router)
3. IF __name__ == "__main__":
     uvicorn.run(app, host="0.0.0.0", port=8002)
```

**Port allocation:**
- Catalog service runs on port `8002` (identity on 8001, cart on 8003, etc.).
- Configurable via environment variable `PORT` (default: 8002).

---

## Tests

*(Matches breakdown Verification Approach count: 6 tests)*

### Test 1: Search returns matching products
**Setup:** Seed `catalog_db.products` with:
- `{name: "Widget", price_minor_units: 1000, sellable: true}`
- `{name: "Gadget", price_minor_units: 2000, sellable: false}`

**Action:** `GET /products/search?q=widget`.

**Assert:**
- Response status 200
- Response body contains `{products: [{id: "...", name: "Widget", price_minor_units: 1000, sellable: true}]}`
- Only one product returned (case-insensitive match)

**Verification:** AC-1 (find a product returns name, price, sellable flag)

---

### Test 2: Fetch sellable product
**Setup:** Seed `catalog_db.products` with `{name: "Widget", price_minor_units: 1000, sellable: true}`; capture its `_id`.

**Action:** `GET /products/{product_id}`.

**Assert:**
- Response status 200
- Response body contains `{id: "<id>", name: "Widget", price_minor_units: 1000, sellable: true}`

**Verification:** AC-1 (fetch product returns name, price, sellable flag indicating it can be added to cart)

---

### Test 3: Fetch unsellable product
**Setup:** Seed `catalog_db.products` with `{name: "Gadget", price_minor_units: 2000, sellable: false}`; capture its `_id`.

**Action:** `GET /products/{product_id}`.

**Assert:**
- Response status 200
- Response body contains `{id: "<id>", name: "Gadget", price_minor_units: 2000, sellable: false}`
- Price is returned even though `sellable: false`

**Verification:** AC-2 (unsellable product price still returned; cart service will enforce add refusal)

---

### Test 4: Product not found returns 404
**Action:** `GET /products/507f1f77bcf86cd799439011` (non-existent ObjectId).

**Assert:**
- Response status 404
- Response body contains `{detail: "Product not found"}`

**Verification:** Product fetch validation (edge case clarification)

---

### Test 5: Swagger UI accessible
**Setup:** Start catalog service on port 8002.

**Action:** HTTP GET `http://localhost:8002/docs`.

**Assert:**
- Response status 200
- Response HTML contains "Catalog Service" title
- Swagger UI lists `/products/search` and `/products/{product_id}` endpoints
- Both endpoints can be expanded and show request/response schemas

**Verification:** AC-11 (Swagger UI for every service)

---

### Test 6: Database isolation
**Setup:** Start catalog service. Seed `catalog_db.products` with one product.

**Action:** Query MongoDB to list all databases and collections accessed by catalog service.

**Assert:**
- Only `catalog_db.products` exists and contains data
- No other databases (`identity_db`, `cart_db`, etc.) are created or accessed by this service
- Catalog service does not call any other service's HTTP endpoints

**Verification:** AC-10 (services do not share databases)

---

## HLD Outcomes Binding This Design

From `hld.md`:

- **AD-1 (Data isolation):** Catalog service MUST use only `catalog_db`. No access to `identity_db`, `cart_db`, etc. Verified by Test 6 (database isolation check).
- **AD-2 (HTTP-only inter-service communication):** Catalog does not call other services. Other services (cart) will call catalog via HTTP `GET /products/{product_id}` to verify sellability and fetch price.
- **AD-3 (Price authority):** Catalog is the source of truth for product price. Cart snapshots the price at add-time by calling `GET /products/{product_id}`.
- **AD-5 (Money representation):** All prices stored and returned as `price_minor_units` (integer). No floating-point arithmetic.
- **AD-8 (Service stack split):** Catalog is Python 3.12 + FastAPI (not Node.js). `requirements.txt` specifies dependencies.
- **AD-9 (Swagger UI per service):** `/docs` endpoint enabled in `main.py`. Verified by Test 5.

**Database schema from HLD:**
```json
{
  "_id": "ObjectId",
  "name": "string",
  "price_minor_units": "integer",
  "sellable": "boolean"
}
```

**No deviations.** The schema and API contracts match HLD exactly.

---

## Known Gaps and Future Work

- **Search ranking:** No relevance scoring or sorting in v1 (results returned in insertion order). Mark with `[SEARCH_RANKING_DEFERRED]`.
- **Pagination:** Search returns all matches. Future: add `limit` and `skip` query parameters. Mark with `[PAGINATION_DEFERRED]`.
- **Full-text search:** Uses regex in v1. Future: MongoDB Atlas Search or Elasticsearch. Mark with `[FULLTEXT_DEFERRED]`.
- **Product mutations:** No endpoints to create, update, or delete products (out of scope). Mark with `[CATALOG_MANAGEMENT_DEFERRED]`.
- **Stock levels:** `sellable` is a boolean flag; no inventory quantity tracking. Mark with `[INVENTORY_DEFERRED]`.
- **Product categories/tags:** Not modeled in v1. Future: add `categories` array field. Mark with `[CATEGORIES_DEFERRED]`.
- **Image URLs:** Not included in product schema. Future: add `image_url` field. Mark with `[IMAGES_DEFERRED]`.
- **Currency:** Implicit (single merchant, single currency). No `currency` field in v1. Mark with `[CURRENCY_DEFERRED]`.
- **Authentication/authorization:** All endpoints are public. Mark in Swagger docs as "public, no auth". Mark with `[AUTH_DEFERRED]`.
- **Logging:** No structured logging (print statements only in v1). Mark with `[LOGGING_DEFERRED]`.
- **Environment config:** `MONGODB_URI` and `PORT` read from environment; no `.env` file loading in v1 (manual setup required). Mark with `[CONFIG_DEFERRED]`.

---

**End of LLD.**
