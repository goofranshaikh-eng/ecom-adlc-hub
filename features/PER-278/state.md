---
schema_version: 1
key: PER-278
status: in_development
current_node: dev
track: quick
involved_repos:
- app
depends_on: []
owner: goos
lease_until: '2026-11-19T12:29:41.115398+00:00'
revision: 89
hub_sha: null
ticket_hash: 2c9a634265fa1277
jira_hash: 2c9a634265fa1277
tpm_approved: true
specs_locked: true
skills_lock:
  bmad: 1cbcfa272fe65787c06a1fa164a901f46117cca7
stories:
- 01-app
- 02-app
- 03-app
- 04-app
- 05-app
- 06-app
child_ticket_keys:
- PER-286
- PER-287
- PER-288
- PER-289
- PER-290
- PER-291
critic_iterations: 2
dev_retries: 0
pr_revision_rounds: 0
gate_rounds: {}
pr_ids:
  hub_plan: '2'
  hub_arch: '3'
  hub: '3'
waiting_for: null
tokens_in: 3946571
tokens_out: 202199
cost_anthropic_usd: 14.872362
cursor_tokens_in: 602424
cursor_tokens_out: 136166
cursor_cache_read_tokens: 4887168
cursor_cache_write_tokens: 0
history:
- node: intake
  note: 'repo graph: app: Gortex skipped (Gortex not found. Install it (https://github.com/zzet/gortex/releases)
    to ~/.adlc/bin/gortex or on PATH, or set ADLC_GORTEX_BIN)'
  at: '2026-11-17T10:26:48.079311+00:00'
- node: triage
  note: repo graph picked app
  at: '2026-11-17T10:26:54.828505+00:00'
- node: intake
  note: snapshot written
  at: '2026-11-17T10:26:54.830078+00:00'
- node: triage
  note: llm classifier; repos from repo graph
  at: '2026-11-17T10:26:55.750463+00:00'
- node: planner
  note: v1 source=llm loops=['feature_with_architecture', 'delivery', 'pr_fix_minor',
    'pr_fix_moderate'] steps=['analyst', 'architect', 'scrum_master', 'designer',
    'test_architect', 'critic'] skipped=['tpm_gate']
  at: '2026-11-17T10:27:14.913189+00:00'
- node: plan_review
  note: awaiting human hub PR approval (plan)
  at: '2026-11-17T10:27:23.706776+00:00'
- node: plan_review
  note: approved via hub PR (reconcile)
  at: '2026-11-17T10:30:01.833906+00:00'
- node: plan_review
  note: "forge unlocked plan_review \u2192 resume"
  at: '2026-11-17T10:30:01.851121+00:00'
- node: plan_review
  note: 'repo graph: app: Gortex skipped (Gortex not found. Install it (https://github.com/zzet/gortex/releases)
    to ~/.adlc/bin/gortex or on PATH, or set ADLC_GORTEX_BIN)'
  at: '2026-11-17T10:30:04.279168+00:00'
- node: analyst
  note: wrote requirements.md
  at: '2026-11-17T10:31:01.752287+00:00'
- node: analyst
  note: proposed_tickets=6
  at: '2026-11-17T10:31:16.057203+00:00'
- node: architect
  note: wrote hld.md
  at: '2026-11-17T10:34:26.725462+00:00'
- node: architect
  note: specs=['1', '2', '3', '4', '5', '6']
  at: '2026-11-17T10:42:59.042322+00:00'
- node: scrum_master
  note: stories=['01-app', '02-app', '03-app', '04-app', '05-app', '06-app']
  at: '2026-11-17T10:46:01.160936+00:00'
- node: designer
  note: lld updated
  at: '2026-11-17T10:58:55.042304+00:00'
- node: test_architect
  note: tests appended
  at: '2026-11-17T11:11:19.184051+00:00'
- node: merge_pack
  note: merged 6 ticket(s)
  at: '2026-11-17T11:11:19.232241+00:00'
- node: critic
  note: 'screened: C6 note'
  at: '2026-11-17T11:12:10.428249+00:00'
- node: critic
  note: "quote check: C2 breakdown.md \u2192 specs/1.md, stories/01-app.md; C3 outdated;\
    \ C5 outdated; C7 breakdown.md \u2192 specs/1.md"
  at: '2026-11-17T11:12:10.474683+00:00'
- node: critic
  note: 'for the gate, not a spec rewrite: C6'
  at: '2026-11-17T11:12:10.514695+00:00'
- node: critic
  note: 'decided C4: Payment service does not accept an idempotency key in v1. Checkout
    will query its own database by payment_reference after payment accepts to detect
    duplicate submissions, rather than sending a stable key to payment.'
  at: '2026-11-17T11:12:32.506419+00:00'
- node: critic
  note: 'fix refused specs/1.md: would drop sections: [atomic_write] insert checkout
    + outbox in one write (verify mongodb transaction support), after payment accepts:'
  at: '2026-11-17T11:14:12.795875+00:00'
- node: critic
  note: repaired ['hld.md', 'stories/01-app.md'] iter=1
  at: '2026-11-17T11:14:40.817372+00:00'
- node: critic
  note: 'screened: C3 new outside the changed sections; C5 new outside the changed
    sections; C6 note'
  at: '2026-11-17T11:15:10.842166+00:00'
- node: critic
  note: 'for the gate, not a spec rewrite: C3, C5, C6, C6'
  at: '2026-11-17T11:15:10.898456+00:00'
- node: critic
  note: 'fix refused specs/1.md: section `### `checkout/routes.py`` is not in the
    file exactly once'
  at: '2026-11-17T11:15:34.076811+00:00'
- node: critic
  note: repaired ['specs/2.md'] iter=2
  at: '2026-11-17T11:16:32.939680+00:00'
- node: critic
  note: 'screened: C2 note; C7 new outside the changed sections'
  at: '2026-11-17T11:17:16.136787+00:00'
- node: critic
  note: fail after rewrite
  at: '2026-11-17T11:17:16.170465+00:00'
- node: critic
  note: 'open items go to the architect gate PR: C4'
  at: '2026-11-17T11:17:16.179368+00:00'
- node: architect_gate
  note: 'critic items on the gate PR: C4'
  at: '2026-11-17T11:17:33.688595+00:00'
- node: architect_gate
  note: awaiting human hub PR approval
  at: '2026-11-17T11:17:33.688692+00:00'
- node: architect_gate
  note: "reviewer answered C4 \u2192 fix"
  at: '2026-11-17T11:42:47.144873+00:00'
- node: critic
  note: 'for the gate, not a spec rewrite: C2, C3, C5, C6, C6, C7'
  at: '2026-11-17T11:42:47.324512+00:00'
- node: critic
  note: 'decided C4: Checkout implements idempotency by querying its own database
    before calling payment. Before calling `POST /charge`, checkout queries the `checkouts`
    collection for any checkout with the same `cart_id` and status `accepted`, `order_created`,
    or `cart_cleared`. If one exists, checkout returns that existing checkout to the
    client and does not call payment. Otherwise, checkout calls payment once, receives
    `payment_data["reference"]`, then inserts the checkout record and outbox row atomically,
    using that reference as the idempotency key. Payment generates a fresh reference
    on every call and does not accept or honor an idempotency key in v1.'
  at: '2026-11-17T11:43:08.463354+00:00'
- node: critic
  note: 'decided C4: Idempotency is checked by `cart_id` before calling payment. If
    a checkout with the same `cart_id` already exists in status `accepted`, `order_created`,
    or `cart_cleared`, return it without calling payment. Otherwise, call payment
    once and insert the checkout and outbox row using the payment response''s `reference`
    as `payment_reference`. The "before payment call" and "after payment call" blocks
    are removed and replaced with one rule in the "Idempotency: Duplicate Checkout
    Submission" section.'
  at: '2026-11-17T11:44:59.975009+00:00'
- node: critic
  note: 'fix refused specs/1.md: would drop sections: [atomic_write] insert checkout
    + outbox in one write (verify mongodb transaction support), after payment accepts:'
  at: '2026-11-17T11:44:59.976077+00:00'
- node: architect_gate
  note: 'critic items on the gate PR: C2'
  at: '2026-11-17T11:46:04.915529+00:00'
- node: architect_gate
  note: 'critic items on the gate PR: C2'
  at: '2026-11-17T11:47:21.194399+00:00'
- node: architect_gate
  note: "reviewer answered C2 \u2192 fix"
  at: '2026-11-17T11:58:59.371136+00:00'
- node: critic
  note: "quote check: C2 specs/1.md \u2192 stories/01-app.md"
  at: '2026-11-17T11:58:59.612385+00:00'
- node: critic
  note: 'for the gate, not a spec rewrite: C3, C4, C5, C6, C6'
  at: '2026-11-17T11:58:59.618405+00:00'
- node: critic
  note: 'decided C2: The idempotency check is before calling payment, using `cart_id`
    as the key. Checkout queries for an existing checkout with the same `cart_id`
    and status in `["accepted", "order_created", "cart_cleared"]`. If found, it returns
    that checkout without calling payment. Otherwise, it creates a pending checkout,
    generates a new `payment_reference`, calls payment with that reference, and then
    inserts the outbox and updates the checkout to accepted. Payment now accepts `payment_reference`
    in the charge request and returns the same reference. The first call with a reference
    creates one charge; a later call with the same reference returns that charge without
    creating another. This change appears in checkout/routes.py (before-payment idempotency
    check, pending checkout insert, payment call with reference), checkout/models.py
    (pending and refused statuses), payment/models.py and payment/routes.py (payment_reference
    field), and the tests (payment call count assertion updated to 1, payment test
    added for duplicate reference).'
  at: '2026-11-17T12:01:51.682326+00:00'
- node: architect_gate
  note: approved via hub PR (reconcile)
  at: '2026-11-17T12:29:41.015845+00:00'
- node: architect_gate
  note: "forge unlocked architect_review \u2192 resume"
  at: '2026-11-17T12:29:41.115622+00:00'
- node: architect_gate
  note: 'repo graph: app: Gortex skipped (Gortex not found. Install it (https://github.com/zzet/gortex/releases)
    to ~/.adlc/bin/gortex or on PATH, or set ADLC_GORTEX_BIN)'
  at: '2026-11-17T12:29:43.122895+00:00'
- node: dev
  note: 'sub-issues created from the approved breakdown: PER-286, PER-287, PER-288,
    PER-289, PER-290, PER-291'
  at: '2026-11-17T12:29:58.317922+00:00'
- node: dev
  note: starting
  at: '2026-11-17T12:29:58.317965+00:00'
- node: dev
  note: 'not checked before PR, commands empty in project.yaml: repos.app.commands.test,
    repos.app.commands.lint'
  at: '2026-11-17T12:29:58.318914+00:00'
- node: dev
  note: done 01-app
  at: '2026-11-17T12:44:16.980201+00:00'
- node: dev
  note: done 02-app
  at: '2026-11-17T12:54:51.954705+00:00'
- node: dev
  note: done 03-app
  at: '2026-11-17T13:05:05.635041+00:00'
---

# Feature state

See frontmatter for machine-readable fields.
