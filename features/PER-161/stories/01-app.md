# LLD — Story 01-app: Identity Service — Sign-in and shopper ID

**Repo:** `app`

---

## Acceptance Criteria (from requirements and breakdown)

### AC-6: Signed-in checkout uses the same total
*(Identity provides the shopper ID that checkout references.)*

**Given** a signed-in shopper with a cart

**When** the client checks out

**Then** the amount sent to payment equals the cart total, and the order is stored against that shopper id

---

### AC-10: Services do not share databases
*(Identity owns `identity_db`; no other service reads or writes it.)*

**Given** a completed purchase

**When** each service is inspected

**Then** catalog, cart, checkout, payment, order, and identity each read and write only their own data

---

### AC-11: Swagger for every service
*(Identity exposes `/docs`.)*

**Given** a running service

**When** a person opens that service's Swagger UI

**Then** every endpoint for that service is listed and can be called from the page

---

## File Plan

All paths under repo root `/home/goos/projects/ecom`. The repo tree is currently empty.

**New files:**

- `app/identity/main.py` — FastAPI application entry point, enables Swagger UI at `/docs`
- `app/identity/requirements.txt` — Python dependencies: `fastapi`, `uvicorn`, `pymongo`, `pydantic`
- `app/identity/models.py` — Pydantic models for request/response validation
- `app/identity/routes.py` — API route handlers for `/sign-in` and `/shoppers/{shopper_id}`
- `app/identity/db.py` — MongoDB connection to `identity_db`, reference to `shoppers` collection

**Directory structure:**

```
app/
└── identity/
    ├── main.py
    ├── requirements.txt
    ├── models.py
    ├── routes.py
    └── db.py
```

---

## Interfaces

### Pydantic Models (`models.py`)

#### `SignInRequest`
**Fields:**
- `email: str` — shopper email
- `password: str` — plaintext password (v1 stub; no real hashing)

**Validation:**
- `email` must be a valid email format (Pydantic `EmailStr`)
- `password` must be non-empty

---

#### `SignInResponse`
**Fields:**
- `shopper_id: str` — MongoDB `ObjectId` as string
- `token: str` — stub token (v1: hardcoded string or simple JWT stub)

---

#### `ShopperResponse`
**Fields:**
- `shopper_id: str` — MongoDB `ObjectId` as string
- `email: str` — shopper email

---

### API Routes (`routes.py`)

#### `POST /sign-in`
**Request body:** `SignInRequest`

**Response:** `SignInResponse` (200 OK) or `401 Unauthorized`

**Logic:**
1. Query `shoppers` collection for a document matching `email`.
2. If not found, return `401 Unauthorized` with `{detail: "Invalid credentials"}`.
3. If found, compare `password` field (plaintext in v1; no bcrypt).
4. If password does not match, return `401 Unauthorized`.
5. If password matches, return `{shopper_id: str(doc["_id"]), token: "stub_token_v1"}`.

**Error cases:**
- Email not found → 401
- Password mismatch → 401
- Invalid request body → 422 (Pydantic validation)

---

#### `GET /shoppers/{shopper_id}`
**Path parameter:** `shopper_id` (string)

**Response:** `ShopperResponse` (200 OK) or `404 Not Found`

**Logic:**
1. Convert `shopper_id` string to MongoDB `ObjectId`.
2. Query `shoppers` collection for `_id == ObjectId(shopper_id)`.
3. If not found, return `404 Not Found` with `{detail: "Shopper not found"}`.
4. If found, return `{shopper_id: str(doc["_id"]), email: doc["email"]}`.

**Error cases:**
- Invalid `ObjectId` format → 400 Bad Request
- Shopper not found → 404

---

### MongoDB Schema (`db.py`)

**Database:** `identity_db`

**Collection:** `shoppers`

**Document structure:**
```json
{
  "_id": "ObjectId",
  "email": "string (unique)",
  "password": "string (plaintext in v1)"
}
```

**Indexes:**
- Unique index on `email`

**Connection:**
- MongoDB URI from environment variable `MONGODB_URI` (default: `mongodb://localhost:27017`)
- Database name: `identity_db`
- Collection reference: `db.shoppers`

---

## Algorithm Notes

### Sign-in flow (`POST /sign-in`)

```
1. Parse request body → SignInRequest(email, password)
2. shoppers.find_one({"email": email})
3. IF not found:
     RETURN 401 Unauthorized
4. IF doc["password"] != password:
     RETURN 401 Unauthorized
5. RETURN SignInResponse(shopper_id=str(doc["_id"]), token="stub_token_v1")
```

**Notes:**
- No password hashing in v1 (plaintext comparison).
- Token is a stub string; no JWT generation or validation.
- Mark password handling with `[STUB_AUTH]` comment for future replacement.

---

### Shopper lookup flow (`GET /shoppers/{shopper_id}`)

```
1. TRY:
     oid = ObjectId(shopper_id)
   EXCEPT InvalidId:
     RETURN 400 Bad Request
2. shoppers.find_one({"_id": oid})
3. IF not found:
     RETURN 404 Not Found
4. RETURN ShopperResponse(shopper_id=str(doc["_id"]), email=doc["email"])
```

**Error handling:**
- Invalid `ObjectId` format → catch `bson.errors.InvalidId`, return 400.
- Missing document → return 404.

---

### Database initialization (`db.py`)

```
1. Read MONGODB_URI from environment (default: "mongodb://localhost:27017")
2. Create MongoClient(uri)
3. db = client["identity_db"]
4. shoppers_collection = db["shoppers"]
5. Create unique index on "email": shoppers_collection.create_index("email", unique=True)
6. Export shoppers_collection for use in routes.py
```

**Connection handling:**
- No connection pooling config in v1 (use MongoClient defaults).
- No retry logic; fail fast on connection errors.

---

### FastAPI app setup (`main.py`)

```
1. app = FastAPI(title="Identity Service", version="1.0.0", docs_url="/docs")
2. Include routes from routes.py: app.include_router(router)
3. IF __name__ == "__main__":
     uvicorn.run(app, host="0.0.0.0", port=8001)
```

**Port allocation:**
- Identity service runs on port `8001` (catalog on 8002, cart on 8003, etc.).
- Configurable via environment variable `PORT` (default: 8001).

---

## Tests

*(Matches breakdown Verification Approach count: 5 tests)*

### Test 1: Sign-in success
**Setup:** Seed `identity_db.shoppers` with `{email: "test@example.com", password: "password123"}`.

**Action:** `POST /sign-in` with `{email: "test@example.com", password: "password123"}`.

**Assert:**
- Response status 200
- Response body contains `shopper_id` (valid ObjectId string)
- Response body contains `token: "stub_token_v1"`

**Verification:** AC-6 (identity provides shopper ID for signed-in checkout)

---

### Test 2: Sign-in with invalid credentials
**Setup:** Seed `identity_db.shoppers` with `{email: "test@example.com", password: "password123"}`.

**Action:** `POST /sign-in` with `{email: "test@example.com", password: "wrongpassword"}`.

**Assert:**
- Response status 401
- Response body contains `{detail: "Invalid credentials"}`

**Verification:** Sign-in validation (edge case clarification of AC-6)

---

### Test 3: Shopper lookup success
**Setup:** Seed `identity_db.shoppers` with a document; capture its `_id`.

**Action:** `GET /shoppers/{shopper_id}`.

**Assert:**
- Response status 200
- Response body contains `{shopper_id: "<id>", email: "test@example.com"}`

**Verification:** AC-6 (checkout and order can retrieve shopper data)

---

### Test 4: Shopper lookup with invalid ID
**Action:** `GET /shoppers/invalid-id-format`.

**Assert:**
- Response status 400
- Response body contains error detail about invalid ObjectId

**Verification:** Shopper lookup validation (edge case clarification)

---

### Test 5: Swagger UI accessible
**Setup:** Start identity service on port 8001.

**Action:** HTTP GET `http://localhost:8001/docs`.

**Assert:**
- Response status 200
- Response HTML contains "Identity Service" title
- Swagger UI lists `/sign-in` and `/shoppers/{shopper_id}` endpoints
- Both endpoints can be expanded and show request/response schemas

**Verification:** AC-11 (Swagger UI for every service)

---

## HLD Outcomes Binding This Design

From `hld.md`:

- **AD-1 (Data isolation):** Identity service MUST use only `identity_db`. No access to `catalog_db`, `cart_db`, etc. Verified by database isolation check (implicit in Test 5 — only `identity_db` exists).
- **AD-2 (HTTP-only inter-service communication):** Identity does not call other services in v1; checkout and order will call identity via HTTP to verify shopper existence (future; not in this story).
- **AD-8 (Service stack split):** Identity is Python 3.12 + FastAPI (not Node.js). `requirements.txt` specifies dependencies.
- **AD-9 (Swagger UI per service):** `/docs` endpoint enabled in `main.py`. Verified by Test 5.

**Database schema from HLD:**
```json
{
  "_id": "ObjectId (shopper_id)",
  "email": "string",
  "password_hash": "string"
}
```

**Deviation:** V1 uses plaintext `password` field instead of `password_hash` (stub authentication). Mark with `[STUB_AUTH]` comment. Future work: bcrypt hashing.

---

## Known Gaps and Future Work

- **Authentication:** No real password hashing (plaintext comparison). Mark all password logic with `[STUB_AUTH]`.
- **Token generation:** Token is hardcoded `"stub_token_v1"`. No JWT, no expiration, no validation. Mark with `[STUB_TOKEN]`.
- **Authorization:** No role-based access control. All endpoints are public. Mark in Swagger docs.
- **Email uniqueness enforcement:** Relies on MongoDB unique index; no application-level duplicate check before insert (acceptable for v1).
- **Connection retry:** No retry logic on MongoDB connection failure (fail fast in v1).
- **Environment config:** `MONGODB_URI` and `PORT` read from environment; no `.env` file loading in v1 (manual setup required).
- **Logging:** No structured logging (print statements only in v1).

---

**End of LLD.**
