# WarrantyVault — Environment & Service Setup

> How to provision every external service and run the app locally. Pairs with
> [`DEV_PLAN.md`](./DEV_PLAN.md) §3 (integrations) and §4 (env vars). Every integration
> must **degrade gracefully when its env vars are absent** — a missing provider logs a
> warning and skips the channel; it never crashes the app or a job.

---

## 0. Quick start (local dev)

```bash
pnpm install
pnpm exec supabase start        # local Postgres + Storage (Docker)
cp .env.example .env            # fill values per sections below
pnpm run gen                    # zenstack generate + prisma migrate dev
pnpm run seed                   # WarrantyDefault + BrandChannel + demo household
pnpm dev
```

Gate scripts (run before every commit): `pnpm check` · `pnpm lint` · `pnpm test` ·
`pnpm build`.

> **Port note (this machine):** other Supabase projects occupy the `12xxx`, `553xx`, and
> `64xxx` port blocks (Perch uses `553xx`). Pick a **fresh block in `supabase/config.toml`
> (e.g. `56321/56322/56323`)** before the first `supabase start`, and point
> `DATABASE_URL`/`DIRECT_URL`/`SUPABASE_URL` at it.

## 1. Supabase — Postgres + Storage

- Local: `pnpm exec supabase start|stop|status` (CLI as a dev dependency, not global).
- `DATABASE_URL` (pooled) + `DIRECT_URL` (direct, migrations) → the local DB port.
- **Storage:** create one **private** bucket `documents` (local: via Studio or a seed
  script; prod: dashboard). Server accesses it with `SUPABASE_SERVICE_ROLE_KEY`
  (server-only — never `PUBLIC_`). Clients receive short-lived signed URLs from
  `GET /api/documents/[id]/url`; nothing in the bucket is ever public.
- Realtime is **not** used in v1 (DEV_PLAN §2).

## 2. better-auth — identity

- `BETTER_AUTH_SECRET` (32+ random bytes; dev placeholder fine, rotate for prod),
  `BETTER_AUTH_URL` = `PUBLIC_APP_URL`.
- Providers: email+password out of the box; **phone OTP** needs an SMS provider — defer
  the real SMS hookup until a later sprint (dev: log the OTP to console); Google
  (`GOOGLE_CLIENT_ID/SECRET` via Google Cloud Console → OAuth consent + web client);
  Apple deferred until the Capacitor sprint.
- Signup hook must create the personal Household (SPRINT_PLAN Sprint 1).

## 3. Claude API — extraction + drafting

- `ANTHROPIC_API_KEY` from the Anthropic Console.
- `EXTRACTION_MODEL=claude-sonnet-5`, `DRAFTING_MODEL=claude-sonnet-5` (env-switchable;
  cost lever per DEV_PLAN §13.2).
- Use the official TypeScript SDK; extraction calls use structured output (tool use) with
  the zod-mirrored JSON schema; set a hard `max_tokens` and a 60s timeout inside the job
  (pg-boss retries cover transient failures).
- No key → capture still works: items land in `NEEDS_REVIEW` with files preserved.

## 4. Postmark — inbound + outbound email

- One Postmark server; grab `POSTMARK_SERVER_TOKEN`.
- **Inbound:** configure the inbound domain (`INBOUND_EMAIL_DOMAIN`, e.g.
  `inbox.warrantyvault.app` — MX to Postmark), set the inbound webhook to
  `https://<app>/api/webhooks/email/<POSTMARK_INBOUND_SECRET>` (generate a long random
  secret). Local dev: use Postmark's webhook test payloads checked into
  `src/lib/server/email-inbound/__fixtures__/`, or a tunnel (`cloudflared`/`ngrok`).
- **Outbound:** verify the sender domain (DKIM + return-path); `EMAIL_FROM`.
- No token → email channel logs-and-skips (push/UI still work).

## 5. Firebase Cloud Messaging — native push

- Firebase project → service account → `FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`,
  `FIREBASE_PRIVATE_KEY` (keep the `\n` escaping). Needed for real pushes only from the
  Capacitor sprint; absent vars degrade to email.
- Android: `google-services.json`; iOS: APNs key uploaded to Firebase — done in Sprint 10.

## 6. WhatsApp Business Cloud API (optional module, Sprint 11)

- Meta developer app → WhatsApp product → business phone number.
  `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_ACCESS_TOKEN` (system-user token, not the 24h dev
  token), `WHATSAPP_APP_SECRET` (payload signature), `WHATSAPP_VERIFY_TOKEN` (any random
  string; used in the GET handshake).
- Webhook URL: `https://<app>/api/webhooks/whatsapp` — requires a public URL even in dev
  (tunnel). Subscribe to `messages`.
- Business-initiated notifications need **approved message templates** (submit early —
  approval takes days) and recorded user opt-in; `STOP` must opt out.
- Dev: use Meta's test number + sandbox recipients before the real number.

## 7. pg-boss — job queue

- Rides `DATABASE_URL` (own schema `pgboss`); no separate infra. Workers run in-process,
  initialized once from `hooks.server.ts` via `setup.ts` (register + schedule everything
  there — a job that isn't in `setup.ts` doesn't exist).

## 8. Secrets hygiene (precision rule)

- `.env` is gitignored; `.env.example` lists **every** var with placeholder values and is
  updated in the same commit that introduces a var.
- Server-only secrets never get the `PUBLIC_` prefix. The service-role key and Firebase
  private key never reach the client bundle — `pnpm build` + a grep of `.svelte-kit/output/client`
  for `SERVICE_ROLE` is the paranoid check.
- Webhook endpoints authenticate every call (Postmark URL secret; Meta signature).

## 9. Provisioning checklist

- [ ] Supabase local running on a fresh port block; `documents` bucket created (private)
- [ ] `.env` filled: DB, storage, better-auth, `ANTHROPIC_API_KEY`
- [ ] `pnpm run gen` clean; `pnpm run seed` loads defaults + demo household
- [ ] Google OAuth client (dev origin + redirect)
- [ ] Postmark: server token, inbound domain MX, inbound webhook + secret, DKIM verified
- [ ] Firebase service account (by Sprint 5; device wiring Sprint 10)
- [ ] Meta app + test number + template submissions (before Sprint 11)
- [ ] Prod only: rotate `BETTER_AUTH_SECRET`, system-user WA token, real APNs key
