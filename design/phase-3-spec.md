# Phase 3 spec — client-level subscription (Option A)

**Goal:** the recurring plan fee becomes the **client's** subscription, not a bot's. One subscription per client; the first agent establishes it; additional agents ride it (no second subscription). This formalises what Part A's `billingReady` heuristic stands in for today.

**Money-sensitive.** Get sign-off on the three decisions below before any code. Full architecture reference: the billing map (in-conversation) — this spec is the plan on top of it.

---

## Current state (two decoupled layers)
- **Subscription layer = per-BOT.** `create-session/route.ts` builds a Stripe subscription keyed on `bot_id` (inline `price_data` from the bot's plan; setup fee on the first invoice; fee routing by direct/destination mode). Stored on `bots.stripe_subscription_id/status/cancel_at_period_end/current_period_end`. Webhooks (`recordBotSubscriptionFromSession` / `syncBotSubscription` / `archiveBotForDeletedSubscription`, **duplicated in both webhook route files**) resolve a single bot; `subscription.deleted` archives that bot **and tears down its Retell agent**.
- **Usage layer = per-CLIENT (already correct).** `client_billing_config`, `usage_counters`, `credit_ledger`, wallet, overage engine, month-end auto-pay, prepaid top-ups — all keyed by `client_id`, charged on `clients.stripe_customer_id` on the connected account. Auto-pay charges **usage only, never the base fee** (the base fee is the subscription).
- **No client-level subscription exists.** `clients` has `stripe_customer_id` but no `stripe_subscription_id`.
- **Two Stripe Customers per client today** (the migration hazard): usage customer = `clients.stripe_customer_id` (connected account); subscription customer = minted at checkout via `customer_email` (connected account in direct mode, **platform account in destination mode**).

---

## Target state
- **One subscription per client**, stored on the client (new columns / small table). The first agent's checkout establishes it; a client with an active subscription skips checkout for further agents.
- **`getBotActivationInfo` returns the client's real subscription status** (drop the "client already has ≥1 agent" heuristic from Part A).
- **`subscription.deleted` acts on the whole client** — suspend/stop all its agents, not one bot.
- **Webhook subscription helpers consolidated** into `subscription-access.ts` (one implementation, both endpoints call it) to halve the surface and prevent drift.
- **Usage layer unchanged.** Subscription = base fee; overage/auto-pay/credits = usage. That split stays.
- **Customer convergence is OUT of scope** (see decision 2): the subscription keeps its own customer as today; we don't try to merge it with the usage customer.

---

## Three decisions needed before coding

### 1. Pricing model — LOCKED: base plan + per-agent seat
The client's subscription has two parts:
- **Base plan** (per client): monthly fee + included-minutes pool, and it **includes the first agent**.
- **Per-agent seat fee**: charged for each agent **beyond the first**. A NEW field on `pricing_plans` (`additional_agent_pence`, default 0 = free), so each plan can price agents differently; £0 → additional agents are free (max agency flexibility).

Stripe shape: one subscription per client with **two items** — the base price + a **per-agent quantity line** (quantity = agents − 1, priced at the plan's seat fee). Included minutes stay a shared per-client pool; overage unchanged. Adding a 2nd+ agent **bumps the seat quantity** (prorated) when the fee > 0, or is free when 0 — this **refines Part A's pay-skip** (skip the full checkout, but increment the seat quantity when the agency prices agents).

### 2. Customer-object convergence — LOCKED: keep today's two-customer model (defer convergence)
Converging the plan sub + usage onto one Stripe customer (so the SME has one card) is only cleanly possible for **direct-charge** agencies; destination-mode subs live on the platform account and can't co-locate with the connected-account usage customer. It's an optimisation, not required for Option A. **Recommend keeping today's two-customer model** (no regression) and treating convergence as a later, direct-charge-only follow-up.

### 3. Migration of existing per-bot subscriptions — LOCKED: safe re-point
Verified read-only against **production** (2026-07-20): 46 clients, 11 live bots all with a subscription, 11 clients with a subscription, **0 clients with 2+ subscriptions**. So migration is a trivial **re-point**: for each live bot carrying a sub, copy `stripe_subscription_id/status/cancel_at_period_end/current_period_end` → the new client columns. No Stripe mutation, no consolidation, no teardown risk. (A guard should still handle the 2+-subs case defensively in case one appears before the migration runs.)

---

## Change set (grouped for incremental, verifiable shipping)

**3A — Foundation (invisible, de-risks everything after):**
- Migration: add `clients.stripe_subscription_id`, `stripe_subscription_status`, `cancel_at_period_end`, `current_period_end` (+ index), and the subscription's customer id if needed for the portal.
- Consolidate the three duplicated webhook helpers into `subscription-access.ts`; both endpoints call the shared version.
- Make the helpers **dual-write** (stamp the bot column AND the client column) so nothing changes behaviourally yet, but client-level data starts populating.
- Migration script: re-point existing live subs onto their client (decision 3).

**3B — Checkout + activation key on the client:**
- `create-session`: key on `client_id`; if the client already has an active subscription, **skip** (no second sub); metadata carries `client_id`; idempotency key client-scoped.
- `getBotActivationInfo`: return the **client's** subscription status; drop the agent-existence heuristic.
- Step-6 UI: "Subscribe & activate" vs "Activate" decided by the client subscription (first agent subscribes; later agents skip — now real, not heuristic).

**3C — Cancellation + surfacing:**
- `customer.subscription.deleted` (both endpoints): resolve the **client**; suspend/stop **all** its agents (not one bot); update the client subscription row.
- Surface the client subscription (plan, status, portal link) on the **client billing page**; key `api/billing/portal` on the client.

**3D — Cleanup:**
- Stop reading `bots.stripe_subscription_id` (keep the column for history or drop later); remove the Part A `billingReady` heuristic once 3B is live.

---

## Risks & verification
- **Highest risk:** the `subscription.deleted → teardown` semantics and the dual webhook endpoints. Mitigate by consolidating first (3A), dual-writing, and verifying against **Stripe test mode** (the same harness approach used for the earlier double-charge fixes) plus **real Postgres** for the client-resolution + suspend-all-agents logic.
- **No double-charge:** subscription still owns the base fee; auto-pay/overage/credits stay usage-only and per-client (unchanged). Verify the two layers stay decoupled.
- **Migration:** verify re-point on a seeded client with one live sub; explicitly assert no Retell teardown fires.
- Each sub-part (3A–3D) ships and is verified independently, smallest-blast-radius first.

## Not in scope (later)
- Customer-object convergence (one card for plan + usage) — direct-charge-only, deferred.
- Phase 4 shared-number routing (separate).
