# LLD — Story 02-app

**Story:** Order service consumes OrderRequested and publishes OrderCreated  
**Repo:** `app`  
**Type:** Implementation  
**Covers AC:** AC3, AC4 (order side), AC8

---

## Acceptance Criteria

### AC3: Order is created after the order service recovers

**Given** an accepted checkout whose `OrderRequested` message could not be handled because the order service was down  
**When** the order service is running again  
**Then** exactly one order exists with the lines, total, ship-to address, and payment reference from that checkout

---

### AC4: Duplicate message does not create two orders (order side)

**Given** an accepted checkout whose `OrderRequested` message is delivered twice  
**When** the order service consumes both messages  
**Then** there is still one order for that payment reference

---

### AC8: Services do not share databases (order side)

**Given** a completed or refunded checkout  
**When** the order service is inspected  
**Then** order reads and writes only `order_db` (collection: `orders`), and no other service's database is accessed

---

## File Plan

### New Files

1. **`order/consumer.js`**
   - RabbitMQ consumer for `OrderRequested` messages
   - Validates message body with Joi schema
   - Queries `orders` collection by `payment_reference` for idempotency
   - Inserts order if not exists, catches duplicate key error (code 11000)
   - Publishes `OrderCreated` to RabbitMQ after successful insertion
   - Manual message acknowledgement
   - Tests: `test_order_created_from_order_requested()`, `test_duplicate_message_creates_one_order()`, `test_duplicate_key_error_handled()`, `test_order_created_published_after_insert()`

2. **`order/rabbitmq.js`**
   - RabbitMQ connection setup: `connectRabbitMQ()` returns connection and channel
   - Exchange and queue declaration: `setupExchangeAndQueue(channel, exchange, queue, routingKey)`
   - Publisher helper: `publishMessage(channel, exchange, routingKey, payload)` with publisher confirms
   - Constants: `RABBITMQ_URI`, `EXCHANGE_NAME = "ecom.events"`
   - Tests: covered by integration tests that verify RabbitMQ topology

---

### Modified Files

1. **`order/models.js`**
   - Add `payment_reference` field to `createOrderSchema` (line 25, inside `createOrderSchema`)
   - Validation: `payment_reference: Joi.string().trim().required()`
   - Tests: all tests that consume `OrderRequested` or query orders by `payment_reference` validate the schema

2. **`order/db.js`**
   - Add unique index on `payment_reference` in `orders` collection (line 17, inside `connectDB()` after `await client.connect()`)
   - `await db.collection("orders").createIndex({ payment_reference: 1 }, { unique: true })`
   - Tests: `test_duplicate_key_error_handled()`, `test_duplicate_message_creates_one_order()`

3. **`order/index.js`**
   - Import and start RabbitMQ consumer after MongoDB connection (line 48, after `connectDB()` resolves)
   - `const { consumeOrderRequested } = require('./consumer');`
   - Call `consumeOrderRequested()` in the `.then()` block
   - Add graceful shutdown for RabbitMQ connection and channel (after line 56, before `process.exit(1)`)
   - `process.on('SIGTERM', async () => { await closeRabbitMQ(); process.exit(0); })`
   - `process.on('SIGINT', async () => { await closeRabbitMQ(); process.exit(0); })`
   - Tests: `test_order_created_after_service_recovers()` (manual test: stop/start order service)

4. **`order/package.json`**
   - Add `amqplib` dependency (line 15, inside `dependencies`)
   - `"amqplib": "^0.10.3"`
   - Tests: no test validates `package.json` directly; consumer tests fail if `amqplib` is missing

---

## Interfaces

### Database Schema: `orders` collection (modified)

**Collection:** `order_db.orders`

**New Field:**
- `payment_reference` (string, required, unique): Idempotency key. Prevents duplicate orders for the same payment.

**Updated Document:**
```javascript
{
  _id: ObjectId,
  payment_reference: string,  // NEW: idempotency key
  shopper_id: string | null,
  email: string | null,
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
      price_minor_units: number  // mapped from price_snapshot_minor_units in message
    }
  ],
  total_minor_units: number,
  created_at: Date
}
```

**Indexes:**
- `{ payment_reference: 1 }` (unique) — enforces one order per payment reference
- `{ shopper_id: 1 }` (existing, from `order/db.js`)
- `{ email: 1 }` (existing)
- `{ created_at: -1 }` (existing)

---

### RabbitMQ Message Schema: `OrderRequested` (consumed)

**Published by:** Checkout (via relay)  
**Routing key:** `order.requested`  
**Queue:** `order.requested.queue`

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

**Field Notes:**
- `payment_reference` (string, required): Idempotency key. Order checks for existing order by this field before inserting.
- `checkout_id` (string, required): Traceability; not stored in order document.
- `shopper_id` (string, optional): Present for signed-in checkout.
- `email` (string, optional): Present for guest checkout.
- `lines[].price_snapshot_minor_units` (integer, required): Mapped to `price_minor_units` in the order document (see HLD).
- `total_minor_units` (integer, required): Trusted from checkout; no recalculation.
- `timestamp` (string, ISO 8601, required): Event timestamp for observability.

---

### RabbitMQ Message Schema: `OrderCreated` (published)

**Published by:** Order  
**Routing key:** `order.created`  
**Queue:** `order.created.queue`

**Payload:**
```json
{
  "event_type": "OrderCreated",
  "payment_reference": "pay_abc123",
  "checkout_id": "chk_xyz789",
  "order_id": "507f1f77bcf86cd799439011",
  "timestamp": "2025-01-10T14:30:05Z"
}
```

**Field Notes:**
- `payment_reference` (string, required): Same as `OrderRequested`.
- `checkout_id` (string, required): Echoed from `OrderRequested` for correlation.
- `order_id` (string, required): MongoDB ObjectId as hex string (`_id.toString()`).
- `timestamp` (string, ISO 8601, required): Order creation timestamp (`new Date().toISOString()`).

---

### Node.js Module: `order/rabbitmq.js`

**Functions:**

```javascript
async function connectRabbitMQ() {
  /**
   * Connect to RabbitMQ and return { connection, channel }.
   * Uses RABBITMQ_URI environment variable (default: amqp://localhost:5672).
   * Throws on connection failure (no retry in v1).
   */
}

async function setupExchangeAndQueue(channel, exchange, queue, routingKey) {
  /**
   * Declare exchange (type: topic, durable) and queue (durable), bind queue to routing key.
   * channel: amqplib channel
   * exchange: exchange name (default: "ecom.events")
   * queue: queue name (e.g. "order.requested.queue")
   * routingKey: routing key (e.g. "order.requested")
   */
}

async function publishMessage(channel, exchange, routingKey, payload) {
  /**
   * Publish a message to RabbitMQ with publisher confirms.
   * channel: amqplib channel
   * exchange: exchange name
   * routingKey: routing key (e.g. "order.created")
   * payload: message body (object, will be JSON.stringify'd)
   * Sets delivery_mode=2 (persistent).
   * Waits for publisher confirm before returning.
   */
}

async function closeRabbitMQ() {
  /**
   * Close RabbitMQ channel and connection.
   * Called on SIGTERM and SIGINT.
   */
}

module.exports = {
  connectRabbitMQ,
  setupExchangeAndQueue,
  publishMessage,
  closeRabbitMQ,
};
```

**Constants:**
```javascript
const RABBITMQ_URI = process.env.RABBITMQ_URI || 'amqp://localhost:5672';
const EXCHANGE_NAME = 'ecom.events';
```

---

### Node.js Module: `order/consumer.js`

**Entry point:**
```javascript
const { consumeOrderRequested } = require('./consumer');

// Start consumer after MongoDB connection
await consumeOrderRequested();
```

**Main function:**
```javascript
async function consumeOrderRequested() {
  /**
   * Connect to RabbitMQ, subscribe to order.requested.queue, consume messages.
   * For each message:
   *   1. Validate message body with Joi schema (OrderRequested)
   *   2. Query orders collection by payment_reference
   *   3. If order exists: skip insertion, publish OrderCreated with existing order_id, ack message
   *   4. If not exists: insert order, publish OrderCreated, ack message
   *   5. Catch duplicate key error (code 11000): query existing order, publish OrderCreated, ack
   *   6. On validation error: log, nack (no requeue), move to next message
   *   7. On database error: log, nack (requeue), let RabbitMQ redeliver
   * Runs forever until process exits.
   */
}
```

---

## Algorithm Notes

### Idempotency: Duplicate Message Delivery

**Goal:** A repeated `OrderRequested` message does not create two orders.

**Key:** `payment_reference`

**Algorithm:**
1. Receive `OrderRequested` message from RabbitMQ
2. Parse and validate message body with Joi schema
3. Query `orders.findOne({ payment_reference: msg.payment_reference })`
4. If order exists:
   - Log: `Order already exists for payment_reference ${msg.payment_reference}, skipping insert`
   - `order_id = existingOrder._id.toString()`
   - Publish `OrderCreated` with existing `order_id`
   - Ack message
   - Return
5. If not exists:
   - Build order document from message fields (see Data Mapping below)
   - Insert: `const result = await orders.insertOne(orderDoc)`
   - On success: `order_id = result.insertedId.toString()`
   - Publish `OrderCreated` with new `order_id`
   - Ack message
6. On duplicate key error (MongoDB error code 11000):
   - Query `orders.findOne({ payment_reference: msg.payment_reference })` again
   - `order_id = existingOrder._id.toString()`
   - Publish `OrderCreated` with existing `order_id`
   - Ack message
   - Log: `Duplicate key error handled for payment_reference ${msg.payment_reference}`

**Code:**
```javascript
const existingOrder = await getOrdersCollection().findOne({
  payment_reference: msg.payment_reference
});

if (existingOrder) {
  console.log(`Order already exists for payment_reference ${msg.payment_reference}`);
  await publishOrderCreated(msg, existingOrder._id.toString());
  channel.ack(rabbitMsg);
  return;
}

try {
  const result = await getOrdersCollection().insertOne(orderDoc);
  await publishOrderCreated(msg, result.insertedId.toString());
  channel.ack(rabbitMsg);
} catch (err) {
  if (err.code === 11000) {
    // Duplicate key error: another consumer inserted first
    const existing = await getOrdersCollection().findOne({
      payment_reference: msg.payment_reference
    });
    console.log(`Duplicate key error handled for payment_reference ${msg.payment_reference}`);
    await publishOrderCreated(msg, existing._id.toString());
    channel.ack(rabbitMsg);
  } else {
    console.error('Database error:', err);
    channel.nack(rabbitMsg, false, true); // requeue
  }
}
```

---

### Data Mapping: `OrderRequested` → Order Document

**Field Mapping:**
- `msg.payment_reference` → `orderDoc.payment_reference`
- `msg.shopper_id` → `orderDoc.shopper_id` (optional; omit field if `null` or `undefined`)
- `msg.email` → `orderDoc.email` (optional; omit field if `null` or `undefined`)
- `msg.ship_to` → `orderDoc.ship_to` (object, required)
- `msg.lines[]` → `orderDoc.lines[]`:
  - `line.product_id` → `orderLine.product_id`
  - `line.quantity` → `orderLine.quantity`
  - `line.price_snapshot_minor_units` → `orderLine.price_minor_units` (rename field)
- `msg.total_minor_units` → `orderDoc.total_minor_units`
- `new Date()` → `orderDoc.created_at`

**Code:**
```javascript
const orderDoc = {
  payment_reference: msg.payment_reference,
  ship_to: msg.ship_to,
  lines: msg.lines.map(line => ({
    product_id: line.product_id,
    quantity: line.quantity,
    price_minor_units: line.price_snapshot_minor_units  // rename
  })),
  total_minor_units: msg.total_minor_units,
  created_at: new Date()
};

// Add shopper_id or email if present
if (msg.shopper_id) {
  orderDoc.shopper_id = msg.shopper_id;
}
if (msg.email) {
  orderDoc.email = msg.email;
}
```

---

### Publishing `OrderCreated`

**Goal:** After order insertion (or idempotency check), publish `OrderCreated` to RabbitMQ.

**Algorithm:**
1. Build `OrderCreated` payload:
   ```javascript
   const payload = {
     event_type: 'OrderCreated',
     payment_reference: msg.payment_reference,
     checkout_id: msg.checkout_id,
     order_id: order_id,  // string (ObjectId.toString())
     timestamp: new Date().toISOString()
   };
   ```
2. Call `publishMessage(channel, 'ecom.events', 'order.created', payload)`
3. Wait for publisher confirm (handled inside `publishMessage`)
4. On success: message is published
5. On failure: throw error, nack the original `OrderRequested` message (do not ack)

**Error Handling:**
- If `publishMessage()` throws (RabbitMQ down, connection lost):
  - Log the error
  - **Do not ack the `OrderRequested` message**
  - Nack with requeue: `channel.nack(rabbitMsg, false, true)`
  - RabbitMQ will redeliver the message
  - On retry: order already exists (idempotency), skip insert, publish `OrderCreated` again

**Code:**
```javascript
async function publishOrderCreated(msg, order_id) {
  const payload = {
    event_type: 'OrderCreated',
    payment_reference: msg.payment_reference,
    checkout_id: msg.checkout_id,
    order_id: order_id,
    timestamp: new Date().toISOString()
  };

  try {
    await publishMessage(channel, 'ecom.events', 'order.created', payload);
    console.log(`Published OrderCreated for payment_reference ${msg.payment_reference}`);
  } catch (err) {
    console.error('Failed to publish OrderCreated:', err);
    throw err;  // Let caller handle nack
  }
}
```

---

### Message Validation

**Goal:** Reject invalid messages before processing.

**Joi Schema:**
```javascript
const orderRequestedSchema = Joi.object({
  event_type: Joi.string().valid('OrderRequested').required(),
  payment_reference: Joi.string().trim().min(1).required(),
  checkout_id: Joi.string().trim().min(1).required(),
  shopper_id: Joi.string().trim().min(1),
  email: Joi.string().trim().min(1),
  ship_to: Joi.object({
    street: Joi.string().trim().min(1).required(),
    city: Joi.string().trim().min(1).required(),
    postal_code: Joi.string().trim().min(1).required(),
    country: Joi.string().trim().min(1).required()
  }).required(),
  lines: Joi.array().items(Joi.object({
    product_id: Joi.string().trim().min(1).required(),
    quantity: Joi.number().integer().positive().required(),
    price_snapshot_minor_units: Joi.number().integer().min(0).required()
  })).min(1).required(),
  total_minor_units: Joi.number().integer().min(0).required(),
  timestamp: Joi.string().isoDate().required()
});
```

**Algorithm:**
1. Parse message body: `const msg = JSON.parse(rabbitMsg.content.toString())`
2. Validate: `const { error, value } = orderRequestedSchema.validate(msg, { abortEarly: false })`
3. If validation error:
   - Log: `Invalid OrderRequested message: ${error.details[0].message}`
   - Nack (no requeue): `channel.nack(rabbitMsg, false, false)`
   - Message goes to DLQ
   - Return (do not process)
4. If valid: proceed with idempotency check and insert

**Code:**
```javascript
let msg;
try {
  msg = JSON.parse(rabbitMsg.content.toString());
} catch (err) {
  console.error('Failed to parse message:', err);
  channel.nack(rabbitMsg, false, false); // DLQ
  return;
}

const { error, value } = orderRequestedSchema.validate(msg, { abortEarly: false });
if (error) {
  console.error('Invalid OrderRequested message:', error.details[0].message);
  channel.nack(rabbitMsg, false, false); // DLQ
  return;
}

// Proceed with value
```

---

### RabbitMQ Connection Setup

**Goal:** Connect to RabbitMQ, declare exchange and queue, start consumer.

**Algorithm:**
1. Connect: `const connection = await amqp.connect(RABBITMQ_URI)`
2. Create channel: `const channel = await connection.createChannel()`
3. Declare exchange: `await channel.assertExchange('ecom.events', 'topic', { durable: true })`
4. Declare queue: `await channel.assertQueue('order.requested.queue', { durable: true })`
5. Bind queue: `await channel.bindQueue('order.requested.queue', 'ecom.events', 'order.requested')`
6. Set prefetch: `await channel.prefetch(10)` (process up to 10 messages concurrently)
7. Consume: `channel.consume('order.requested.queue', async (msg) => { ... }, { noAck: false })`

**Error Handling:**
- Connection failure: log error, exit process (crash-loop backoff in deployment orchestrator)
- Channel error: log error, close connection, exit process

**Code:**
```javascript
const amqp = require('amqplib');

const RABBITMQ_URI = process.env.RABBITMQ_URI || 'amqp://localhost:5672';
const EXCHANGE_NAME = 'ecom.events';
const QUEUE_NAME = 'order.requested.queue';
const ROUTING_KEY = 'order.requested';

let connection, channel;

async function connectRabbitMQ() {
  try {
    connection = await amqp.connect(RABBITMQ_URI);
    channel = await connection.createChannel();

    await channel.assertExchange(EXCHANGE_NAME, 'topic', { durable: true });
    await channel.assertQueue(QUEUE_NAME, { durable: true });
    await channel.bindQueue(QUEUE_NAME, EXCHANGE_NAME, ROUTING_KEY);
    await channel.prefetch(10);

    console.log('RabbitMQ connected, exchange and queue declared');
    return { connection, channel };
  } catch (err) {
    console.error('Failed to connect to RabbitMQ:', err);
    process.exit(1);
  }
}

async function closeRabbitMQ() {
  try {
    if (channel) await channel.close();
    if (connection) await connection.close();
    console.log('RabbitMQ connection closed');
  } catch (err) {
    console.error('Error closing RabbitMQ:', err);
  }
}

module.exports = {
  connectRabbitMQ,
  setupExchangeAndQueue: async (ch, ex, q, rk) => {
    await ch.assertExchange(ex, 'topic', { durable: true });
    await ch.assertQueue(q, { durable: true });
    await ch.bindQueue(q, ex, rk);
  },
  publishMessage: async (ch, ex, rk, payload) => {
    await ch.publish(ex, rk, Buffer.from(JSON.stringify(payload)), {
      persistent: true
    });
    await ch.waitForConfirms();
  },
  closeRabbitMQ
};
```

---

### Graceful Shutdown

**Goal:** Close RabbitMQ channel and connection on SIGTERM/SIGINT before exiting.

**Algorithm:**
1. Register signal handlers in `order/index.js` after starting the consumer
2. On `SIGTERM` or `SIGINT`:
   - Log: `Received SIGTERM, closing RabbitMQ connection`
   - Close channel: `await channel.close()`
   - Close connection: `await connection.close()`
   - Exit: `process.exit(0)`

**Code (in `order/index.js`):**
```javascript
const { closeRabbitMQ } = require('./rabbitmq');

process.on('SIGTERM', async () => {
  console.log('Received SIGTERM, closing RabbitMQ connection');
  await closeRabbitMQ();
  process.exit(0);
});

process.on('SIGINT', async () => {
  console.log('Received SIGINT, closing RabbitMQ connection');
  await closeRabbitMQ();
  process.exit(0);
});
```

---

## Tests

### Test 1: `test_order_created_from_order_requested()`

**Covers:** AC3 (happy path: order service running, message consumed, order created)

**Code:**
```javascript
const amqp = require('amqplib');
const { MongoClient, ObjectId } = require('mongodb');

const MONGODB_URI = process.env.MONGODB_URI || 'mongodb://localhost:27017';
const RABBITMQ_URI = 'amqp://localhost:5672';

test('order created from OrderRequested message', async () => {
  const mongoClient = new MongoClient(MONGODB_URI);
  await mongoClient.connect();
  const ordersCollection = mongoClient.db('order_db').collection('orders');
  await ordersCollection.deleteMany({});

  const msg = {
    event_type: 'OrderRequested',
    payment_reference: 'pay_test_1',
    checkout_id: 'chk_test_1',
    email: 'guest@example.com',
    ship_to: {
      street: '123 Main St',
      city: 'Springfield',
      postal_code: '12345',
      country: 'US'
    },
    lines: [
      {
        product_id: 'prod_001',
        quantity: 2,
        price_snapshot_minor_units: 1500
      }
    ],
    total_minor_units: 3000,
    timestamp: new Date().toISOString()
  };

  const connection = await amqp.connect(RABBITMQ_URI);
  const channel = await connection.createChannel();
  channel.sendToQueue('order.requested.queue', Buffer.from(JSON.stringify(msg)), { persistent: true });
  await channel.close();
  await connection.close();

  await new Promise(resolve => setTimeout(resolve, 2000));

  const order = await ordersCollection.findOne({ payment_reference: 'pay_test_1' });
  expect(order).not.toBeNull();
  expect(order.payment_reference).toBe('pay_test_1');
  expect(order.email).toBe('guest@example.com');
  expect(order.ship_to.street).toBe('123 Main St');
  expect(order.lines).toHaveLength(1);
  expect(order.lines[0].product_id).toBe('prod_001');
  expect(order.lines[0].quantity).toBe(2);
  expect(order.lines[0].price_minor_units).toBe(1500);
  expect(order.total_minor_units).toBe(3000);
  expect(order.created_at).toBeInstanceOf(Date);

  await mongoClient.close();
});
```

---

### Test 2: `test_order_created_after_service_recovers()`

**Covers:** AC3 (order service down, message in queue, service starts, order created)

**Code:**
```javascript
test.skip('order created after service recovers (manual test)', async () => {
  const mongoClient = new MongoClient(MONGODB_URI);
  await mongoClient.connect();
  const ordersCollection = mongoClient.db('order_db').collection('orders');
  await ordersCollection.deleteMany({});

  const msg = {
    event_type: 'OrderRequested',
    payment_reference: 'pay_test_2',
    checkout_id: 'chk_test_2',
    email: 'guest@example.com',
    ship_to: {
      street: '123 Main St',
      city: 'Springfield',
      postal_code: '12345',
      country: 'US'
    },
    lines: [
      {
        product_id: 'prod_001',
        quantity: 1,
        price_snapshot_minor_units: 1000
      }
    ],
    total_minor_units: 1000,
    timestamp: new Date().toISOString()
  };

  const connection = await amqp.connect(RABBITMQ_URI);
  const channel = await connection.createChannel();
  channel.sendToQueue('order.requested.queue', Buffer.from(JSON.stringify(msg)), { persistent: true });
  await channel.close();
  await connection.close();

  // Order service is stopped; message is in queue
  let order = await ordersCollection.findOne({ payment_reference: 'pay_test_2' });
  expect(order).toBeNull();

  // Manual: start order service
  console.log('Start order service manually, then press Enter');
  await new Promise(resolve => process.stdin.once('data', resolve));

  await new Promise(resolve => setTimeout(resolve, 3000));

  order = await ordersCollection.findOne({ payment_reference: 'pay_test_2' });
  expect(order).not.toBeNull();
  expect(order.payment_reference).toBe('pay_test_2');
  expect(order.total_minor_units).toBe(1000);

  await mongoClient.close();
});
```

---

### Test 3: `test_duplicate_message_creates_one_order()`

**Covers:** AC4 (duplicate message delivery does not create two orders)

**Code:**
```javascript
test('duplicate message creates one order', async () => {
  const mongoClient = new MongoClient(MONGODB_URI);
  await mongoClient.connect();
  const ordersCollection = mongoClient.db('order_db').collection('orders');
  await ordersCollection.deleteMany({});

  const msg = {
    event_type: 'OrderRequested',
    payment_reference: 'pay_dup',
    checkout_id: 'chk_dup',
    email: 'guest@example.com',
    ship_to: {
      street: '123 Main St',
      city: 'Springfield',
      postal_code: '12345',
      country: 'US'
    },
    lines: [
      {
        product_id: 'prod_001',
        quantity: 1,
        price_snapshot_minor_units: 1000
      }
    ],
    total_minor_units: 1000,
    timestamp: new Date().toISOString()
  };

  const connection = await amqp.connect(RABBITMQ_URI);
  const channel = await connection.createChannel();
  channel.sendToQueue('order.requested.queue', Buffer.from(JSON.stringify(msg)), { persistent: true });
  channel.sendToQueue('order.requested.queue', Buffer.from(JSON.stringify(msg)), { persistent: true });
  await channel.close();
  await connection.close();

  await new Promise(resolve => setTimeout(resolve, 2000));

  const orders = await ordersCollection.find({ payment_reference: 'pay_dup' }).toArray();
  expect(orders).toHaveLength(1);

  await mongoClient.close();
});
```

---

### Test 4: `test_duplicate_key_error_handled()`

**Covers:** AC4 (duplicate key error is caught and handled gracefully)

**Code:**
```javascript
test('duplicate key error handled', async () => {
  const mongoClient = new MongoClient(MONGODB_URI);
  await mongoClient.connect();
  const ordersCollection = mongoClient.db('order_db').collection('orders');
  await ordersCollection.deleteMany({});

  const existingOrderId = new ObjectId('507f1f77bcf86cd799439011');
  await ordersCollection.insertOne({
    _id: existingOrderId,
    payment_reference: 'pay_preexist',
    email: 'existing@example.com',
    ship_to: {
      street: '456 Elm St',
      city: 'Shelbyville',
      postal_code: '67890',
      country: 'US'
    },
    lines: [
      {
        product_id: 'prod_002',
        quantity: 1,
        price_minor_units: 2000
      }
    ],
    total_minor_units: 2000,
    created_at: new Date()
  });

  const msg = {
    event_type: 'OrderRequested',
    payment_reference: 'pay_preexist',
    checkout_id: 'chk_preexist',
    email: 'guest@example.com',
    ship_to: {
      street: '123 Main St',
      city: 'Springfield',
      postal_code: '12345',
      country: 'US'
    },
    lines: [
      {
        product_id: 'prod_001',
        quantity: 1,
        price_snapshot_minor_units: 1000
      }
    ],
    total_minor_units: 1000,
    timestamp: new Date().toISOString()
  };

  const connection = await amqp.connect(RABBITMQ_URI);
  const channel = await connection.createChannel();
  channel.sendToQueue('order.requested.queue', Buffer.from(JSON.stringify(msg)), { persistent: true });
  await channel.close();
  await connection.close();

  await new Promise(resolve => setTimeout(resolve, 2000));

  const orders = await ordersCollection.find({ payment_reference: 'pay_preexist' }).toArray();
  expect(orders).toHaveLength(1);
  expect(orders[0]._id.toString()).toBe(existingOrderId.toString());
  expect(orders[0].email).toBe('existing@example.com'); // Not overwritten

  // Verify OrderCreated was published with existing order id
  const conn = await amqp.connect(RABBITMQ_URI);
  const ch = await conn.createChannel();
  await ch.assertQueue('order.created.queue', { durable: true });
  const result = await ch.get('order.created.queue', { noAck: true });
  expect(result).not.toBe(false);
  const orderCreated = JSON.parse(result.content.toString());
  expect(orderCreated.order_id).toBe(existingOrderId.toString());
  await ch.close();
  await conn.close();

  await mongoClient.close();
});
```

---

### Test 5: `test_order_created_published_after_insert()`

**Covers:** AC3 (OrderCreated message is published after order insertion)

**Code:**
```javascript
test('OrderCreated published after insert', async () => {
  const mongoClient = new MongoClient(MONGODB_URI);
  await mongoClient.connect();
  const ordersCollection = mongoClient.db('order_db').collection('orders');
  await ordersCollection.deleteMany({});

  // Clear order.created.queue
  const conn = await amqp.connect(RABBITMQ_URI);
  let ch = await conn.createChannel();
  await ch.assertQueue('order.created.queue', { durable: true });
  await ch.purgeQueue('order.created.queue');
  await ch.close();

  const msg = {
    event_type: 'OrderRequested',
    payment_reference: 'pay_publish',
    checkout_id: 'chk_publish',
    email: 'guest@example.com',
    ship_to: {
      street: '123 Main St',
      city: 'Springfield',
      postal_code: '12345',
      country: 'US'
    },
    lines: [
      {
        product_id: 'prod_001',
        quantity: 1,
        price_snapshot_minor_units: 1000
      }
    ],
    total_minor_units: 1000,
    timestamp: new Date().toISOString()
  };

  ch = await conn.createChannel();
  ch.sendToQueue('order.requested.queue', Buffer.from(JSON.stringify(msg)), { persistent: true });
  await ch.close();

  await new Promise(resolve => setTimeout(resolve, 2000));

  ch = await conn.createChannel();
  const result = await ch.get('order.created.queue', { noAck: true });
  expect(result).not.toBe(false);
  const orderCreated = JSON.parse(result.content.toString());
  expect(orderCreated.event_type).toBe('OrderCreated');
  expect(orderCreated.payment_reference).toBe('pay_publish');
  expect(orderCreated.checkout_id).toBe('chk_publish');
  expect(orderCreated.order_id).toMatch(/^[0-9a-f]{24}$/);
  expect(orderCreated.timestamp).toBeTruthy();

  const order = await ordersCollection.findOne({ _id: new ObjectId(orderCreated.order_id) });
  expect(order).not.toBeNull();
  expect(order.payment_reference).toBe('pay_publish');

  await ch.close();
  await conn.close();
  await mongoClient.close();
});
```

---

### Test 6: `test_database_isolation_order_side()`

**Covers:** AC8 (order service does not access other databases)

**Code:**
```javascript
test('database isolation (order side)', async () => {
  const fs = require('fs');
  const path = require('path');

  const otherDatabases = ['checkout_db', 'payment_db', 'cart_db', 'identity_db', 'catalog_db'];
  const orderFiles = fs.readdirSync(path.join(__dirname, '..')).filter(f => f.endsWith('.js'));

  for (const file of orderFiles) {
    const source = fs.readFileSync(path.join(__dirname, '..', file), 'utf-8');
    for (const dbName of otherDatabases) {
      expect(source).not.toContain(dbName);
    }
  }

  // Also assert order service only uses order_db
  const dbSource = fs.readFileSync(path.join(__dirname, '..', 'db.js'), 'utf-8');
  expect(dbSource).toContain('order_db');
  expect(dbSource).toContain("collection('orders')");
});
```

---

**End of LLD.**
