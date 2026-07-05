# Signal — All-in-One Voice Platform (Builder + Monitor Merge)

**Status:** ALL FOUR PHASES COMPLETE 2026-07-05. Data MIGRATED into the new Supabase. Remaining: per-agency cutover per `scripts/migrate-from-legacy/CUTOVER.md` (user-driven: provider keys → DNS → webhooks → sync-now).

**Phase 4 (migration) — executed 2026-07-05:**
- `scripts/migrate-from-legacy/migrate.ts` (dry-run default, --apply, --skip-calls; idempotent) + `CUTOVER.md` runbook.
- Applied: 8 monitor agencies (IDs preserved → R2 audio intact), 29 users WITH password hashes + identities (1 deduped by email), 33 clients, 57 agents, 34 profiles, 22 memberships, 2,733 calls; 6 builder agencies merged/inserted (Twilio/GHL tokens re-encrypted; Express Connect ids carried where Standard absent; custom domains carried), 176 builder users (magic-link, no hashes needed; 15 email-matched), 24 memberships, 26 bots, 16 live bots go-live-backfilled (clients+agents rows → monitored + metered).
- Post-apply repair: ALL migrated jsonb columns were double-encoded (string scalars) — unwrapped in place (bots.draft, calls.raw_metadata ×2733, clients.business_hours ×33, call_email_fields, twilio_regulatory); client/agent names re-derived from drafts. Root cause guarded: migrate.ts now runs a jsonb-normalization step before the go-live backfill on every --apply, so delta re-runs are safe.
- Final state: 13 agencies, 206 auth users, 59 profiles, 49 clients, 73 agents, 2,733 calls, 26 bots (16 live+linked). Builder's "macaws" agency merged into the seeded Macaws AI row by slug (custom domain voice-builder.macaws.ai attached).

**Phase 3 (builder port) — implemented:**
- `src/lib/bots/compile/` + `src/lib/bots/prefill/` — compileBot() and AI prefill ported near-verbatim (PDF knowledge uploads dropped: no unpdf dep; txt/md still work).
- `src/lib/bots/deploy.ts` (create-retell-llm → create-agent; teardown), `src/lib/bots/twilio.ts` (search / idempotent buy with regulatory bundle resolution from agencies.twilio_regulatory / full SIP-trunk link with legacy VoiceUrl blanking), `src/lib/bots/access.ts` (per-bot authorisation: agency roles any agency bot, client roles own bots only).
- Wizard UI at /bots/new (intro → 1 basics → 2 voice → 6 phone/checkout/activate; 3/9/10 stubs mirror source) with server-action autosave (`saveBotDraft` — browser-Supabase writes replaced, RLS-safe), server-side draft resume, /bots list page + nav. Draft JSON shape byte-compatible with vb.bots.draft.
- Checkout on the agency's connected account (Standard): direct or destination charges keyed on agencies.use_direct_charges, price from client_price_pence/currency, platform fee from platform_fee_bps, promo codes ported. /api/checkout/return stamps subscription onto the bot.
- Webhooks (both endpoints) handle bot subscriptions: status mirroring + auto Retell teardown & archive on cancellation.
- `/api/bots/golive` — THE handoff-killer: creates clients row + client_membership + agents row, flips bot live, kicks metering (onCreditsChanged). Every wizard bot is monitored + metered from its first call.
- SECURITY improvement over source: deploy compiles the persisted draft server-side (never trusts a browser payload); deploy/buy/link routes take bot_id and derive the agency from the session.
- Deferred from source: drop-off recovery cron + emails, /api/bots/notify-welcome, PDF prefill uploads.

**Phase 2 (metering & risk controls) — implemented:**
- `src/lib/usage/metering.ts` — reconciliation-style metering: every ingest recomputes the (client, UTC-month) usage counter from `calls` and squares the credit ledger against it (missing debits + duration-correction adjustments + balance rematerialisation). Idempotent under webhook/cron concurrency; `credits_metered_from` anchors wallets so enabling credits never retro-debits history. `applyCreditEntry` with invoice-id dedupe notes.
- `src/lib/usage/enforcement.ts` — threshold engine on each snapshot: usage_80 once per period (guarded counter-flag claim), cap action at 100% (alert_only | suspend; transfer=suspend+note until forward plumbing exists), credit_low/credit_exhausted with open-alert dedupe, auto-restore when balance recovers or month rolls over. Hooks: `onCallIngested` (all 3 ingest paths), `onCreditsChanged` (webhooks/config saves), `runUsageMaintenance` (after every agency sync).
- `src/lib/usage/suspension.ts` — Retell numbers detached via `inbound_agents: []` with original bindings stored in suspension_state for verbatim restore; ElevenLabs agents recorded as skipped + surfaced in the alert (follow-up).
- `src/lib/usage/topup.ts` — auto top-up via auto-charging Stripe invoice on the agency's connected account (platform fee, Smart Retries, 1h attempt rate-limit); Connect webhook credits invoice.paid (kind=auto_topup) and checkout.session.completed (kind=pack_purchase), both idempotent.
- UI: risk-controls card + wallet + suspended banner on `/clients/[id]/billing`; minute-packs manager on `/billing`; client-facing `/portal/minutes` (balance, buy packs via Stripe Checkout, history) + portal nav entry.

**Phase 1 notes (implemented):**
- Legacy drizzle migrations squashed to a fresh baseline (old files kept in `drizzle_legacy/` for the Phase 4 data-migration reference). The `CREATE TABLE auth.users` stub was stripped from the generated SQL, matching the legacy convention.
- RLS `0002` also enables RLS on five previously-uncovered tables (monthly_invoices, call_shares, call_bookings, offer_claims, agency_signups) and locks agencies-row credential columns away from browser roles via column-level grants.
- Credentials: AES-256-GCM (`CREDENTIALS_ENCRYPTION_KEY`), resolver in `src/lib/tenancy/credentials.ts` (agency DB creds → platform env fallback, 30s cache). Managed on /settings (super_admin only) with test-connection actions.
- Tenancy: `src/lib/tenancy/resolve.ts` (verified custom domain → {slug}.PLATFORM_ROOT_DOMAIN → DEFAULT_AGENCY_SLUG). `requireSession` returns the resolved agency; owner-email auto-promotion ported from the builder.
- All 7 crons fan out over active agencies (pool of 3; billing-monthly sequential). Webhooks resolve the agency by platform_agent_id before signature verification with that agency's key.
- Email templates take optional per-agency `branding` (default platform) — callers still pass nothing; wire agency branding through in Phase 2/3 polish.

**Known follow-ups (non-blocking):**
- `dispatchAlert` WhatsApp prefix + alert links still use platform branding/APP_URL — switch to agency branding + agencyBaseUrl.
- README.md / ONBOARDING.md / AGENTS.md and scripts/seed-agency.ts still describe the old AGENCY_ID model — rewrite at cutover (Phase 4).
- Supabase Auth redirect allowlist must include agency domains ({slug}.macaws.ai wildcard + custom domains) for magic links / recovery.

**Original plan date:** 2026-07-04.
**Working copies:** `C:\python\Signal\voice-retell-elevenlabs` (base) and `C:\python\Signal\voice builder` (port source).
**Originals untouched:** `C:\python\voice builder` and `C:\python\voice-retell-elevenlabs` keep running in production until per-agency cutover.

## Locked decisions

1. **New Supabase project.** The merged platform runs on a fresh Supabase; data is migrated in, existing production projects are never modified.
2. **Risk controls are per-client, chosen by the agency:** (a) hard monthly minute cap, (b) auto top-up credits (saved card, off-session), (c) manual top-up credits (client buys packs). Alert at **80%**, configurable action at **100%** (alert-only / suspend agent / transfer-to-human).
3. **Agencies bring their own Retell, ElevenLabs, and Twilio accounts** and pay providers directly. One metering layer: agency → client.
4. **Base codebase: voice-monitor (SIGNAL)** — Next.js 16, Drizzle, webhooks/crons/alerts/R2/billing engine. Builder features are ported in.
5. **Single domain-resolved deployment.** One Vercel project for all agencies (builder's model). New agency = DB row + domain. All per-agency credentials move from env vars to encrypted DB columns.

## Why (the problem being fixed)

The builder sells bots at a flat monthly price and never sees a call — no webhook, no duration tracking, no caps. The monitor tracks `duration_sec` per call and can invoice rate × minutes, but only monthly in arrears, and builder bots aren't auto-linked into it. Result: an agency that sells a bot is exposed to unlimited minute usage. No provider cost or margin is captured anywhere. The merge wires every bot into the call-ingestion pipeline at go-live and adds real-time usage accounting with caps, credits, and enforcement.

---

## Target architecture

- **Repo:** evolve `C:\python\Signal\voice-retell-elevenlabs` (rename later if desired). Next.js 16 App Router, Drizzle, Tailwind 4.
- **One Vercel project** for all agencies. Agency resolved per request: `Host` header → `agencies.custom_domain` (verified) → `{slug}.<platform-domain>` wildcard subdomain → `?agency=` dev override. Port `lib/agency/resolve.ts` from the builder; delete the `env.AGENCY_ID` model.
- **New Supabase project**, single `public` schema, Drizzle migrations from day one. Same RLS helper-function pattern as the monitor.
- **R2:** reuse the existing bucket. Keys are `agency/{agencyId}/calls/...` — we **preserve agency UUIDs during migration**, so all historical audio stays reachable with zero object copying.
- **Branding from DB** (`agencies.brand_*`), not `NEXT_PUBLIC_AGENCY_*` env.

### De-AGENCY_ID refactor (the big base-code change)

- `requireSession()` gains the resolved agency (from host) instead of `env.AGENCY_ID`; a profile must exist for (user, resolved agency).
- **Crons iterate all active agencies** (bounded concurrency, per-agency error isolation) instead of syncing one.
- **Webhooks resolve the agency from the payload:** Retell → look up `agents.platform_agent_id`; ElevenLabs already does per-agency secret lookup by `agent_id` — extend the same pattern. Per-agency webhook secrets live in DB.
- Provider clients (`retell.ts`, `elevenlabs.ts`, Twilio) take credentials as arguments resolved from the agency row, never from env.

### Credentials in DB (encrypted)

`agencies` gains: `retell_api_key`, `retell_webhook_secret`, `elevenlabs_api_key` (+ existing `elevenlabs_webhook_*`), `twilio_account_sid`, `twilio_auth_token`, `twilio_regulatory` jsonb, `from_email`/`from_name`, GHL agency-level creds. App-layer AES-GCM encryption (`lib/crypto/credentials.ts`, key in `CREDENTIALS_ENCRYPTION_KEY` env). Agency settings UI with "test connection" validators per provider (borrow the builder's diagnostic scripts).

---

## Data model (merged, key changes only)

Keep the monitor's 15 tables as the spine. Add / merge:

- **`agencies`** — + builder fields: `custom_domain`, `custom_domain_verified`, `brand_logo_url`, `brand_color`, `client_price_pence`/`client_currency`, `owner_email`, `use_direct_charges`; + credential columns above; + platform-subscription fields (both apps have variants — unify).
- **`clients`** — absorbs the builder's `agency_clients` (portal users become `profiles` + `client_memberships`).
- **`bots` (new, ported from `vb.bots`)** — wizard config (`draft` jsonb), `status`, `client_id`, **`agent_id` FK → `agents`**, `phone_e164`, Stripe sub fields, drop-off tracking. On deploy, a `bots` row **automatically creates its `agents` row** (platform `retell`) — this closes the manual "voice monitor handoff" gap forever.
- **`client_billing_config` (new)** — per client: `mode` enum `payg | cap | credits_auto | credits_manual`, `included_minutes`, `monthly_cap_minutes`, overage rate/unit/currency (reuse existing client rate fields), `cap_action` enum `alert_only | suspend | transfer`, `transfer_number`, `auto_topup_threshold_minutes`, `auto_topup_pack_id`.
- **`credit_ledger` (new)** — append-only: `client_id`, `type` (`topup_auto | topup_manual | usage_debit | adjustment | refund`), `seconds_delta`, `amount`, `currency`, `stripe_payment_intent_id`, `call_id`, timestamps. Materialised `clients.credit_balance_seconds` updated in the same transaction.
- **`usage_counters` (new)** — per (client, year, month): `seconds_used`, `calls_count`, `alert80_fired_at`, `cap_hit_at`, `suspended_at`. Unique (client, period). Recomputable from `calls` (self-healing).
- **`credit_packs` (new)** — agency-defined bundles (name, minutes, price, currency) for the client "Buy minutes" UI.
- **`alerts`** — extend `alert_type`: `usage_80`, `usage_cap_reached`, `credit_low`, `credit_exhausted`, `client_suspended`, `topup_failed`.

Roles: keep the monitor's (`super_admin`, `agency_staff`, `client_admin`, `client_user`). Builder mapping: `owner|admin` → `super_admin`, `staff` → `agency_staff`, `agency_clients` → `client_admin`.

---

## Metering & enforcement pipeline (the core fix)

1. **Ingest** (already real-time-ish): Retell `call_ended`/`call_analyzed` webhook + ElevenLabs `post_call_transcription` webhook; hourly cron as reconciliation backstop.
2. **On every ingested/updated call:** transactionally bump `usage_counters.seconds_used`; in credits modes also write a `usage_debit` to the ledger and decrement the balance.
3. **Threshold checks after each bump:**
   - ≥ 80% of cap/included minutes → `usage_80` alert, once per period (agency always; client optional).
   - ≥ 100% → execute the client's `cap_action`.
   - `credits_auto`: balance < threshold → off-session Stripe PaymentIntent on the saved card → success credits the ledger; failure → `topup_failed` alert + retry; balance ≤ 0 → `cap_action`.
   - `credits_manual`: low balance → email client a buy-minutes link; zero → `cap_action`.
4. **Suspension mechanics** (restore is the mirror image, triggered by top-up or new period):
   - **Retell:** repoint the phone number's inbound agent to a per-agency "suspended" agent (polite message, optional transfer) using the existing phone-number repointing code. State recorded on `usage_counters.suspended_at`.
   - **Twilio-level fallback (platform-agnostic):** PATCH the number's voice config away from the SIP trunk to a message/forward — works for ElevenLabs too since the agency's Twilio creds are in DB.
5. **Reconciliation cron:** hourly recompute of the current period from `calls`, healing counters and re-running threshold checks (idempotent — alerts deduped per period).
6. **Overshoot bound:** metering is post-call, so one in-flight call can exceed the cap. Mitigation: per-call `max_call_duration` is already compiled into every builder agent — surface it in the agency UI as a required risk setting (recommended default ~15 min) so worst-case overshoot = one call.

## Billing (Stripe)

- **Base subscription:** port the builder's checkout + webhook handlers wholesale (they support both direct and destination charges, so legacy Express-connected agencies keep working). New agencies onboard via the monitor's Connect Standard flow. `subscription.deleted` still tears down Retell + archives the bot.
- **Top-up packs:** client portal "Buy minutes" → one-off Stripe Checkout on the agency's connected account → webhook credits the ledger.
- **Auto top-up:** SetupIntent captured at subscribe time (or later in portal); off-session PaymentIntents when the threshold trips.
- **PAYG in arrears:** keep the monitor's monthly aggregation + auto-pay invoicing as the `payg` mode for clients the agency trusts.

---

## Phases (recommended order — risk fix ships before wizard port)

**Phase 1 — Foundation (tenancy + schema).** New Supabase; merged Drizzle schema + RLS; domain-resolution tenancy; creds-in-DB + encryption + agency settings UI; crons fan out over all agencies; webhook agency resolution; branding from DB. *Exit: monitor features work multi-agency in ONE deployment against the new DB.*

**Phase 2 — Metering & risk controls.** `usage_counters`, `credit_ledger`, `client_billing_config`, threshold engine, 80% alerts, cap actions incl. suspension/restore, credit packs + buy-minutes portal page, auto top-up, PAYG mode wiring. *Exit: an agency can cap or credit-gate any client; runaway minutes become impossible beyond one call.*

**Phase 3 — Builder port.** Wizard (4 steps), prefill, `compileBot`, Retell deploy, Twilio search/buy/link (agency creds), checkout + Stripe webhooks, drop-off recovery cron, coupons. Deploy auto-creates the `agents` row → instant monitoring + metering. *Exit: full build → sell → monitor → bill loop in one app.*

**Phase 4 — Migration & cutover.** Scripts in `scripts/migrate-from-legacy/`: agencies (union of both DBs, matched by slug/name, **preserving monitor UUIDs** for R2), auth users (pg_dump `auth` schema from both projects; dedupe by email, monitor wins; remap builder user IDs), profiles/clients/memberships, bots → agents links, calls history, Stripe IDs, offer_claims/invoices. Per-agency cutover runbook: point domain at new Vercel project → repoint Retell/ElevenLabs webhooks → final delta call sync (25h lookback covers it) → mark agency migrated in old deployments' decommission checklist.

## Open items (decide during build, none blocking)

- Platform wildcard domain for `{slug}.` subdomains (monitor tenants already use `*.macaws.ai`).
- Numbers previously auto-bought on the platform Twilio account: leave in place (builder already supports per-agency Twilio override); new purchases use the agency's own account.
- Whether Chao needs a cross-agency platform-admin view in v1 (current model: `chao+alias@` per agency).
- Legacy Express Stripe Connect agencies: keep both charge paths (ported) or re-onboard to Standard at cutover.
- Credential encryption key rotation story.
