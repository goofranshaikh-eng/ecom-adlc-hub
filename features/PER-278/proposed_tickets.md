# Proposed tickets

## 1
- title: Checkout outbox and OrderRequested publish via RabbitMQ
- repo: app
- ac: AC2, AC4 (checkout side), AC8

## 2
- title: Order service consumes OrderRequested and publishes OrderCreated
- repo: app
- ac: AC3, AC4 (order side), AC8

## 3
- title: Checkout empties the cart on OrderCreated with retry
- repo: app
- ac: AC5, AC8

## 4
- title: Refund a charge when the order never appears within the completion window
- repo: app
- ac: AC6, AC8

## 5
- title: Checkout status endpoint returns accepted, order created, cart cleared, or refunded with order id when present
- repo: app
- ac: AC7

## 6
- title: Integration test suite for distributed checkout acceptance criteria
- repo: app
- ac: AC1, AC2, AC3, AC4, AC5, AC6, AC7, AC8
