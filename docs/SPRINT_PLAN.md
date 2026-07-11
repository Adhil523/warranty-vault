# WarrantyVault — Build Sprint Plan

> **Audience.** A Claude coding instance building WarrantyVault **with maximum precision**.
> This doc is the execution contract: read it, then build sprint by sprint, in order,
> stopping at each **Definition of Done** gate.
> **Sources of truth:** [`PRODUCT_PLAN.md`](./PRODUCT_PLAN.md) (intent) ·
> [`DEV_PLAN.md`](./DEV_PLAN.md) (architecture, schema, routes) ·
> [`UI_DESIGN.md`](./UI_DESIGN.md) (look & feel, tokens, components).
> **Last updated:** 2026-07-11

---

## 0. How to use this document

- Build **one sprint at a time, in order.** Do not start a sprint until the previous
  sprint's **Definition of Done (DoD)** is green.
- Each task is a checkbox. Implement, verify, check it off. Keep tasks atomic.
- When DEV_PLAN and this doc disagree, **DEV_PLAN wins on architecture**, UI_DESIGN wins on
  visuals, this doc wins on sequencing. If a real conflict blocks you, **stop and surface
  it** — do not guess silently.
- **Do not scope-creep.** Build exactly what the sprint lists. Park good ideas in
  `docs/BACKLOG.md`, don't implement them. PRODUCT_PLAN §12 non-goals are hard walls.

---

## 1. Operating rules (the "maximum precision" contract)

**Non-negotiables — apply to every task:**

1. **Types are law.** TypeScript `strict`. No `any` without a `// reason:` comment.
   `pnpm check` must pass with zero errors before a task is "done".
2. **Lint + format clean.** `pnpm lint` and `pnpm format` pass; no warnings introduced.
3. **Test what has logic.** Every service function with branching (warranty status/date
   math, dedupe, claim transitions, follow-up scheduling, extraction validation, routing,
   free-tier gating) ships with unit tests. UI gets at least a render/smoke test.
   Target: **all new logic covered; `pnpm test` green.**
4. **Migrations only via `pnpm run gen`** (`zenstack generate` → `prisma migrate dev`).
   Commit the generated migration. Never edit an applied migration; add a new one.
   All schema edits go in `schema.zmodel` — `prisma/schema.prisma` is generated output.
5. **Follow the conventions** (DEV_PLAN §1): cuid IDs, `householdID`-style FKs,
   `createdAt`/`updatedAt`, jobs export `registerXWorker()` + `scheduleX()` wired in
   `setup.ts`, routes use `locals.db` (policy-enhanced), jobs/webhooks use
   `dangerousPrismaDoNotUse` with sender-derived scoping.
6. **Security posture.** Household isolation is enforced by ZenStack policies — never by
   hand-rolled `where` filters alone. Webhooks verify signatures and derive identity from
   the sender, never from payload IDs. No secret in the repo; all via env (DEV_PLAN §4).
   Validate every request/webhook body (zod) at the boundary. Storage bucket is private;
   clients only ever get short-lived signed URLs.
7. **Every capture channel funnels through `capture.ts`** (DEV_PLAN §6.1). If a channel
   needs its own item-creation path, the design is wrong — stop and surface it.
8. **Use the design tokens** from `global.css` — never new hex/fonts (UI_DESIGN §2, §9).
   Every component implements the five states (UI_DESIGN §6).
9. **Accessibility is acceptance, not polish.** WCAG 2.2 AA, keyboard path,
   reduced-motion, no color-only status (UI_DESIGN §7).
10. **Commit discipline.** Small, conventional commits, one logical change each. Never
    commit on the default branch — branch per sprint: `sprint/<n>-<slug>`, cascading
    (branch sprint N+1 off sprint N).
11. **Verify, then claim.** "Done" means you ran the verification and saw it pass. If a
    step was skipped or failed, say so explicitly. Never report green on unrun checks.

---

## 2. Global Definition of Done (every sprint)

- [ ] All sprint tasks implemented and checked off.
- [ ] `pnpm check` · `pnpm lint` · `pnpm test` · `pnpm build` all pass.
- [ ] New logic has meaningful tests (assert behavior, not implementation).
- [ ] DB in sync: `prisma migrate status` clean; generated client current.
- [ ] No secrets committed; `.env.example` updated for any new var.
- [ ] Screens meet UI_DESIGN states + a11y; verified by keyboard + reduced-motion check.
- [ ] A short note appended to `docs/BUILD_LOG.md`: what shipped, decisions, follow-ups.
- [ ] Sprint branch ready as a PR; description links the tasks done.

---

## 3. Tech invariants (lock these in Sprint 0, never drift)

| Concern         | Decision (from DEV_PLAN §1)                                        |
| --------------- | ------------------------------------------------------------------ |
| Framework       | SvelteKit 2, adapter-node, TypeScript strict, Svelte 5 runes       |
| Package manager | pnpm                                                                |
| Auth            | better-auth (Prisma adapter) — email, phone OTP, Google/Apple      |
| ORM / DB        | ZenStack over Prisma; PostgreSQL (Supabase)                        |
| Files           | Supabase Storage, private bucket, signed URLs                      |
| Jobs            | pg-boss; `register*`+`schedule*`; init in `hooks.server.ts`        |
| AI              | Claude API — extraction (vision) + claim drafting; models via env  |
| Email           | Postmark (inbound webhooks + outbound transactional)               |
| Push            | FCM (Firebase Admin); email fallback                               |
| Mobile          | Capacitor shell over the web views                                 |
| Styling         | Tailwind 4 + `global.css` `@theme` tokens                          |
| Validation      | zod at every route and webhook boundary                            |
| Testing         | Vitest (unit/integration) + Playwright (key flows, later sprints)  |

---

## 4. Sprints

> Maps to PRODUCT_PLAN §13 (Suggested Build Order) and DEV_PLAN §12. Each sprint lists
> **Goal · Depends on · Tasks · Acceptance · Verify.**

### Sprint 0 — Project foundation & guardrails

**Goal:** A typechecking, linting, testing SvelteKit skeleton with CI-able scripts.
**Depends on:** nothing.

- [ ] Scaffold SvelteKit 2 + adapter-node + TS strict; Tailwind 4 wired to
      `src/styles/global.css`.
- [ ] Author `global.css` `@theme`: brand primitives + the **semantic token alias layer**
      (UI_DESIGN §2) — components consume only semantic tokens.
- [ ] pnpm scripts: `dev`, `build`, `check`, `lint`, `format`, `test`, `gen`, `seed`.
- [ ] Vitest + testing-library; one trivial passing test. ESLint + Prettier; Husky
      pre-commit runs `lint`+`check`.
- [ ] `.env.example` with every var from DEV_PLAN §4 (no real values).
- [ ] `docs/BUILD_LOG.md` + `docs/BACKLOG.md` initialized.
- **Acceptance:** `pnpm check && pnpm lint && pnpm test && pnpm build` green on a fresh
  clone; fonts + token classes render on a demo page.
- **Verify:** run all four scripts; open `/` and confirm tokens/fonts.

### Sprint 1 — Auth + identity + personal household

**Goal:** Users sign up/in; every user gets a personal Household; session context is
available server-side. **Depends on:** Sprint 0. **(DEV_PLAN §3.1)**

- [ ] better-auth (Prisma adapter, Postgres): email+password, phone OTP, Google
      (Apple later; absence degrades). Served in-hook (Perch pattern).
- [ ] `src/lib/server/auth.ts` + `src/lib/auth-client.ts`; `(auth)/sign-in`, `sign-up`
      pages using brand tokens.
- [ ] Signup hook creates the personal `Household` (+ unique `inboundSlug`) and
      `HouseholdMember` row atomically.
- [ ] `hooks.server.ts`: session → `locals.session` → `{ userID, householdID }`;
      `GET /api/me` for client guards.
- **Acceptance:** sign-up → a household exists for the user; session persists across
  reloads; `/api/me` returns `{ userID, householdID }`.
- **Verify:** unit test for household-on-signup; manual sign-in round-trip.

### Sprint 2 — Schema, policies & storage plumbing

**Goal:** Full data model live with household isolation, immutable claim events, and a
working private storage layer. **Depends on:** Sprint 1. **(DEV_PLAN §5, §3.7)**

- [ ] Author `schema.zmodel`: all models from DEV_PLAN §5 with `@@allow`/`@@deny`
      policies. `pnpm run gen`; commit migration.
- [ ] `src/lib/server/db.ts`: enhanced client (`getDb()` from session) +
      `dangerousPrismaDoNotUse`.
- [ ] `src/lib/server/storage.ts`: private bucket upload / signed-URL read / delete;
      sha256 content hashing.
- [ ] Seed: `WarrantyDefault` rows (common categories), a few `BrandChannel` rows (IN),
      demo user + household + 2 items with documents.
- [ ] Policy tests: cross-household reads denied; household member reads allowed;
      **`ClaimEvent` update/delete rejected even for its owner**; `Document` hash unique
      per household.
- **Acceptance:** policies provably isolate households; the claim-event immutability test
  fails any attempted edit; a stored file round-trips via signed URL.
- **Verify:** policy integration tests green; `prisma migrate status` clean.

### Sprint 3 — Capture → extraction → confirmation (the core loop)

**Goal:** A user captures a receipt (photo/upload/manual) and, after AI extraction,
confirms a tracked item — end to end. **Depends on:** Sprint 2.
**(DEV_PLAN §3.2, §6.1–6.3, §7; PRODUCT_PLAN §5.1–5.2)**

- [ ] `capture.ts`: the single funnel (files → storage → hash/dedupe → Item → enqueue).
      Multi-page support. Manual entry creates `ACTIVE` directly when fields complete.
- [ ] pg-boss wired (`setup.ts`, init in `hooks.server.ts`); `extract-document.job` with
      retries; terminal failure → `NEEDS_REVIEW` (raw file preserved).
- [ ] `extraction.ts`: Claude vision call, strict JSON schema, zod-validated;
      `ExtractionRun` logging; `WarrantyDefault` inference labeled `ESTIMATED`.
- [ ] `warranty.ts` pure functions (status, end-date, thresholds) — exhaustive unit tests.
- [ ] `POST /api/capture` (multipart + manual), `GET /api/items/[id]/extraction`,
      `POST /api/items/[id]/confirm`, `GET /api/documents/[id]/url`.
- [ ] UI: `vault/capture` (camera/upload/manual tabs, multi-page), optimistic
      "reading your receipt…" state, `vault/items/[id]/confirm`
      (`ExtractionConfirmCard`: per-field edit, estimated-warranty chip, one-tap confirm).
- **Acceptance:** photo → item appears instantly → extraction fills fields → confirm sets
  `USER_CONFIRMED` warranty + end date; a garbage image lands in `NEEDS_REVIEW` with the
  file intact; the same file captured twice attaches, not duplicates.
- **Verify:** `pnpm test` (warranty math, dedupe, extraction validation with a mocked
  Claude client); one real end-to-end capture against the live API.

### Sprint 4 — The Vault (home + item detail)

**Goal:** The vault list and item detail are complete and pleasant. **Depends on:**
Sprint 3. **(PRODUCT_PLAN §5.3; UI_DESIGN §5.1–5.2)**

- [ ] `GET /api/items` with sort (default soonest-expiring), filters (category, brand,
      price), search (name/brand/seller — PG `ILIKE`/FTS).
- [ ] `vault/` home: `ItemRow` + `WarrantyChip` (Active/Expiring/Expired/Unknown),
      filter/sort/search UI, empty state that points at capture.
- [ ] `vault/items/[id]`: document viewer (signed URLs), extracted data,
      `WarrantyTimelineBar`, actions ("Something broke" placeholder → Sprint 7,
      "Add document", "Edit details"); `PATCH /api/items/[id]`.
- [ ] Onboarding flow shell (PRODUCT_PLAN §8): value promise → camera-first capture →
      account creation after first capture (held-draft handoff).
- **Acceptance:** 20-item seeded vault sorts/filters/searches correctly; chips match
  `warranty.ts` for boundary dates; onboarding reaches a confirmed first item without an
  account existing before capture.
- **Verify:** list-query integration tests (sort/filter/search); manual onboarding run.

### Sprint 5 — Warranty lifecycle notifications

**Goal:** Expiry alerts and digests arrive reliably; prefs are respected.
**Depends on:** Sprints 3–4. **(DEV_PLAN §6.7, §7; PRODUCT_PLAN §5.4)**

- [ ] `notify.ts` + `push.ts` (FCM) + `email.ts` (Postmark) with the layered fallback;
      `Notification` log; `POST /api/devices`; `NotificationPref` + `PATCH
      /api/me/notifications` + settings UI.
- [ ] `expiry-sweep.job` (daily): 30d + 7d alerts with the "claim now" nudge;
      extended-warranty explainer for high-value items (neutral, no upsell).
- [ ] `weekly-digest.job` (opt-in).
- [ ] Notification-permission prompt at first warranty registration (UI, framed per item).
- [ ] Tests: threshold selection (simulated clock), channel fallback, idempotency (a
      re-run sweep doesn't double-send).
- **Acceptance:** an item crossing the 30d boundary produces exactly one alert on the
  right channel; FCM/Postmark vars absent → warning logged, no crash; digest contains
  newly-expiring items only for opted-in users.
- **Verify:** job unit tests with fake clock; one real email delivery.

### Sprint 6 — Email-forwarding capture

**Goal:** Forwarding a purchase email files it automatically. **Depends on:** Sprint 3.
**(DEV_PLAN §3.4; PRODUCT_PLAN §5.1)**

- [ ] Postmark inbound: `POST /api/webhooks/email/[secret]` — verify, resolve household by
      `inboundSlug`, unknown → drop+log; `InboundMessage` idempotency.
- [ ] Parse attachments (PDF/images) → `capture.ts` with `source: EMAIL`; body-only
      purchase emails → render HTML body to a stored document, same pipeline.
- [ ] Surface the user's forwarding address in settings + onboarding prompt.
- [ ] Tests: slug resolution, idempotent redelivery, unknown-sender rejection.
- **Acceptance:** forwarding a real invoice email creates a `NEEDS_CONFIRMATION` item with
  the PDF attached and "via email" source label; replaying the webhook is a no-op.
- **Verify:** webhook integration tests with recorded payloads; one live forward.

### Sprint 7 — Claims engine I: intake, routing, claim file, initial draft

**Goal:** "Something broke" produces a complete claim file, the right channel, and a
ready-to-send message. **Depends on:** Sprints 3–4.
**(DEV_PLAN §6.4–6.6; PRODUCT_PLAN §6.1–6.3)**

- [ ] `claims.ts`: `startClaim` (intake → AI defect summary → Claim + first `ClaimEvent`),
      free-tier gate (one active claim; clean upgrade error).
- [ ] Intake UI (`claims/[id]/intake` via item detail): what's wrong (text; voice-note
      deferred to BACKLOG), when, defect photos (`Document` kind `DEFECT_EVIDENCE`).
- [ ] `routing.ts` + `BrandChannel` KB: resolve → show `ChannelCard`; unknown brand →
      guided search + user-confirm → write back `verified` row.
- [ ] `drafting.ts` + `POST /api/claims/[id]/drafts`: INITIAL stage, tone (polite/firm) +
      language toggles; references invoice no., dates, warranty terms; stored as
      `ClaimDraft`; **never auto-sent** — `DraftComposer` with copy/share actions.
- [ ] Jurisdiction content plug-in scaffold (`content/jurisdictions/in.ts` + types).
- **Acceptance:** from a tracked item: intake → claim file (proof + status + defect +
  evidence) → correct channel shown → INITIAL draft cites real item fields; a second
  simultaneous claim on free tier is blocked server-side.
- **Verify:** claim-creation + gating tests (mocked drafting model); manual flow run.

### Sprint 8 — Claims engine II: tracker, follow-ups, escalation, outcomes

**Goal:** Claims are tracked on a tamper-evident timeline, nudged on schedule, escalated
credibly, and closed with recovered value. **Depends on:** Sprint 7 (+ Sprint 5 for
notifications). **(DEV_PLAN §6.4, §6.6, §6.8, §7; PRODUCT_PLAN §6.3–6.5, §10)**

- [ ] `transitionClaim`: one-tap status updates (Sent/Replied/In service/Resolved/Denied)
      — each writes `Claim` + `ClaimEvent` in one txn; `ClaimTimeline` UI.
- [ ] Follow-up engine: `Sent` sets `nextFollowUpAt`; `claim-followup-sweep.job` nudges
      with a pre-drafted FOLLOW_UP referencing the earlier message/date; user-visible,
      user-adjustable timer.
- [ ] Escalation ladder drafts: denial request → legal notice (India CPA content) →
      consumer-forum complaint with the full evidence timeline; disclaimer on all.
- [ ] Outcome capture (`repaired/replaced/refunded/denied` + value) →
      `RecoveredTotalHero` on vault home once nonzero.
- [ ] `export.ts` + dossier job: single PDF (timeline + documents + drafts);
      `GET /api/claims/[id]/dossier`.
- [ ] Tests: transition legality matrix, follow-up scheduling (fake clock), recovered-sum
      aggregation, event immutability under transitions.
- **Acceptance:** a full lifecycle (start → sent → no reply 7d → follow-up nudge →
  escalate → resolved ₹X) leaves a complete, uneditable timeline; the dossier PDF renders
  every event; the home hero shows ₹X.
- **Verify:** service test suite + one generated dossier inspected by eye.

### Sprint 9 — Household sharing

**Goal:** A spouse joins the vault and both see everything. **Depends on:** Sprint 2
(policies) + Sprint 4 (UI). **(PRODUCT_PLAN §5.5)**

- [ ] Invite flow: `POST /api/household/invites` (code/link, expiring `LinkCode`) →
      `POST /api/household/join` — joining moves the joiner onto the shared household
      (their personal items merge in; decide + document the merge rule); leave flow.
- [ ] `HouseholdPanel` in settings: members, invite, leave; free tier = 2 members
      (gate ready, PRODUCT_PLAN §9).
- [ ] Tests: post-join visibility both ways; departed member loses access; invite expiry.
- **Acceptance:** two real accounts share one vault; either captures, both see it with
  correct attribution; a third member hits the tier gate.
- **Verify:** multi-user policy integration tests; manual two-account run.

### Sprint 10 — Packaging, trust hardening & E2E

**Goal:** Installable app with native camera/push; the trust requirements (§10) are all
real; the core loop is E2E-tested. **Depends on:** Sprints 3–5, 8.
**(DEV_PLAN §3.5, §6.8, §13; PRODUCT_PLAN §10)**

- [ ] Capacitor shell: native camera capture into `CaptureSheet`, push permission +
      `DeviceToken`, deep links (`vault/items/[id]`, `claims/[id]`).
- [ ] Vault export (zip: documents + JSON) + full account deletion (storage + DB) with
      confirmation UX; `POST /api/me/export`, `DELETE /api/me`.
- [ ] Polish pass: five states everywhere, error/retry on all mutations, empty states,
      reduced-motion + keyboard sweep, no console errors.
- [ ] Playwright E2E: capture → confirm → vault; expiring item → alert; claim happy path.
- **Acceptance:** on-device: install → capture with the native camera → receive a real
  push; export zip opens and is complete; deleted account leaves no rows or files;
  E2E suite green.
- **Verify:** Playwright suite + a manual device run.

### Sprint 11 — WhatsApp module (optional, separable)

**Goal:** A linked user WhatsApps a receipt and it appears in their vault; opted-in users
get alerts on WhatsApp. **Depends on:** Sprints 3, 5. **(DEV_PLAN §3.6; PRODUCT_PLAN §7)**

- [ ] Linking: deep-link chat with one-time code + phone-number-entry fallback;
      `WhatsAppLink` (one number → one account); unlink in settings.
- [ ] Webhook (verify handshake, signature check, `InboundMessage` idempotency): media →
      Graph API download → `capture.ts` (`source: WHATSAPP`); confirmation reply with
      extracted summary; text-only quick entry parsing; unlinked sender → friendly
      explainer, **nothing stored**.
- [ ] `wa-group-window.job`: 90s multi-image grouping + the one-question clarifier.
- [ ] Duplicate handling across channels (hash merge + merge-confirming reply).
- [ ] Opt-in notifications via approved templates; `STOP` opt-out; extraction-failure →
      `NEEDS_REVIEW` + "saved, needs a check" reply.
- [ ] Tests: sender resolution, idempotency, grouping window, unlinked rejection.
- **Acceptance:** matches PRODUCT_PLAN §7.2–7.3 behaviors exactly, including duplicates
  and unreadable documents; deleting `src/lib/server/whatsapp/` still builds the core app.
- **Verify:** webhook tests with recorded payloads; live sandbox-number run.

### Sprint 12 — Monetization gates

**Goal:** Free/Pro boundaries enforced and purchasable-ready (no payment processor yet
unless the user says otherwise). **Depends on:** Sprints 8, 9, 11. **(PRODUCT_PLAN §9)**

- [ ] `tier` on user/household + entitlement service; gates: simultaneous claims,
      escalation-grade drafting, >2 household members, WhatsApp channel.
- [ ] Upgrade surfaces (honest, non-dark-pattern); tier visible in settings.
- [ ] Tests: each gate on/off by tier.
- **Acceptance:** flipping a household to Pro in the DB unlocks exactly the documented
  features; free behavior matches PRODUCT_PLAN §9.
- **Verify:** entitlement test matrix.

---

## 5. Cross-cutting checklists (apply throughout)

**Per API route:** zod-validate → `locals.db` → never trust client IDs → typed JSON →
explicit not-found/forbidden/conflict/tier-gate cases.

**Per webhook:** verify signature/secret → resolve sender to household → `InboundMessage`
idempotency → raw client with derived scope only → never leak errors to the caller.

**Per component:** semantic tokens only → five states → 44px targets → keyboard + focus
ring → reduced-motion → status never color-only.

**Per job:** idempotent → retry/backoff → stale-entity guard → outcomes logged →
registered + scheduled in `setup.ts`.

**Per AI call:** zod-validate the output → log the run → user confirms before it counts →
never fabricate (warranty terms, statutes) — missing data stays missing.

**Per merge:** Global DoD (§2) green → BUILD_LOG updated → no secret committed.

---

## 6. Risk watchlist (from DEV_PLAN §13 — keep visible while building)

- Extraction accuracy is the first impression → confirmation screen always in the loop;
  `NEEDS_REVIEW` never loses a file. (Sprint 3)
- Webhook trust boundaries — sender-derived scoping or someone writes into a stranger's
  vault. (Sprints 6, 11)
- Claim-timeline immutability is a product promise → schema-level `@@deny`, tested.
  (Sprints 2, 8)
- iOS web push unreliability → Capacitor push + email fallback carry the retention
  engine. (Sprints 5, 10)
- Legal content correctness → statutes from reviewed static content only; disclaimers.
  (Sprints 7–8)
- WhatsApp platform policy (templates, 24h window, opt-in) → module last, separable.
  (Sprint 11)

---

_Build in order. Gate on each Definition of Done. Surface conflicts instead of guessing.
The product promise is trust — a vault that loses a receipt or fumbles a claim timeline
breaks it permanently._
