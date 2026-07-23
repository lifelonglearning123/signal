# Phase 2 spec — Client-as-parent, tabbed pages, "Add bot reuses the client"

**Goal:** make a client the account and its AI agents the children *in the UI and the creation flow* — so an agency can add a second agent to an existing client without minting a duplicate. Convert the client and agent pages to tabbed layouts.

**Decisions locked 2026-07-19:** user-facing term is **"AI Agents"** (routes stay `/agents`). The top-level cross-client list is **kept and renamed Bots → AI Agents** (stays for big agencies; also shown as a tab inside each client). Client tabs = **Overview · AI Agents · Calls · Billing · Settings** with Team/Integrations/Details inside Settings. Open: interim billing for an agency-added 2nd agent (decision 4).

**Depends on:** Phase 1 (shipped, commit 987d373). **Out of scope (later phases):** the per-client billing consolidation (Phase 3) and shared-number routing (Phase 4). Phase 2 does **not** change how money is charged.

---

## Current state (grounded)

- **Data model is ready.** `bots.client_id` is **nullable and settable** (`schema.ts:1147`, `onDelete: set null`). `agents.client_id` is required with no unique index (`schema.ts:343`) → a client already can own many agents. **No migration needed for Phase 2.**
- **The nesting already exists**, just not surfaced well:
  - `clients/[id]/page.tsx` — a **hub of cards** (AI agents, Team, Integrations, Billing, Details) + a portal-audio toggle + delete.
  - `clients/[id]/agents` — the client's bot list · `clients/[id]/agents/[agentId]` — the **bot editor** (renders `AgentSettingsForm` / `ElevenLabsAgentSettingsForm` — one big form: voice, prompt, transfer, calendar…).
  - `clients/[id]/{team,integrations,billing,details}` — sub-pages.
- **`/bots`** (`(app)/bots/page.tsx`) — a separate **flat list of all bots** across clients (the top-nav "Bots").
- **Wizard** (`bots/new/page.tsx`, own layout + auth gate) — accepts `?botId=` to resume/edit; a NEW bot calls `saveBotDraft` (`lib/bots/actions.ts:66`) which inserts a bot with **`clientId` unset**.
- **Go-live** (`api/bots/golive/route.ts:111`) — `if (!bot.clientId)` **auto-creates a fresh client**. It already **reuses `bot.clientId` when set** — so setting it upstream is the entire fix.
- **Calls** carry both `clientId` and `agentId` (`schema.ts`), so per-client and per-bot call views are just queries.

---

## Change set

### A. "Add bot reuses the client" — the bug fix (highest value, smallest change)
1. **`saveBotDraft(botId, draft, clientId?)`** (`lib/bots/actions.ts`) — accept an optional `clientId`; on the *insert* branch include it, after validating the client belongs to `session.agencyId`. Existing calls (no clientId) are unchanged.
2. **Wizard entry** (`bots/new/page.tsx` + `entry-client.tsx` + `lib/bots/wizard/context`) — read `?clientId=` from the URL and thread it into the provider so the **first** `saveBotDraft` carries it. (This is the fiddly bit — the wizard's first save happens client-side via `WizardProvider`.)
3. **"＋ New AI agent" button** on the client's AI Agents tab → `/bots/new?clientId=${client.id}`.
4. **Go-live** — no change; `bot.clientId` is now set, so it reuses the client instead of creating one.
5. **Guard** — when `clientId` is present, the wizard's `business_name` is cosmetic (go-live's client-create branch is skipped). No other change.

> Result: an agency (or SME) adding a bot to an existing client stops producing duplicate client records. **Billing note:** until Phase 3, a second bot still runs the wizard pay step and creates its own Stripe subscription; usage metering is already per-client, so there's no double-charge — just two monthly subs on one client until Phase 3 consolidates them. (See open decision 4.)

### B. Client page → tabs
Add **`clients/[id]/layout.tsx`** rendering a persistent tab bar (client component, `usePathname` for active state) that wraps all `clients/[id]/*` routes. Tabs → routes:

| Tab | Route | Source |
|---|---|---|
| Overview | `/clients/[id]` | repurpose the current hub page into a summary (counts, health, recent calls) |
| AI Agents | `/clients/[id]/agents` | existing list + **＋ New AI agent** button |
| Calls | `/clients/[id]/calls` | **new** — calls filtered by `clientId`, with a per-agent filter |
| Billing | `/clients/[id]/billing` | existing |
| Settings | `/clients/[id]/settings` | **new hub** consolidating Integrations + Details + Team + portal-audio (+ Numbers placeholder for Phase 4) |

Keep the existing `integrations`/`details`/`team` routes as sub-pages the Settings tab links to (or inline them) — minimal churn, no dead links.

### C. AI agent page → tabs
On `clients/[id]/agents/[agentId]/page.tsx`, wrap the editor in a **client-side tab component** (not sub-routes — keep one form + one save) with panels:

| Tab | Contents (from today's `AgentSettingsForm`) |
|---|---|
| Overview | status, phone number, platform, quick stats |
| Voice & script | voice picker, opening line, prompt/instructions |
| Booking & transfer | calendar booking toggle, transfer-to-human target |
| Hours | timezone (inherit-from-client default) + when it answers |

Refactor `AgentSettingsForm` fields into these fieldsets; the tab toggles which fieldset shows. This is the largest UI change in Phase 2. The "Wizard" concept retires into "＋ New AI agent" (create) + this page (edit).

### D. Top-nav cleanup
- **Rename "Bots" → "AI Agents" and keep it** (decision 1): the Manage group stays `Clients · AI Agents`, and the `/bots` route becomes the cross-client "AI Agents" list. It also appears as a tab inside each client.
- **Remove "Alerts"** from Monitor; add an **Alerts tab** to the Calls page (`/calls` gets a tab bar: All calls · Alerts · Bookings), keeping the `/alerts` route reachable/redirected. (Still planned — flag if you'd rather keep Alerts as its own item.)

---

## Sequencing (safest first, each independently shippable)
1. **A — Add-bot-reuses-client** (server action + wizard thread + button). Ships the bug fix without any nav/tab change.
2. **B — Client tabs** (layout + Overview/Calls/Settings). Pure structure; routes already exist.
3. **C — Bot tabs** (form refactor). Biggest UI diff; do it on its own.
4. **D — Nav cleanup + Alerts→Calls tab.** Do last, once B/C give bots and alerts their new homes.

## Risks & mitigations
- **Wizard state threading (A2)** is the trickiest — the provider must carry `clientId` from URL to first save. Mitigation: add `clientId` to `InitialBot`/context; unit-test that `/bots/new?clientId=X` produces a bot row with `client_id = X`.
- **Bot-form refactor (C)** could regress saving. Mitigation: keep it one form + one submit; tabs only toggle visibility; verify each control still round-trips.
- **Don't strand routes (D).** Keep `/alerts` and `/bots` resolvable (redirect or "All bots" view) so old links/bookmarks don't 404.

## Verification
- **A:** `/bots/new?clientId=X` → complete wizard → go-live → assert exactly **one** client (the existing X), one new bot with `client_id=X`, no duplicate client. Plus the existing "new client" path still works (no clientId → creates one).
- **B/C/D:** tsc + eslint; click-through of Client tabs and Bot tabs; confirm Alerts tab shows the same data as `/alerts`; mobile drawer still fine.

## Decisions
1. ✅ **Keep** the cross-client list, renamed **Bots → AI Agents** (top-level + a tab inside each client).
2. ✅ Client tabs **Overview · AI Agents · Calls · Billing · Settings**; Team/Integrations/Details inside Settings.
3. ✅ User-facing term **"AI Agents"** (routes stay `/agents`).
4. ⏳ **Interim billing for an agency-added 2nd agent** — still open:
   - **(a) Interim / lowest-risk:** "＋ New AI agent" runs the existing wizard incl. the pay step → a 2nd Stripe subscription on the same client (no double-charge; usage is already per-client). Zero billing code in Phase 2; Phase 3 later merges the subs.
   - **(b) Cleaner / recommended:** for an agency adding an agent to an already-billed client, **skip the pay step** — the agent deploys and goes live (go-live only needs a deployed Retell agent + number, not a paid sub), and Phase 3 formalises client-level billing. Better UX, adds a "skip pay when the client already pays" branch in the wizard.
