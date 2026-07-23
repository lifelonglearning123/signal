# Phase 3C — cancellation stops the whole client + surface the subscription

Part of the Option-A billing move (client = the account). Follows 3A (mirror columns) + 3B (checkout/activation/seats). **No schema change / no migration.**

## The model (verified 2026-07-20)
- `agents` is the canonical *answering-entity* table. A wizard `bot` at go-live also inserts an `agents` row (`golive/route.ts:201`) and points `bots.agentId` at it; linked external agents and call-sync also create `agents` rows.
- `suspendClient({agencyId, clientId, reason})` (`lib/usage/suspension.ts`) detaches the Retell inbound-agent binding for **every** active `agents` row of the client and stores reversible `suspension_state` on `client_billing_config`. Restored by `restoreClient` (the "Resume service" button → `resumeClientAction`). This is the same teeth enforcement uses for cap/credits/overage.
- So stopping "all a client's agents" = one `suspendClient` call. It covers wizard bots AND linked agents.
- Today `customer.subscription.deleted` → `archiveBotForDeletedSubscription(sub)` in BOTH webhook routes: archives the ONE owning bot (status=archived, retell ids nulled, `teardownRetellBot`) + mirrors `canceled` to the client. It does NOT stop the client's other agents (code comment says "that's 3C").

## Part 1 — cancellation stops the whole client (the teeth)
In both routes' `archiveBotForDeletedSubscription(sub)`, after the per-bot archive:
1. Bot already resolved → `clientId`, `agencyId`.
2. **Gate**: read `clients.stripe_subscription_id` BEFORE the mirror overwrites it. Proceed only if it is null OR equals `sub.id` — i.e. the deleted sub is genuinely the client's current subscription, not a stray older sub while a newer one is active. (Prod has 0 multi-sub clients, but the gate keeps it correct.)
3. Call new shared helper `stopClientForCanceledSubscription({agencyId, clientId})`:
   - `suspendClient({agencyId, clientId, reason: "subscription"})` — reversible detach of all agents' numbers.
   - Fire a `client_suspended` alert (severity error, reason "subscription") so the agency sees it and the billing banner shows.
   - Non-fatal (try/catch + log) — archive + mirror must still complete.
4. Keep the existing mirror call (sets client status `canceled`).

New bits: `"subscription"` added to `suspendClient`'s `reason` union; `SuspendedBanner` copy branch for it; helper lives in a new `lib/billing/subscription-enforcement.ts` (imports suspendClient + dispatchAlert + alert insert, mirroring enforcement.ts's `fireAlert`).

**Decision A — owning bot:** keep its existing HARD archive (Retell agent torn down, needs re-deploy) and SOFT-suspend the rest of the client's agents (recommended, additive, lowest risk). Alternative: uniform soft-suspend (don't hard-archive) — fully reversible but changes established behaviour.

**Decision B — restore on re-subscribe:** v1 = manual "Resume service" only (recommended; the button already reattaches numbers, and a re-subscribe re-deploys the owning bot via go-live). Alternative: auto-`restoreClient` on the next active-sub webhook (more moving parts). Deferred either way if we pick manual.

## Part 2 — surface the subscription on the client Billing tab
`clients/[id]/billing/page.tsx`: add a "Subscription" card near the top (after the SuspendedBanner) reading the client's Phase-3 columns (already on the loaded `client` row) + plan name via `client_billing_config.plan_id → pricing_plans.name`:
- Status badge: active / trialing / past_due / canceled / none.
- Renews / ends on `subscription_current_period_end`; if `subscription_cancel_at_period_end` → "cancels on <date>".
- "Manage subscription" button (new client component `ManageSubscriptionButton`) → POST `/api/billing/portal {client_id}` → redirect to the Stripe portal. Hidden when the client has no subscription.

## Part 3 — portal route keyed on client
`/api/billing/portal`: accept `{client_id}` in addition to `{bot_id}` (back-compat). For client_id: load+authorise the client (agency staff of the client's agency, or a portal member), use `clients.stripe_subscription_id` to find the customer, open the portal on the correct account (direct vs platform). bot_id path unchanged.

## Verification
- tsc + eslint; render the client Billing tab.
- Part 1 teeth reuse the enforcement-proven `suspendClient`/alert path — new trigger + reason only. Verify the gate/branch by inspection + a throwaway-Neon DB harness if the test Neon URL is available (assert: deleted-sub == client sub → suspension_state written + alert row; deleted-sub != client sub → no suspend). No Stripe charge involved.
- No migration.
