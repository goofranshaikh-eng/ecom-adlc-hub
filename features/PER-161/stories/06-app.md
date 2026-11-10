# LLD — Story 06-app: Order Service — Store and retrieve paid orders (Node.js)

**Repo:** `app`

---

## Acceptance Criteria (from requirements and breakdown)

### AC-8: Payment accepted

**Given** a cart at checkout

**When** payment accepts the charge

**Then** an order exists with the lines, the total charged, a payment reference, and the ship-to address, and the cart is empty

*(Order service stores the order data sent by checkout.)*

---

### AC-9: Look up the order

**Given** a paid order

**When** the client fetches it

**Then** the response includes the order id, lines, total, and address that were stored

---

### AC-10: Services do not share databases

**Given** a completed purchase

**When** each service is inspected

**Then** catalog, cart, checkout, payment, order, and identity each read and write only their own data

*(Order owns `order_db`; no other service reads or writes it.)*

---

### AC-11: Swagger for every service

**Given** a running service

**When** a person opens that service's Swagger UI

**Then** every endpoint for that service is listed and can be called from the page

*(Order exposes `/docs`.)*

---

## File Plan

All paths under repo root `/home/goos/projects/ecom`. The repo tree is currently empty.

**New files:**

- `app/order/index.js` — Express application entry point, enables Swagger UI at `/docs`
- `app/order/package.json` — Node.js dependencies: `express`, `mongodb`, `swagger-ui-express`, `swagger-jsdoc`, `joi`
- `app/order/models.js` — Order schema validation using Joi
- `app/order/routes.js` — API route handlers for `/orders` and `/orders/:order_id`
- `app/order/db.js` — MongoDB connection to `order_db`, reference to `orders` collection

**Directory structure:**

```
app/
└── order/
    ├── index.js
    ├── package.json
    ├── models.js
    ├── routes.js
    └── db.js
```

---

## Interfaces

### Joi Schema (`models.js`)

#### `ShipTo`
**Fields:**
- `street: string` — street address (required)
- `city: string` — city name (required)
- `postal_code: string` — postal/zip code (required)
- `country: string` — country name (required)

**Validation:**
- All fields must be non-empty strings

---

#### `OrderLine`
**Fields:**
- `product_id: string` — product identifier (required)
- `quantity: integer` — number of items (required, > 0)
- `price_minor_units: integer` — price per unit in minor units (required, >= 0)

**Validation:**
- `product_id` must be non-empty string
- `quantity` must be integer > 0
- `price_minor_units` must be integer >= 0

---

#### `CreateOrderRequest`
**Fields:**
- `shopper_id: string` — shopper ID from identity service (optional)
- `email: string` — contact email (optional, but required for guest orders)
- `ship_to: ShipTo` — shipping address (required)
- `lines: Array<OrderLine>` — order lines (required, non-empty array)
- `total_minor_units: integer` — total amount charged (required, >= 0)
- `payment_reference: string` — payment reference from payment service (required)

**Validation:**
- At least one of `shopper_id` or `email` must be present
- `ship_to` must be a valid `ShipTo` object
- `lines` must be non-empty array of valid `OrderLine` objects
- `total_minor_units` must be integer >= 0
- `payment_reference` must be non-empty string

---

#### `OrderResponse`
**Fields:**
- `order_id: string` — MongoDB `ObjectId` as string
- `shopper_id: string` — shopper ID (present if provided at creation)
- `email: string` — contact email (present if provided at creation)
- `ship_to: ShipTo` — shipping address
- `lines: Array<OrderLine>` — order lines
- `total_minor_units: integer` — total amount charged
- `payment_reference: string` — payment reference
- `created_at: string` — ISO 8601 timestamp

---

### API Routes (`routes.js`)

#### `POST /orders`
**Request body:** `CreateOrderRequest`

**Response:** `{order_id: string}` (201 Created), `400 Bad Request`, or `500 Internal Server Error`

**Logic:**

1. **Validate request body:**
   - Use Joi schema to validate `CreateOrderRequest`.
   - If validation fails, return `400 Bad Request` with validation error details.

2. **Validate guest vs. signed-in:**
   - If both `shopper_id` and `email` are absent, return `400 Bad Request` with `{error: "Either shopper_id or email is required"}`.

3. **Insert order document:**
   - Create document:
     ```javascript
     {
       shopper_id: req.body.shopper_id (if present),
       email: req.body.email (if present),
       ship_to: req.body.ship_to,
       lines: req.body.lines,
       total_minor_units: req.body.total_minor_units,
       payment_reference: req.body.payment_reference,
       created_at: new Date()
     }
     ```
   - Insert into `orders` collection.
   - Capture `insertedId` from result.

4. **Return order ID:**
   - Return `201 Created` with `{order_id: insertedId.toString()}`.

**Error cases:**
- Validation failure → 400 with Joi error details
- Missing both `shopper_id` and `email` → 400
- Database insertion failure → 500

---

#### `GET /orders/:order_id`
**Path parameter:** `order_id` (string)

**Response:** `OrderResponse` (200 OK) or `404 Not Found`

**Logic:**

1. **Convert order_id to ObjectId:**
   - Try to convert `order_id` string to MongoDB `ObjectId`.
   - If conversion fails (invalid format), return `400 Bad Request` with `{error: "Invalid order_id format"}`.

2. **Query orders collection:**
   - `orders.findOne({_id: ObjectId(order_id)})`.
   - If not found, return `404 Not Found` with `{error: "Order not found"}`.

3. **Return order document:**
   - Convert `_id` to string: `order_id: doc._id.toString()`.
   - Convert `created_at` to ISO string: `created_at: doc.created_at.toISOString()`.
   - Return `OrderResponse` with all fields.

**Error cases:**
- Invalid `ObjectId` format → 400
- Order not found → 404

---

### MongoDB Schema (`db.js`)

**Database:** `order_db`

**Collection:** `orders`

**Document structure:**
```json
{
  "_id": "ObjectId (order_id)",
  "shopper_id": "string (optional)",
  "email": "string (optional, required for guest)",
  "ship_to": {
    "street": "string",
    "city": "string",
    "postal_code": "string",
    "country": "string"
  },
  "lines": [
    {
      "product_id": "string",
      "quantity": "integer (> 0)",
      "price_minor_units": "integer (>= 0)"
    }
  ],
  "total_minor_units": "integer (>= 0)",
  "payment_reference": "string",
  "created_at": "Date"
}
```

**Indexes:**
- Primary key on `_id` (default)
- Optional index on `shopper_id` (for future queries by shopper)
- Optional index on `email` (for guest order lookups)
- Optional index on `created_at` (descending, for order history)

**Connection:**
- MongoDB URI from environment variable `MONGODB_URI` (default: `mongodb://localhost:27017`)
- Database name: `order_db`
- Collection reference: `db.collection('orders')`

---

## Algorithm Notes

### Create order flow (`POST /orders`)

```javascript
1. // Validate request body
   const { error, value } = createOrderSchema.validate(req.body);
   IF error:
     RETURN 400 Bad Request, {error: error.details[0].message}

2. // Validate guest vs. signed-in
   IF !value.shopper_id AND !value.email:
     RETURN 400 Bad Request, {error: "Either shopper_id or email is required"}

3. // Prepare order document
   const orderDoc = {
     shopper_id: value.shopper_id (if present),
     email: value.email (if present),
     ship_to: value.ship_to,
     lines: value.lines,
     total_minor_units: value.total_minor_units,
     payment_reference: value.payment_reference,
     created_at: new Date()
   };

4. // Insert into database
   TRY:
     const result = AWAIT ordersCollection.insertOne(orderDoc);
     const order_id = result.insertedId.toString();
     RETURN 201 Created, {order_id}
   CATCH error:
     RETURN 500 Internal Server Error, {error: "Failed to create order"}
```

**Notes:**
- **Joi validation:** Use `createOrderSchema.validate(req.body, {abortEarly: false})` to collect all validation errors at once.
- **Optional fields:** When constructing `orderDoc`, omit `shopper_id` or `email` if not provided (use object spread or conditional assignment).
- **Date handling:** `created_at` is a native JavaScript `Date` object, stored as MongoDB `ISODate`.
- **Error logging:** Log database insertion errors to console/logger but return generic 500 to client. Mark with `[ERROR_DETAIL_DEFERRED]`.

---

### Get order flow (`GET /orders/:order_id`)

```javascript
1. // Convert order_id to ObjectId
   TRY:
     const oid = new ObjectId(req.params.order_id);
   CATCH error:
     RETURN 400 Bad Request, {error: "Invalid order_id format"}

2. // Query database
   TRY:
     const doc = AWAIT ordersCollection.findOne({_id: oid});
   CATCH error:
     RETURN 500 Internal Server Error, {error: "Database query failed"}

3. IF !doc:
     RETURN 404 Not Found, {error: "Order not found"}

4. // Convert to response format
   const response = {
     order_id: doc._id.toString(),
     shopper_id: doc.shopper_id (if present),
     email: doc.email (if present),
     ship_to: doc.ship_to,
     lines: doc.lines,
     total_minor_units: doc.total_minor_units,
     payment_reference: doc.payment_reference,
     created_at: doc.created_at.toISOString()
   };

5. RETURN 200 OK, response
```

**Notes:**
- **ObjectId import:** `const { ObjectId } = require('mongodb');`
- **Invalid ID handling:** Catch `BSONError` or any error from `new ObjectId()` and return 400.
- **Optional field handling:** Include `shopper_id` or `email` in response only if present in document (use conditional object spread or check `doc.shopper_id !== undefined`).
- **ISO timestamp:** Convert MongoDB `Date` to ISO 8601 string using `.toISOString()`.

---

### Database initialization (`db.js`)

```javascript
1. // Read MongoDB URI from environment
   const uri = process.env.MONGODB_URI || 'mongodb://localhost:27017';

2. // Create MongoDB client
   const { MongoClient } = require('mongodb');
   const client = new MongoClient(uri);

3. // Connect and export collection reference
   let ordersCollection;
   
   async function connectDB() {
     AWAIT client.connect();
     const db = client.db('order_db');
     ordersCollection = db.collection('orders');
     
     // Optional: create indexes
     AWAIT ordersCollection.createIndex({shopper_id: 1});
     AWAIT ordersCollection.createIndex({email: 1});
     AWAIT ordersCollection.createIndex({created_at: -1});
   }

4. // Export collection reference
   module.exports = { connectDB, getOrdersCollection: () => ordersCollection };
```

**Connection handling:**
- **Async initialization:** Call `connectDB()` in `index.js` before starting the server.
- **No connection pooling config:** Use MongoClient defaults in v1.
- **No retry logic:** Fail fast on connection errors. Mark with `[NO_RETRY]`.
- **Index creation:** Optional indexes for `shopper_id`, `email`, and `created_at` (created on first connection). Mark with `[OPTIONAL_INDEXES]`.

---

### Express app setup (`index.js`)

```javascript
1. const express = require('express');
   const swaggerUi = require('swagger-ui-express');
   const swaggerJsdoc = require('swagger-jsdoc');
   const { connectDB } = require('./db');
   const routes = require('./routes');

2. const app = express();
   app.use(express.json());

3. // Swagger setup
   const swaggerOptions = {
     definition: {
       openapi: '3.0.0',
       info: {
         title: 'Order Service',
         version: '1.0.0',
         description: 'Order creation and retrieval API'
       },
       servers: [{url: `http://localhost:${process.env.PORT || 8006}`}]
     },
     apis: ['./routes.js']
   };
   const swaggerSpec = swaggerJsdoc(swaggerOptions);
   app.use('/docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec));

4. // Routes
   app.use('/orders', routes);

5. // Start server
   const PORT = process.env.PORT || 8006;
   connectDB().then(() => {
     app.listen(PORT, () => {
       console.log(`Order service running on port ${PORT}`);
       console.log(`Swagger UI: http://localhost:${PORT}/docs`);
     });
   }).catch(err => {
     console.error('Failed to connect to database:', err);
     process.exit(1);
   });
```

**Port allocation:**
- Order service runs on port `8006` (identity on 8001, catalog on 8002, cart on 8003, checkout on 8004, payment on 8005).
- Configurable via environment variable `PORT` (default: 8006).

**Swagger annotations:**
- Use JSDoc comments in `routes.js` to document endpoints. Example:
  ```javascript
  /**
   * @swagger
   * /orders:
   *   post:
   *     summary: Create a new order
   *     requestBody:
   *       required: true
   *       content:
   *         application/json:
   *           schema:
   *             $ref: '#/components/schemas/CreateOrderRequest'
   *     responses:
   *       201:
   *         description: Order created
   *         content:
   *           application/json:
   *             schema:
   *               type: object
   *               properties:
   *                 order_id:
   *                   type: string
   */
  ```

---

### Package.json

```json
{
  "name": "order-service",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mongodb": "^6.3.0",
    "joi": "^17.11.0",
    "swagger-ui-express": "^5.0.0",
    "swagger-jsdoc": "^6.2.8"
  },
  "devDependencies": {
    "nodemon": "^3.0.2"
  }
}
```

**Notes:**
- **Version pins:** Use latest stable versions as of LLD creation date. Mark with `[VERSION_CURRENT]` for future updates.
- **Dev dependency:** `nodemon` for auto-restart during development (optional).
- **No test framework:** Tests are integration tests run externally (e.g., via Python `pytest` or Postman). Mark with `[TESTS_EXTERNAL]`.

---

## Tests

*(Matches breakdown Verification Approach count: 6 tests)*

### Test 1: Create guest order — verifies AC-8 (order creation)
**Setup:** Start order service on port 8006.

**Action:** `POST http://localhost:8006/orders` with:
```json
{
  "email": "guest@example.com",
  "ship_to": {
    "street": "123 Main St",
    "city": "Springfield",
    "postal_code": "12345",
    "country": "USA"
  },
  "lines": [
    {
      "product_id": "prod1",
      "quantity": 2,
      "price_minor_units": 1000
    }
  ],
  "total_minor_units": 2000,
  "payment_reference": "pay_abc123"
}
```

**Assert:**
- Response status 201
- Response body contains `{order_id: "<ObjectId string>"}` (valid 24-character hex string)
- MongoDB `order_db.orders` contains one document:
  - `_id` matches `order_id`
  - `email == "guest@example.com"`
  - No `shopper_id` field
  - `ship_to` matches request
  - `lines` matches request
  - `total_minor_units == 2000`
  - `payment_reference == "pay_abc123"`
  - `created_at` is a recent timestamp (within 1 second of test execution)

**Verification:** AC-8 (order exists with lines, total charged, payment reference, ship-to address).

---

### Test 2: Create signed-in order — verifies AC-8 (order creation with shopper_id)
**Setup:** Start order service on port 8006.

**Action:** `POST http://localhost:8006/orders` with:
```json
{
  "shopper_id": "shopper123",
  "ship_to": {
    "street": "456 Elm St",
    "city": "Shelbyville",
    "postal_code": "67890",
    "country": "USA"
  },
  "lines": [
    {
      "product_id": "prod2",
      "quantity": 1,
      "price_minor_units": 1500
    }
  ],
  "total_minor_units": 1500,
  "payment_reference": "pay_xyz789"
}
```

**Assert:**
- Response status 201
- Response body contains `{order_id: "<ObjectId string>"}`
- MongoDB `order_db.orders` contains document:
  - `shopper_id == "shopper123"`
  - No `email` field
  - All other fields match request

**Verification:** AC-8 (order stored against shopper_id for signed-in checkout).

---

### Test 3: Retrieve order — verifies AC-9
**Setup:**
1. Start order service on port 8006.
2. Create an order via `POST /orders`, capture `order_id`.

**Action:** `GET http://localhost:8006/orders/{order_id}`.

**Assert:**
- Response status 200
- Response body `OrderResponse` contains all fields:
  - `order_id` matches
  - `email` or `shopper_id` (whichever was provided at creation)
  - `ship_to` object with all fields
  - `lines` array with all line items
  - `total_minor_units` matches
  - `payment_reference` matches
  - `created_at` is ISO 8601 string (e.g., `"2024-01-15T12:34:56.789Z"`)

**Verification:** AC-9 (response includes order id, lines, total, and address that were stored).

---

### Test 4: Order not found returns 404 — verifies AC-9 edge case
**Action:** `GET http://localhost:8006/orders/507f1f77bcf86cd799439011` (non-existent ObjectId).

**Assert:**
- Response status 404
- Response body contains `{error: "Order not found"}`

**Verification:** AC-9 edge case (order lookup validation).

---

### Test 5: Swagger UI accessible — verifies AC-11
**Setup:** Start order service on port 8006.

**Action:** HTTP GET `http://localhost:8006/docs`.

**Assert:**
- Response status 200
- Response HTML contains "Order Service" title
- Swagger UI lists endpoints:
  - `POST /orders`
  - `GET /orders/{order_id}`
- Both endpoints can be expanded and show:
  - `POST /orders`: request schema with all `CreateOrderRequest` fields
  - `GET /orders/{order_id}`: response schema with all `OrderResponse` fields
- Each endpoint can be executed from the Swagger UI (Try it out button works)

**Verification:** AC-11 (Swagger UI for every service; all endpoints listed and callable).

---

### Test 6: Database isolation — verifies AC-10
**Setup:**
1. Start order service.
2. Create multiple orders via `POST /orders`.

**Action:** Query MongoDB to list all databases and collections accessed by order service.

**Assert:**
- Only `order_db.orders` exists and contains data
- No other databases (`identity_db`, `catalog_db`, `cart_db`, `checkout_db`, `payment_db`) are created or accessed by this service
- Order service does not call any other service's HTTP endpoints (verified by code inspection — no `axios`, `node-fetch`, or similar imports; no HTTP client usage in `routes.js` or `index.js`)

**Verification:** AC-10 (order service reads and writes only its own data; no shared databases).

---

## HLD Outcomes Binding This Design

From `hld.md`:

- **AD-1 (Data isolation):** Order service MUST use only `order_db`. No access to `checkout_db`, `payment_db`, `cart_db`, etc. Verified by Test 6 (database isolation check).
- **AD-2 (HTTP-only inter-service communication):** Order does not call other services. Checkout calls order via HTTP `POST /orders`. Verified by algorithm (no cross-service calls in order code).
- **AD-5 (Money representation):** All amounts stored and returned as `total_minor_units` and `price_minor_units` (integers). No floating-point arithmetic. Verified by Joi schema (integer validation) and all tests (amounts are integers).
- **AD-6 (Guest vs. signed-in checkout):** Order accepts either `shopper_id` (signed-in) or `email` (guest) at creation. Validates that at least one is present. Verified by algorithm step 2 and Tests 1-2.
- **AD-8 (Service stack split):** Order is Node.js + Express (not Python/FastAPI). `package.json` specifies dependencies.
- **AD-9 (Swagger UI per service):** `/docs` endpoint enabled in `index.js` using `swagger-ui-express` and `swagger-jsdoc`. Verified by Test 5.

**Database schema from HLD:**
```json
{
  "_id": "ObjectId (order_id)",
  "shopper_id": "string (optional)",
  "email": "string (optional, required for guest)",
  "ship_to": {
    "street": "string",
    "city": "string",
    "postal_code": "string",
    "country": "string"
  },
  "lines": [
    {
      "product_id": "string",
      "quantity": "integer",
      "price_minor_units": "integer"
    }
  ],
  "total_minor_units": "integer",
  "payment_reference": "string",
  "created_at": "ISODate"
}
```

**No deviations.** The schema and API contracts match HLD exactly.

---

## Known Gaps and Future Work

- **No authentication/authorization:** All endpoints are public. Any client can create or retrieve any order. Mark in Swagger docs as "public, no auth". Mark with `[AUTH_DEFERRED]`. Future: validate shopper_id against identity service, require auth token.
- **No order ownership validation:** `GET /orders/:order_id` does not verify that the caller owns the order (no token or shopper_id check). Mark with `[ORDER_OWNERSHIP_DEFERRED]`. Future: require auth token and match against `order.shopper_id` or `order.email`.
- **No pagination:** No `GET /orders` list endpoint. Future: add pagination with `limit`, `skip`, and filters (by shopper_id, date range, status). Mark with `[PAGINATION_DEFERRED]`.
- **No order status:** No `status` field (e.g., "paid", "shipped", "delivered"). All orders are implicitly "paid". Mark with `[ORDER_STATUS_DEFERRED]`. Future: add `status` field and update endpoints.
- **No order updates:** No `PATCH /orders/:order_id` endpoint to update shipping address or status. Mark with `[ORDER_UPDATES_DEFERRED]`. Future: add update logic with audit trail.
- **No order cancellation/refunds:** No `DELETE /orders/:order_id` or cancel endpoint. Mark with `[ORDER_CANCELLATION_DEFERRED]`. Future: coordinate with payment service for refunds.
- **No idempotency:** Duplicate `POST /orders` requests create multiple identical orders. Mark with `[IDEMPOTENCY_DEFERRED]`. Future: add idempotency key header (from checkout), store in separate collection or as unique index.
- **No audit log:** Order mutations (if added) are not logged. Mark with `[AUDIT_LOG_DEFERRED]`. Future: log all changes to `order_db.audit_log`.
- **No order confirmation email:** Order service does not send email to guest or shopper. Mark with `[EMAIL_DEFERRED]`. Future: integrate notification service (trigger email on order creation).
- **No product name/details in order:** Order stores only `product_id`; no product name, description, or image URL. Client must call catalog to fetch display details. Mark with `[ORDER_DISPLAY_DEFERRED]`. Acceptable for v1 (order is data only, no UI).
- **No total validation:** Order service does not verify that `total_minor_units` equals sum of `line.price_minor_units * line.quantity`. Trusts checkout. Mark with `[TOTAL_VALIDATION_DEFERRED]`. Future: add server-side validation and reject mismatches.
- **No order search:** No `GET /orders/search?email=...` endpoint. Mark with `[ORDER_SEARCH_DEFERRED]`. Future: add search by email, shopper_id, payment_reference.
- **No order history:** No `GET /shoppers/:shopper_id/orders` endpoint to list a shopper's order history. Mark with `[ORDER_HISTORY_DEFERRED]`. Future: add with pagination.
- **No ship-to validation:** No validation that `ship_to` is a valid/deliverable address (e.g., via Google Maps API or address verification service). Mark with `[ADDRESS_VALIDATION_DEFERRED]`. Future: integrate address validation.
- **No currency:** Implicit (single merchant, single currency). No `currency` field in order. Mark with `[CURRENCY_DEFERRED]`. Future: add `currency` field (ISO 4217 code) for multi-currency support.
- **No tax/shipping in order:** No `tax_minor_units` or `shipping_minor_units` fields. `total_minor_units` is the full charge. Mark with `[TAX_SHIPPING_DEFERRED]`. Future: add breakdown fields when tax/shipping are implemented.
- **No order expiry/archival:** Orders stored indefinitely. Mark with `[ORDER_ARCHIVAL_DEFERRED]`. Future: add TTL index or archival process for old orders.
- **Logging:** No structured logging (console.log only in v1). Mark with `[LOGGING_DEFERRED]`. Future: add Winston or Pino for structured JSON logs.
- **Environment config:** `MONGODB_URI` and `PORT` read from environment; no `.env` file loading in v1 (manual setup required). Mark with `[CONFIG_DEFERRED]`. Future: add `dotenv` package.
- **Error detail exposure:** 500 errors may expose stack traces in debug mode. Mark with `[ERROR_DETAIL_DEFERRED]`. Future: sanitize errors in production.
- **No connection retry:** MongoDB connection fails fast. No retry logic. Mark with `[NO_RETRY]`. Future: add retry with exponential backoff.
- **No health check endpoint:** No `GET /health` or `GET /status` endpoint for uptime monitoring. Mark with `[HEALTH_CHECK_DEFERRED]`. Future: add health check that pings MongoDB.
- **No graceful shutdown:** Server does not close MongoDB connection on SIGTERM/SIGINT. Mark with `[GRACEFUL_SHUTDOWN_DEFERRED]`. Future: add signal handlers to close client before exit.

---

**End of LLD.**
