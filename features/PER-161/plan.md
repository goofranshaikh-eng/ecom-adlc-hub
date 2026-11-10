# Plan — PER-161 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Multi-service e-commerce backend with catalog, cart, checkout, payment, order, and identity; requires architecture design for service boundaries, API contracts, and data isolation, plus thorough story breakdown.

## Steps

1. [x] **analyst (requirements)** — Convert the detailed job story into formal requirements with clear Given/When/Then acceptance criteria.
   - focus: The ticket already has well-formed AC; validate and formalize them in requirements.md, preserving the service ownership table and boundaries.
2. [x] **architect (hld)** — Six services with separate databases, API contracts between them, and data isolation rules require high-level design and API specs.
   - focus: Design the inter-service contracts (cart→catalog price check, checkout→cart+payment, order storage), data schemas per service, and the payment test double. Document how services never share databases.
3. [x] **scrum_master (breakdown)** — Work spans six services across Python FastAPI and Node.js, each with its own MongoDB database and Swagger UI.
   - focus: Break down by service: identity, catalog, cart, checkout, payment (all Python/FastAPI), and order (Node.js). Each story delivers one service's endpoints, database, and Swagger UI.
4. [x] **designer (lld)** — Each service needs concrete file plans, FastAPI route definitions, MongoDB schemas, and Swagger configuration before coding.
   - focus: For each service story: specify the directory structure, route handlers, Pydantic models (or Node.js equivalents for order), MongoDB collections, and how to expose /docs.
5. [x] **test_architect (tests)** — Eleven acceptance criteria with edge cases (unsellable products, payment refusal, quantity zero, data isolation) need explicit test cases.
   - focus: Map each AC to integration tests: catalog sellability, cart total calculation, checkout flows (guest and signed-in), payment acceptance/refusal, order retrieval, and database isolation checks.
6. [ ] **critic (consistency_check)** — Requirements, HLD (service contracts), breakdown (six services), LLD (per-service file plans), and tests must all agree on API boundaries and data ownership.
   - focus: Cross-check that no service accesses another's database, API contracts in HLD match the routes in LLD, and test cases cover all eleven AC plus the Swagger requirement.
7. [ ] **merge_pack (pack)** _(auto)_ — Pack per-ticket specs into merged_plan.md for review.
8. [x] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [x] **dev** _(auto)_ — Loop delivery.
10. [x] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [x] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — The ticket already states clear scope, acceptance criteria, boundaries, and what services exist; no product ambiguity to resolve.
