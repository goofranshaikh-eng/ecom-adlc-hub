# Plan — PER-278 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Checkout becomes async and resilient through an outbox, RabbitMQ, and event-driven order creation; this is a contract change across services that needs high-level design.

## Steps

1. [x] **analyst (requirements)** — Turn the detailed ticket into formal requirements with Given/When/Then acceptance criteria.
   - focus: Keep all eight acceptance criteria; emphasize the distributed transaction boundaries, idempotency, and the completion window for refunds.
2. [x] **architect (hld)** — New async contract between checkout, order, and payment via RabbitMQ; outbox pattern; schema for OrderRequested, OrderCreated, RefundRequested messages.
   - focus: Define the message schemas, the outbox table, the completion window, and the RabbitMQ topology. Address idempotency at checkout and order. Keep HTTP for cart and payment where the ticket says so.
3. [x] **scrum_master (breakdown)** — The work splits into at least checkout outbox+publish, order consume+create, cart delete retry, refund logic, and status endpoint.
   - focus: Break down by service slice: checkout database and outbox, relay, order consumer, cart retry, payment refund, checkout status API.
4. [x] **designer (lld)** — Each story needs concrete file-by-file plans for the outbox schema, RabbitMQ setup, message handlers, and retry logic.
   - focus: Show the outbox table DDL, the RabbitMQ exchange and queue config, the publish relay, and the idempotency checks in checkout and order.
5. [x] **test_architect (tests)** — Eight acceptance criteria with edge cases: payment refused, order service down, duplicate checkout, cart delete retry, refund, status fetch.
   - focus: Map each AC to test cases; include timeout scenarios for the completion window and duplicate message delivery.
6. [x] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [x] **critic (consistency_check)** — Requirements, HLD, breakdown, stories, and tests must all agree on the message schemas, idempotency keys, and service boundaries.
8. [x] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [x] **dev** _(auto)_ — Loop delivery.
10. [x] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [x] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — The ticket states scope and acceptance criteria clearly; no ambiguity or cross-team sign-off needed.
