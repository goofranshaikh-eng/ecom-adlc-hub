# Breakdown — A shopper browses a catalog, checks out, and receives a paid order

## Overview

This epic delivers a complete multi-service e-commerce backend with six independent services: **identity** (sign-in), **catalog** (products and pricing), **cart** (shopping cart with totals), **checkout** (orchestration), **payment** (test double), and **order** (paid order storage). Each service owns its data in a separate MongoDB database and exposes HTTP APIs with Swagger UI. The client can browse products, add them to a cart, check out as a guest or signed-in user, and retrieve the resulting paid order.

## Dependencies

**Before this epic:**
- None. This is the opening epic.

**After this epic:**
- Future work: tax, shipping, coupons, inventory management, returns, admin UI (all deferred).

## Architecture summary

Six services communicate only via HTTP. No service reads or writes another service's database. Money is represented as integers in minor units (e.g., cents). Catalog is the source of truth for product price; cart snapshots the price at add-time. Checkout orchestrates cart → payment → order, empties the cart on payment acceptance, and leaves it unchanged on refusal. Payment is a deterministic test double. Order stores the final paid order. Identity provides shopper IDs for signed-in users; guests provide email and ship-to address at checkout.

**Service stack:**
- **Python services** (identity, catalog, cart, checkout, payment): Python 3.12 + FastAPI + MongoDB
- **Node.js service** (order): Node.js + Express (or equivalent) + MongoDB
- **Swagger:** Every service exposes `/docs`.

**Repo:** `app` (one directory per service).

## Allowed paths

All new files. From `repo-tree/app`: root is `/home/goos/projects/ecom`, currently empty.

**New directories and files:**
```
app/
├── identity/
│   ├── main.py
│   ├── requirements.txt
│   ├── models.py
│   ├── routes.py
│   └── db.py
├── catalog/
│   ├── main.py
│   ├── requirements.txt
│   ├── models.py
│   ├── routes.py
│   └── db.py
├── cart/
│   ├── main.py
│   ├── requirements.txt
│   ├── models.py
│   ├── routes.py
│   └── db.py
├── checkout/
│   ├── main.py
│   ├── requirements.txt
│   ├── models.py
│   ├── routes.py
│   └── db.py
├── payment/
│   ├── main.py
│   ├── requirements.txt
│   ├── models.py
│   ├── routes.py
│   └── db.py
└── order/
    ├── index.js
    ├── package.json
    ├── models.js
    ├── routes.js
    └── db.js
```

Each story below specifies which service directory and files it creates or modifies.

---

## Story breakdown

### Story 1: Identity Service — Sign-in and shopper ID

**Repo:** `app`

**Description:**
Create the identity service: sign-in endpoint that returns a shopper ID, and a shopper lookup endpoint. No real authentication in v1 (stub token). This service owns the `identity_db` MongoDB database and the `shoppers` collection. Expose Swagger UI at `/docs`.

**Covers:**
- AC-6 (signed-in checkout uses shopper ID)
- AC-11 (Swagger UI)

**Dependencies:**
- None. This is the first service.

**After:**
- Stories 2, 3 can start (catalog and cart do not depend on identity, but checkout will need it).

**Verification Approach:**
1. Start the identity service.
2. Call `POST /sign-in` with `{email, password}` → verify response includes `{shopper_id, token}`.
3. Call `GET /shoppers/{shopper_id}` → verify response includes `{shopper_id, email}`.
4. Open `http://localhost:<port>/docs` → verify Swagger UI lists both endpoints and can execute them.

**Verification checks (see LLD stories/01-app.md Tests section):**
1. Sign-in success (AC-6)
2. Sign-in with invalid credentials (edge case clarification of AC-6)
3. Shopper lookup success (AC-6)
4. Shopper lookup with invalid ID (edge case clarification of AC-6)
5. Swagger UI accessible (AC-11)

**Test count:** 5 tests

**Allowed paths:**
- `app/identity/main.py` — FastAPI app, Swagger enabled
- `app/identity/requirements.txt` — `fastapi`, `uvicorn`, `pymongo`, `pydantic`
- `app/identity/models.py` — Pydantic models for `SignInRequest`, `SignInResponse`, `ShopperResponse`
- `app/identity/routes.py` — `/sign-in`, `/shoppers/{shopper_id}`
- `app/identity/db.py` — MongoDB connection to `identity_db`, `shoppers` collection reference

---

### Story 2: Catalog Service — Products, prices, and sellability

**Repo:** `app`

**Description:**
Create the catalog service: search and fetch product endpoints. Each product has a name, price (integer, minor units), and a sellable boolean. This service owns the `catalog_db` MongoDB database and the `products` collection. Expose Swagger UI at `/docs`.

**Covers:**
- AC-1 (find a product)
- AC-2 (unsellable product price still returned)
- AC-10 (catalog owns its database)
- AC-11 (Swagger UI)

**Dependencies:**
- None. Catalog is independent.

**After:**
- Story 3 (cart) needs catalog to verify product sellability and fetch price.

**Verification Approach:**
1. Seed `catalog_db.products` with one sellable and one unsellable product.
2. Call `GET /products/search?q=<name>` → verify response includes matching products with `{id, name, price_minor_units, sellable}`.
3. Call `GET /products/{product_id}` for sellable product → verify response includes name, price, `sellable: true`.
4. Call `GET /products/{product_id}` for unsellable product → verify response includes name, price, `sellable: false`.
5. Open Swagger UI → verify both endpoints listed and executable.
6. Inspect MongoDB → verify only `catalog_db` is accessed by this service.

**Verification checks (see LLD stories/02-app.md Tests section):**
1. Search returns matching products (AC-1)
2. Fetch sellable product (AC-1)
3. Fetch unsellable product (AC-2)
4. Product not found returns 404 (edge case clarification of AC-1)
5. Swagger UI accessible (AC-11)
6. Database isolation (AC-10)

**Test count:** 6 tests

**Allowed paths:**
- `app/catalog/main.py` — FastAPI app, Swagger enabled
- `app/catalog/requirements.txt` — `fastapi`, `uvicorn`, `pymongo`, `pydantic`
- `app/catalog/models.py` — Pydantic models for `Product`, `ProductSearchResponse`
- `app/catalog/routes.py` — `/products/search`, `/products/{product_id}`
- `app/catalog/db.py` — MongoDB connection to `catalog_db`, `products` collection reference

---

### Story 3: Cart Service — Add items, update quantity, compute totals

**Repo:** `app`

**Description:**
Create the cart service: create cart, add item (with sellability check via catalog HTTP call), update item quantity, get cart with total, delete cart. Cart snapshots the catalog price at add-time and computes total as sum of `price_snapshot_minor_units * quantity`. When quantity is zero, remove the line. Reject adding unsellable products with a reason. This service owns the `cart_db` MongoDB database and the `carts` collection. Expose Swagger UI at `/docs`.

**Covers:**
- AC-2 (unsellable product add refused with reason)
- AC-3 (cart total is catalog price × quantity)
- AC-4 (quantity change updates total; zero removes line)
- AC-10 (cart owns its database)
- AC-11 (Swagger UI)

**Dependencies:**
- Story 2 (catalog service must be running to verify sellability and fetch price).

**After:**
- Story 4 (checkout) needs cart to get the cart total.

**Verification Approach:**
1. Start catalog and cart services.
2. Call `POST /cart` → verify `{cart_id}` returned.
3. Call `POST /cart/{cart_id}/items` with sellable product and quantity 2 → verify cart response includes line with `quantity: 2`, `price_snapshot_minor_units` from catalog, and `total_minor_units = price * 2`.
4. Call `POST /cart/{cart_id}/items` with unsellable product → verify `{error: "Product cannot be added", reason: "..."}` and cart unchanged.
5. Call `PATCH /cart/{cart_id}/items/{product_id}` with `quantity: 3` → verify line updated, total recalculated.
6. Call `PATCH /cart/{cart_id}/items/{product_id}` with `quantity: 0` → verify line removed, total updated.
7. Call `GET /cart/{cart_id}` → verify current state matches.
8. Open Swagger UI → verify all endpoints listed.
9. Inspect MongoDB → verify only `cart_db` accessed.

**Verification checks (see LLD stories/03-app.md Tests section):**
1. Create cart (prerequisite for all AC)
2. Add sellable item to cart (AC-3)
3. Add unsellable item refused (AC-2)
4. Update item quantity (AC-4)
5. Remove line with zero quantity (AC-4)
6. Get cart returns current state (AC-3, AC-4)
7. Delete cart (required for AC-8: checkout empties cart)
8. Swagger UI accessible (AC-11)
9. Database isolation (AC-10)

**Test count:** 9 tests

**Allowed paths:**
- `app/cart/main.py` — FastAPI app, Swagger enabled
- `app/cart/requirements.txt` — `fastapi`, `uvicorn`, `pymongo`, `pydantic`, `httpx` (for calling catalog)
- `app/cart/models.py` — Pydantic models for `CartResponse`, `AddItemRequest`, `UpdateItemRequest`
- `app/cart/routes.py` — `/cart`, `/cart/{cart_id}`, `/cart/{cart_id}/items`, `/cart/{cart_id}/items/{product_id}`
- `app/cart/db.py` — MongoDB connection to `cart_db`, `carts` collection reference

---

### Story 4: Checkout Service — Orchestrate cart → payment → order

**Repo:** `app`

**Description:**
Create the checkout service: `POST /checkout` endpoint that orchestrates the flow. Accepts `{cart_id, shopper_id?, email?, ship_to?}`. Calls cart to get total and lines, calls payment to charge, and on payment acceptance calls order to store the order and cart to delete the cart. On payment refusal, returns the refusal reason and leaves cart unchanged. This service owns the `checkout_db` MongoDB database (may be empty or used for idempotency logs). Expose Swagger UI at `/docs`.

**Covers:**
- AC-5 (guest checkout)
- AC-6 (signed-in checkout)
- AC-7 (payment refused, cart unchanged)
- AC-8 (payment accepted, order created, cart emptied)
- AC-11 (Swagger UI)

**Dependencies:**
- Story 3 (cart service must be running)
- Story 5 (payment service must be running)
- Story 6 (order service must be running)
- Story 1 (identity service must be running for signed-in flow)

**After:**
- All other stories complete. This is the final integration story.

**Verification Approach:**
1. Start all services (identity, catalog, cart, checkout, payment, order).
2. Create a cart with items.
3. Call `POST /checkout` with `{cart_id, email, ship_to}` (guest) and payment accepts → verify response `{order_id, status: "paid"}`, cart is empty, order exists in order service.
4. Create another cart, call `POST /checkout` with `{cart_id, shopper_id}` (signed-in) and payment accepts → verify order includes `shopper_id`, cart is empty.
5. Create a cart, call `POST /checkout` with payment refusal trigger (e.g., amount >= 100000 minor units or card ending in "0000") → verify response `{status: "refused", reason: "..."}` and cart unchanged.
6. Open Swagger UI → verify `/checkout` endpoint listed.

**Verification checks (see LLD stories/04-app.md Tests section):**
1. Guest checkout success (AC-5, AC-8)
2. Signed-in checkout success (AC-6, AC-8)
3. Payment refused (AC-7)
4. Cart unchanged on payment refusal (AC-7)
5. Cart emptied on payment success (AC-8)
6. Order created on payment success (AC-8)
7. Swagger UI accessible (AC-11)

**Test count:** 7 tests

**Allowed paths:**
- `app/checkout/main.py` — FastAPI app, Swagger enabled
- `app/checkout/requirements.txt` — `fastapi`, `uvicorn`, `pymongo`, `pydantic`, `httpx` (for calling cart, payment, order)
- `app/checkout/models.py` — Pydantic models for `CheckoutRequest`, `CheckoutResponse`
- `app/checkout/routes.py` — `/checkout`
- `app/checkout/db.py` — MongoDB connection to `checkout_db`

---

### Story 5: Payment Service — Test double for charge acceptance/refusal

**Repo:** `app`

**Description:**
Create the payment service: `POST /charge` endpoint that accepts `{amount_minor_units, card?}` and returns `{status: "accepted", reference}` or `{status: "refused", reason}`. Deterministic test double logic: accept if `amount_minor_units < 100000` and card number (if provided) does not end in `"0000"`; refuse otherwise. Store charge records in `payment_db.charges`. Expose Swagger UI at `/docs`.

**Covers:**
- AC-7 (payment refused)
- AC-8 (payment accepted, reference returned)
- AC-10 (payment owns its database)
- AC-11 (Swagger UI)

**Dependencies:**
- None. Payment is independent.

**After:**
- Story 4 (checkout) needs payment.

**Verification Approach:**
1. Start payment service.
2. Call `POST /charge` with `{amount_minor_units: 5000}` → verify `{status: "accepted", reference: "<UUID or stub>"}`.
3. Call `POST /charge` with `{amount_minor_units: 150000}` → verify `{status: "refused", reason: "..."}`.
4. Call `POST /charge` with `{amount_minor_units: 5000, card: "1234567890000000"}` (ends in "0000") → verify `{status: "refused", reason: "..."}`.
5. Open Swagger UI → verify `/charge` endpoint listed.
6. Inspect MongoDB → verify charge records stored in `payment_db.charges` and no other service's data accessed.

**Verification checks (see LLD stories/05-app.md Tests section):**
1. Charge accepted (low amount) (AC-8)
2. Charge refused (high amount) (AC-7)
3. Charge refused (card ending in "0000") (AC-7)
4. Charge record stored in database (AC-10)
5. Swagger UI accessible (AC-11)
6. Database isolation (AC-10)

**Test count:** 6 tests

**Allowed paths:**
- `app/payment/main.py` — FastAPI app, Swagger enabled
- `app/payment/requirements.txt` — `fastapi`, `uvicorn`, `pymongo`, `pydantic`
- `app/payment/models.py` — Pydantic models for `ChargeRequest`, `ChargeResponse`
- `app/payment/routes.py` — `/charge`
- `app/payment/db.py` — MongoDB connection to `payment_db`, `charges` collection reference

---

### Story 6: Order Service — Store and retrieve paid orders (Node.js)

**Repo:** `app`

**Description:**
Create the order service in Node.js: `POST /orders` endpoint to create an order from checkout, and `GET /orders/{order_id}` to retrieve it. Order includes `{shopper_id?, email?, ship_to, lines, total_minor_units, payment_reference, created_at}`. This service owns the `order_db` MongoDB database and the `orders` collection. Expose Swagger UI at `/docs` using `swagger-ui-express` or equivalent.

**Covers:**
- AC-8 (order created with lines, total, payment reference, address)
- AC-9 (look up the order)
- AC-10 (order owns its database)
- AC-11 (Swagger UI)

**Dependencies:**
- None. Order is independent (checkout calls it, but order does not depend on checkout to start).

**After:**
- Story 4 (checkout) needs order.

**Verification Approach:**
1. Start order service.
2. Call `POST /orders` with `{email, ship_to, lines, total_minor_units, payment_reference}` (guest order) → verify `{order_id}` returned.
3. Call `POST /orders` with `{shopper_id, ship_to, lines, total_minor_units, payment_reference}` (signed-in order) → verify `{order_id}` returned.
4. Call `GET /orders/{order_id}` → verify response includes all fields: `order_id`, `email` or `shopper_id`, `ship_to`, `lines`, `total_minor_units`, `payment_reference`, `created_at`.
5. Open Swagger UI → verify both endpoints listed and executable.
6. Inspect MongoDB → verify order stored in `order_db.orders` and no other service's data accessed.

**Verification checks (see LLD stories/06-app.md Tests section):**
1. Create guest order (AC-8)
2. Create signed-in order (AC-8)
3. Retrieve order (AC-9)
4. Order not found returns 404 (edge case clarification of AC-9)
5. Swagger UI accessible (AC-11)
6. Database isolation (AC-10)

**Test count:** 6 tests

**Allowed paths:**
- `app/order/index.js` — Express app, Swagger UI enabled
- `app/order/package.json` — `express`, `mongodb`, `swagger-ui-express`, `swagger-jsdoc`
- `app/order/models.js` — Order schema validation (Joi or Mongoose)
- `app/order/routes.js` — `/orders`, `/orders/{order_id}`
- `app/order/db.js` — MongoDB connection to `order_db`, `orders` collection reference

---

## Build order

1. **Story 1** (Identity) — no dependencies
2. **Story 2** (Catalog) — no dependencies
3. **Story 5** (Payment) — no dependencies
4. **Story 6** (Order) — no dependencies
5. **Story 3** (Cart) — after Story 2 (needs catalog running)
6. **Story 4** (Checkout) — after Stories 1, 3, 5, 6 (needs all other services running)

Stories 1, 2, 5, 6 can start in parallel. Story 3 waits for Story 2. Story 4 is last.

---

## Risks and open questions

| Risk | Mitigation |
|------|-----------|
| **Price drift between cart view and checkout** | Cart snapshots price at add-time. Checkout sends the cart total without recalculating. Deferred: price-lock or re-verification. |
| **Network failure in checkout orchestration** | Deferred: idempotency keys and retry logic. V1 accepts partial state as a known gap. Mark in code with `[KNOWN_GAP]`. |
| **Concurrent cart updates** | V1 uses last-write-wins. Deferred: optimistic locking or versioning. |
| **Payment test double too simplistic** | Accepted for v1. Mark with `[TEST_DOUBLE]` in code. Future: add realistic failure modes. |
| **No authentication/authorization** | Deferred. V1 trusts the caller. Mark endpoints as "public, no auth" in Swagger. |
| **No database transactions** | Accepted. Document in code where atomicity would matter. Future: sagas or 2PC. |

**Open questions:** None. All acceptance criteria and service boundaries are defined.

---

## Total test count

- **Story 1 (Identity):** 5 tests
- **Story 2 (Catalog):** 6 tests
- **Story 3 (Cart):** 9 tests
- **Story 4 (Checkout):** 7 tests
- **Story 5 (Payment):** 6 tests
- **Story 6 (Order):** 6 tests

**Total:** 39 tests

Each story's LLD and final test suite must match the test count listed in its Verification Approach and the detailed verification checks listed above. All 11 acceptance criteria are covered across the six stories as documented in the "Covers" section of each story and the verification checks.

---

**End of Breakdown.**
