# WarrantyVault — Handoff (for a fresh Claude instance)

> **Read this first, then [`SPRINT_PLAN.md`](./SPRINT_PLAN.md), [`DEV_PLAN.md`](./DEV_PLAN.md),
> [`UI_DESIGN.md`](./UI_DESIGN.md), [`SETUP.md`](./SETUP.md).** PRODUCT_PLAN.md is the
> product spec (the *why* behind every task) — skim it fully once before Sprint 0.
> **As of:** 2026-07-11 — **greenfield: no code exists yet.** This doc pack is the entire
> project state.

---

## 1. What WarrantyVault is

A consumer app (mobile-first, India-first) that stores purchase proof, tracks warranty
expiry, and — the differentiator — actively fights warranty claims: drafts the messages,
finds the right channel, tracks a tamper-evident timeline, escalates with
consumer-protection-grade language when brands stall. "Never lose a receipt, never lose a
claim." Product intent: PRODUCT_PLAN.md · architecture: DEV_PLAN.md · visuals:
UI_DESIGN.md · sequencing: SPRINT_PLAN.md · services/env: SETUP.md.

## 2. Where the stack came from

The stack (SvelteKit 2 + ZenStack/Prisma + Postgres/Supabase + pg-boss + Capacitor +
better-auth + Claude API) was deliberately chosen to mirror **Perch** (the owner's
previous product, same author) because it fits this product's shape — household-scoped
access policies, an append-only evidence log, job-heavy async pipelines — and because the
conventions are already proven. When in doubt about an idiom (client guards without
server `load`, `locals.db` vs `dangerousPrismaDoNotUse`, the `register*`/`schedule*` job
pattern, zod contracts in `$lib`), DEV_PLAN §1 records them; they are requirements, not
suggestions.

Three decisions that are **not** Perch-inherited, so don't copy blindly:
1. **No realtime channel** — the vault has no live-contention surface; TanStack Query is
   enough (DEV_PLAN §2).
2. **Postmark, not Resend** — inbound email parsing (the forwarding feature) and outbound
   in one provider (DEV_PLAN §3.4).
3. **Light-first UI** with its own token set (UI_DESIGN §2) — do not import Perch's dark
   brand system.

## 3. The working agreement (HOUSE RULES — follow these)

1. **One sprint at a time, in order** (SPRINT_PLAN §0). Each: implement → self-review
   (`/simplify` + `/code-review` mindset) → gates green → **commit** → branch the next
   sprint off the current one (cascading `sprint/N` ← `sprint/N-1`).
2. **Gates before every commit:** `pnpm check` (0 errors) · `pnpm lint` · `pnpm test` ·
   `pnpm build`. Husky pre-commit runs `lint`+`check`.
3. **Schema changes only in `schema.zmodel`; migrations only via `pnpm run gen`.**
   `prisma/schema.prisma` is generated — never edit it. (If the gen script prompts for a
   migration name, pipe it: `printf 'name\n' | pnpm run gen`. For hand-written SQL,
   `--create-only`, edit, then apply.)
4. **No `+page.server.ts` / server `load`.** API via `+server.ts`; pages are universal
   (`+page.ts` fetches `/api/...`); guards client-side via `/api/me`. This keeps the app
   SPA-shaped for the Capacitor shell. adapter-node hosts the API.
5. **Every capture channel goes through `capture.ts`** — no exceptions (SPRINT_PLAN §1.7).
6. **Semantic tokens only** in components; five states; 44px targets; keyboard +
   reduced-motion; status never color-only (UI_DESIGN).
7. **Use the UI skills for any UI work** (`emil-design-eng`, `design-taste-frontend`,
   `impeccable`) and **TanStack Query for all client data** (ZenStack hooks for CRUD;
   `createQuery` for custom endpoints; never bare `fetch` in components).
8. Don't scope-creep; park ideas in `docs/BACKLOG.md`. Append to `docs/BUILD_LOG.md`
   every sprint. Surface real conflicts instead of guessing.

## 4. Current state & first steps

- **Nothing is built.** No repo scaffold, no env, no Supabase project for this app yet.
- Step 1: initialize the repo (git init if standalone), then **Sprint 0** exactly as
  written. Pick a fresh Supabase port block first (SETUP §0 note — `553xx`/`12xxx`/`64xxx`
  are taken on this machine).
- Step 2 onward: follow SPRINT_PLAN. Sprint 3 (capture→extraction→confirm) is the heart
  of the product — budget the most care there.
- `.env` values: the owner provides real keys per SETUP as each sprint needs them; build
  so absent providers degrade gracefully (SETUP preamble).

## 5. Known decisions already made (don't re-litigate)

- Async extraction pipeline from day one; capture returns instantly (DEV_PLAN Overview).
- Personal-household-at-signup so sharing needs no data migration (DEV_PLAN §3.1).
- `ClaimEvent` immutability at the schema level (`@@deny('update,delete', true)`) — it's
  the §10 tamper-evidence requirement, not a style choice.
- WhatsApp module last and separable; core app must be whole without it.
- India (`IN`) is the first jurisdiction; legal content is pluggable static content, the
  model never invents statutes.
- Free tier = one active claim; gates server-side (DEV_PLAN §6.4).

## 6. Open items for the owner (ask when relevant, don't block on them)

- Production domain (affects `INBOUND_EMAIL_DOMAIN`, `PUBLIC_APP_URL`, OAuth redirects).
- SMS provider for phone OTP (dev logs the code to console until then).
- WhatsApp Business number + Meta business verification (needed before Sprint 11; template
  approval has days of lead time — flag it when Sprint 9 starts).
- Payment processor choice (Sprint 12 builds gates only, no checkout, unless told).
- App Store / Play accounts for the Sprint 10 device runs.

## 7. Conventions cheat-sheet

- IDs `cuid`; FKs `householdID`-style; `createdAt`/`updatedAt` everywhere. Routes use
  `locals.db` (policy-enhanced); jobs/webhooks use `dangerousPrismaDoNotUse` with
  **sender-derived** scoping + `InboundMessage` idempotency.
- Jobs export `registerXWorker()`+`scheduleX()`, wired in `setup.ts`, init from
  `hooks.server.ts`.
- Shared contracts: zod schemas + inferred types in `$lib/contracts/**` — routes and
  pages import the same schema so they can't drift.
- Tests beside code (`*.spec.ts`); behavior-asserting; integration tests run against the
  local DB (`skipIf !DATABASE_URL`); every `it` needs an `expect`
  (`requireAssertions: true`).
- Money is `Decimal`, currency `INR` default; all user-visible dates/amounts render in
  the mono face (UI_DESIGN §2.2).
