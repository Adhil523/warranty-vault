# WarrantyVault — Implementation Plan

## Overview

WarrantyVault is a consumer app that stores purchase proof, tracks warranty periods, and
actively helps users fight warranty claims. Full product intent lives in
[`PRODUCT_PLAN.md`](./PRODUCT_PLAN.md) — that document is the source of truth for *what*
and *why*; this document decides *how*. The build is sequenced in
[`SPRINT_PLAN.md`](./SPRINT_PLAN.md); visuals in [`UI_DESIGN.md`](./UI_DESIGN.md).

Three architectural convictions shape everything below:

1. **The extraction pipeline is asynchronous from day one.** Capture returns instantly
   (product principle §4.1: under 10 seconds); AI extraction, email parsing, and WhatsApp
   processing all run as background jobs. Retrofitting async later would be a rewrite.
2. **Access control lives in the schema.** Household sharing, claim-timeline immutability,
   and account isolation are ZenStack `@@allow`/`@@deny` policies, not hand-rolled `where`
   filters. The claim timeline is tamper-evident because the schema forbids edits.
3. **Every capture channel converges on one pipeline.** In-app photo, file upload, manual
   entry, forwarded email, and WhatsApp all produce the same `(Document[], CaptureSource)`
   input to the same extraction → confirmation flow. Channels are thin adapters.

---

## 1. Tech Stack

| Concern         | Decision                                                                  |
| --------------- | ------------------------------------------------------------------------- |
| Framework       | SvelteKit 2, adapter-node, TypeScript strict, Svelte 5 runes              |
| Package manager | pnpm                                                                       |
| Auth            | better-auth (Prisma adapter) — email + phone OTP + Google/Apple           |
| ORM / DB        | ZenStack over Prisma; PostgreSQL (Supabase)                               |
| File storage    | Supabase Storage (private buckets, signed URLs) — never binaries in PG    |
| Jobs            | pg-boss; `register*`+`schedule*` pattern; init in `hooks.server.ts`       |
| AI extraction   | Claude API — `claude-sonnet-5` vision for documents, structured JSON out  |
| AI drafting     | Claude API — claim messages, follow-ups, escalation drafts                |
| Inbound email   | Postmark inbound webhooks (unique per-user forwarding addresses)          |
| Outbound email  | Postmark transactional                                                    |
| Push            | FCM (Firebase Admin) — app installs                                       |
| WhatsApp module | Meta WhatsApp Business Cloud API (webhooks + Graph API media download)    |
| Mobile shell    | Capacitor over the web views (native camera + push; deep links)           |
| Styling         | Tailwind 4 + `global.css` `@theme` tokens (UI_DESIGN §2)                  |
| Validation      | zod at every route and webhook boundary                                   |
| PDF (dossier)   | HTML template → headless Chromium print (Playwright) in a job             |
| Testing         | Vitest (unit/integration) + Playwright (key flows, later sprints)         |

**Conventions** (carried from Perch — they worked): cuid IDs; `householdID`-style FK
naming; `createdAt`/`updatedAt` on every model; routes use `locals.db` (policy-enhanced);
jobs and webhooks use `dangerousPrismaDoNotUse` with explicit scoping; jobs export
`registerXWorker()` + `scheduleX()` wired in `setup.ts`; shared client/server contracts in
`$lib/**` as zod schemas + inferred types; tests live beside code as `*.spec.ts`.

## 2. Architecture at a glance

```
 Capture channels (thin adapters)                    Core
 ┌──────────────────────────────┐        ┌───────────────────────────┐
 │ App camera / file upload     │──┐     │ POST /api/capture         │
 │ Manual entry form            │──┼────▶│  → store file(s)          │
 │ Postmark inbound webhook     │──┤     │  → create Item (PENDING)  │
 │ WhatsApp Cloud API webhook   │──┘     │  → enqueue extract job    │
 └──────────────────────────────┘        └────────────┬──────────────┘
                                                      ▼
                                         extract-document.job (Claude)
                                                      ▼
                                         Item NEEDS_CONFIRMATION ──▶ user confirms
                                                      ▼
                       ┌─────────────── Item ACTIVE (warranty tracked) ───────────────┐
                       ▼                              ▼                               ▼
              expiry-sweep.job                weekly-digest.job              Claims engine
              (30d/7d alerts)                                        (intake → route → draft →
                                                                      track → follow-up →
                                                                      escalate → outcome)
```

- One Postgres database (Supabase). One SvelteKit node process serves web + API + workers
  (pg-boss runs in-process; can split to a dedicated worker later without code changes).
- Documents live in a private Supabase Storage bucket; the DB stores keys, hashes, and
  metadata. Clients get short-lived signed URLs from the API.
- Webhooks (email, WhatsApp) have no user session — they authenticate the *sender*
  (forwarding address / linked phone number) and use the raw client with explicit scoping.
- No realtime channel is needed in v1 (unlike Perch): the vault is not a live-contention
  surface. TanStack Query polling/invalidations are enough.

## 3. External Services & Integrations

### 3.1 better-auth (identity)

- Email+password and phone OTP (primary personas live on their phones); Google + Apple
  social sign-in. Prisma adapter on the same Postgres.
- Onboarding flow requires **capture before account** (PRODUCT_PLAN §8): the first capture
  is held client-side (or as an anonymous draft) and committed on account creation. Keep
  this in mind in Sprint 3 — the capture endpoint must accept a just-created session.
- `hooks.server.ts` resolves session → `locals.session` → `{ userID, householdID }`.
  Every user gets a personal Household at signup (a vault of one); sharing later just adds
  members — **no data migration when a spouse joins.**

### 3.2 Claude API — extraction

- `extractPurchase(images | pdf) → ExtractedFields` — one vision call per capture with a
  strict JSON schema (tool use / structured output), zod-validated on return. Fields:
  productName, brand, sellerName, purchaseDate, price, currency, invoiceNumber, category,
  statedWarrantyMonths (nullable), perFieldConfidence.
- Model via env (`EXTRACTION_MODEL`, default `claude-sonnet-5`). Never trust silently:
  extraction lands the Item in `NEEDS_CONFIRMATION`; the confirmation screen is the gate.
- When the document states no warranty: look up `WarrantyDefault` (category+brand → typical
  months) and label the result `ESTIMATED` (PRODUCT_PLAN §5.2).
- Log every run to `ExtractionRun` (model, latency, raw JSON, outcome) — this is the
  debugging surface and the future training set for the "priority extraction review" Pro
  feature.

### 3.3 Claude API — claim drafting

- `draftClaimMessage(claim, stage, tone, language)` — stages: `INITIAL`, `FOLLOW_UP`,
  `ESCALATION_DENIAL_REQUEST`, `ESCALATION_LEGAL_NOTICE`, `ESCALATION_FORUM_COMPLAINT`.
- Legal escalation content is **jurisdiction-pluggable**: templates + statute references in
  `src/lib/content/jurisdictions/in.ts` (India: Consumer Protection Act, 2019) behind a
  `Jurisdiction` interface. The model fills templates; it never invents statutes. Every
  legal draft carries the not-legal-advice disclaimer (PRODUCT_PLAN §10).

### 3.4 Postmark — inbound + outbound email

- Inbound: each user/household gets `{slug}@inbox.<domain>`; Postmark posts parsed JSON
  (attachments base64) to `POST /api/webhooks/email`. Verify by Postmark's webhook auth +
  a shared secret in the URL. Unknown address → drop and log.
- Outbound: expiry alerts (fallback channel), digests, household invites.

### 3.5 FCM — native push

- Primary alert channel once the Capacitor app is installed. `POST /api/devices`
  registers/refreshes `DeviceToken`. Web users fall back to email. Same layered-guarantee
  pattern as Perch: push → email fallback, every attempt logged, dead tokens pruned.

### 3.6 WhatsApp Business Cloud API (optional module — build last, cleanly separable)

- Webhook `POST /api/webhooks/whatsapp` (GET for Meta's verify handshake). All logic lives
  in `src/lib/server/whatsapp/` — deleting that folder must not break the core app.
- Media (images/PDFs) downloaded via Graph API using the media ID, then fed to the standard
  capture pipeline with `source: WHATSAPP`.
- Multi-image grouping: images from the same sender within a **90-second window** become
  candidate pages of one purchase; the bot asks one clarifying question (PRODUCT_PLAN §7.2).
  Implemented as a debounced job (`whatsapp-group-window.job`) keyed on `waMessageGroup`.
- Business-initiated notifications require pre-approved template messages + explicit opt-in;
  `STOP` reply flips the opt-out flag. Free-form replies are fine within the 24-hour
  customer-service window.

### 3.7 Duplicate detection

- `sha256` of file bytes stored on `Document.contentHash` (unique per household). Exact
  match on any channel → attach/merge into the existing Item instead of creating a new one,
  and say so in the confirmation (PRODUCT_PLAN §7.3). Fuzzy dedupe (same
  seller+date+price) surfaces a "possible duplicate" hint on the confirmation screen —
  never auto-merges.

## 4. Environment Variables

```bash
# Database (Supabase)
DATABASE_URL=            # pooled
DIRECT_URL=              # direct, for migrations

# Supabase storage
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=   # server-only; storage access
STORAGE_BUCKET=documents

# better-auth
BETTER_AUTH_SECRET=
BETTER_AUTH_URL=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
APPLE_CLIENT_ID=             # later sprint; absence degrades gracefully
APPLE_CLIENT_SECRET=

# Claude API
ANTHROPIC_API_KEY=
EXTRACTION_MODEL=claude-sonnet-5
DRAFTING_MODEL=claude-sonnet-5

# Postmark
POSTMARK_SERVER_TOKEN=
POSTMARK_INBOUND_SECRET=     # random slug segment in the webhook URL
INBOUND_EMAIL_DOMAIN=inbox.warrantyvault.app
EMAIL_FROM=alerts@warrantyvault.app

# Push (Firebase Admin / FCM)
FIREBASE_PROJECT_ID=
FIREBASE_CLIENT_EMAIL=
FIREBASE_PRIVATE_KEY=

# WhatsApp Cloud API (optional module)
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_VERIFY_TOKEN=       # webhook handshake
WHATSAPP_APP_SECRET=         # payload signature check

# App
PUBLIC_APP_URL=
NODE_ENV=
```

Every integration must **degrade gracefully when its vars are absent** (log a warning,
skip the channel) — Perch proved this keeps early sprints unblocked.

## 5. Data Model (ZenStack / `schema.zmodel`)

> Sketch — the builder authors the full zmodel in Sprint 2. Policies shown are the
> load-bearing ones. `auth()` is the better-auth user.

```zmodel
model User {
  id           String   @id @default(cuid())
  name         String?
  email        String?  @unique
  phone        String?  @unique
  // better-auth relation fields...
  memberships  HouseholdMember[]
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  @@allow('read,update', auth().id == id)
}

model Household {                       // "the vault" — every user gets one at signup
  id            String  @id @default(cuid())
  name          String  @default("My vault")
  inboundSlug   String  @unique        // {slug}@inbox.<domain>
  jurisdiction  String  @default("IN")
  members       HouseholdMember[]
  items         Item[]
  whatsappLinks WhatsAppLink[]
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  @@allow('read,update', members?[userID == auth().id])
}

model HouseholdMember {
  id           String    @id @default(cuid())
  householdID  String
  household    Household @relation(fields: [householdID], references: [id])
  userID       String
  user         User      @relation(fields: [userID], references: [id])
  // Phase 1: all members equal (PRODUCT_PLAN §5.5)
  createdAt    DateTime  @default(now())

  @@unique([householdID, userID])
  @@allow('read', household.members?[userID == auth().id])
  @@allow('create,delete', household.members?[userID == auth().id]) // invite/leave; invites via server flow
}

enum ItemStatus     { PENDING_EXTRACTION  NEEDS_CONFIRMATION  ACTIVE  NEEDS_REVIEW }
enum WarrantySource { STATED  ESTIMATED  USER_CONFIRMED  UNKNOWN }
enum CaptureSource  { APP_CAMERA  APP_UPLOAD  MANUAL  EMAIL  WHATSAPP }

model Item {
  id               String    @id @default(cuid())
  householdID      String
  household        Household @relation(fields: [householdID], references: [id])
  status           ItemStatus @default(PENDING_EXTRACTION)
  source           CaptureSource
  productName      String?
  brand            String?
  sellerName       String?
  category         String?          // electronics | appliance | furniture | vehicle | ...
  purchaseDate     DateTime?
  price            Decimal?
  currency         String  @default("INR")
  invoiceNumber    String?
  warrantyMonths   Int?
  warrantyEndDate  DateTime?        // derived on confirm; indexed for expiry sweeps
  warrantySource   WarrantySource @default(UNKNOWN)
  documents        Document[]
  claims           Claim[]
  extractionRuns   ExtractionRun[]
  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt

  @@index([householdID, warrantyEndDate])
  @@allow('all', household.members?[userID == auth().id])
}

model Document {
  id            String   @id @default(cuid())
  itemID        String
  item          Item     @relation(fields: [itemID], references: [id])
  householdID   String                    // denormalized for the hash-dedupe unique
  storageKey    String
  mimeType      String
  pageIndex     Int      @default(0)
  contentHash   String                    // sha256 — dedupe across channels
  kind          String   @default("PROOF") // PROOF | DEFECT_EVIDENCE
  source        CaptureSource
  createdAt     DateTime @default(now())

  @@unique([householdID, contentHash])
  @@allow('read,create', item.household.members?[userID == auth().id])
  @@allow('delete', item.household.members?[userID == auth().id])
}

model ExtractionRun {
  id         String   @id @default(cuid())
  itemID     String
  item       Item     @relation(fields: [itemID], references: [id])
  model      String
  status     String                       // SUCCEEDED | FAILED
  rawResult  Json?
  confidence Json?                        // per-field
  latencyMs  Int?
  createdAt  DateTime @default(now())

  @@allow('read', item.household.members?[userID == auth().id])
}

model WarrantyDefault {                   // typical-duration lookup (seeded, growable)
  id        String @id @default(cuid())
  category  String
  brand     String?                       // null = category-wide default
  months    Int
  @@unique([category, brand])
  @@allow('read', auth() != null)
}

model BrandChannel {                      // claims-routing knowledge base (§6.2)
  id           String  @id @default(cuid())
  brand        String
  category     String?
  region       String  @default("IN")
  channelType  String                     // EMAIL | PHONE | PORTAL | SELLER_FIRST
  value        String                     // address / number / URL / policy note
  verified     Boolean @default(false)    // user-confirmed lookups get true
  @@index([brand, region])
  @@allow('read', auth() != null)         // writes via server flow after user confirms
}

enum ClaimStatus { DRAFTED  SENT  AWAITING_RESPONSE  IN_SERVICE  RESOLVED  DENIED  ESCALATED }
enum ClaimOutcome { REPAIRED  REPLACED  REFUNDED  DENIED }

model Claim {
  id                String      @id @default(cuid())
  itemID            String
  item              Item        @relation(fields: [itemID], references: [id])
  status            ClaimStatus @default(DRAFTED)
  defectDescription String                 // AI-summarized from intake
  defectStartedAt   DateTime?
  tone              String      @default("polite")   // polite | firm
  language          String      @default("en")
  followUpDays      Int         @default(7)          // user-visible timer (§6.3)
  nextFollowUpAt    DateTime?
  outcome           ClaimOutcome?
  valueRecovered    Decimal?               // feeds the "money recovered" total (§6.5)
  events            ClaimEvent[]
  drafts            ClaimDraft[]
  createdAt         DateTime    @default(now())
  updatedAt         DateTime    @updatedAt

  @@allow('all', item.household.members?[userID == auth().id])
}

model ClaimEvent {                        // APPEND-ONLY — the evidence timeline (§6.4, §10)
  id        String   @id @default(cuid())
  claimID   String
  claim     Claim    @relation(fields: [claimID], references: [id])
  type      String                        // STATUS_CHANGE | USER_NOTE | MESSAGE_SENT | ...
  payload   Json
  createdAt DateTime @default(now())

  @@allow('read,create', claim.item.household.members?[userID == auth().id])
  @@deny('update,delete', true)           // tamper-evident: no edits, ever, for anyone
}

model ClaimDraft {
  id        String   @id @default(cuid())
  claimID   String
  claim     Claim    @relation(fields: [claimID], references: [id])
  stage     String                        // INITIAL | FOLLOW_UP | ESCALATION_* (§3.3)
  body      String
  model     String
  createdAt DateTime @default(now())
  @@allow('read,create', claim.item.household.members?[userID == auth().id])
}

model DeviceToken { ... }                 // as in Perch: userID, token, platform, lastSeenAt
model Notification { ... }                // log of every alert attempt: channel, status, error
model NotificationPref {                  // per-user: push/email/whatsapp toggles, digest opt-in
  ...
}

model WhatsAppLink {                      // optional module
  id           String    @id @default(cuid())
  householdID  String
  household    Household @relation(fields: [householdID], references: [id])
  phoneE164    String    @unique          // one number → one account (§7.1)
  linkedByID   String
  notifyOptIn  Boolean   @default(false)
  createdAt    DateTime  @default(now())
  @@allow('read,delete', household.members?[userID == auth().id])
}

model InboundMessage {                    // idempotency + audit for email/WhatsApp webhooks
  id          String   @id @default(cuid())
  channel     String                      // EMAIL | WHATSAPP
  externalID  String                      // Postmark MessageID / WA message id
  payload     Json
  status      String                      // RECEIVED | PROCESSED | REJECTED | FAILED
  createdAt   DateTime @default(now())
  @@unique([channel, externalID])         // webhook retries are no-ops
}

model LinkCode { ... }                    // one-time codes: WhatsApp linking, household invites
```

**Policy notes**

- Everything user-facing is scoped through `household.members?[userID == auth().id]` —
  household sharing (§5.5) is *free* once this is the only access path.
- `ClaimEvent` is the immutability showpiece: `@@deny('update,delete', true)` +
  timestamps = the tamper-evident timeline PRODUCT_PLAN §10 requires. Status changes on
  `Claim` always also write a `ClaimEvent` (in one transaction, in the service layer).
- Webhooks and jobs use `dangerousPrismaDoNotUse` — they must **derive the household from
  the sender** (inbound slug / linked phone), never from payload-supplied IDs.
- Account deletion (§10) is a server flow: delete storage objects, then cascade DB rows.

## 6. Core Services (`src/lib/server/services/`)

### 6.1 `capture.ts` — the single funnel

`createCapture({ householdID, source, files | manualFields, dedupeKey? })`:
store files → hash → dedupe check (attach vs. create) → create `Item`
(`PENDING_EXTRACTION`, or `ACTIVE` for complete manual entries) → enqueue
`extract-document` → return the item id immediately. Every channel calls this and nothing
else. Multi-page: multiple files = one item, `pageIndex` ordered.

### 6.2 `extraction.ts`

Runs inside the job: signed-read the documents → Claude vision call (§3.2) → zod-validate
→ write fields + `ExtractionRun` → status `NEEDS_CONFIRMATION` (or `NEEDS_REVIEW` on
failure — the raw file is already safe in the vault, §7.3). Warranty inference from
`WarrantyDefault` when unstated, marked `ESTIMATED`.

### 6.3 `warranty.ts` — pure functions, exhaustively tested

`warrantyEndDate(purchaseDate, months)`, `warrantyStatus(item, now)` →
`ACTIVE | EXPIRING_SOON (≤30d) | EXPIRED | UNKNOWN`, `monthsLeft`, and the alert-schedule
helpers the sweep uses (30d / 7d thresholds, §5.4). No I/O — this is the logic the whole
product promise rests on.

### 6.4 `claims.ts`

`startClaim` (intake → AI defect summary → assemble Claim File), `transitionClaim`
(status change + `ClaimEvent` in one txn), `recordOutcome` (outcome + valueRecovered →
the running recovered total is `SUM(valueRecovered)` over the household), free-tier gate
(`countActiveClaims` — one active claim on free, §9).

### 6.5 `routing.ts`

`resolveChannel(brand, category, region)` → `BrandChannel` hit, or a
search-then-confirm-with-user flow that writes back `verified: true` rows (§6.2 — the KB
grows from real usage).

### 6.6 `drafting.ts`

Stage-aware draft generation (§3.3) with jurisdiction plug-in content. Drafts are stored
(`ClaimDraft`), never auto-sent (§6.3: the user sends everything themselves in this phase).
Follow-up scheduling sets `nextFollowUpAt = sentAt + followUpDays`.

### 6.7 `notify.ts` / `push.ts` / `email.ts`

Layered delivery exactly as Perch: enqueue → push to devices → email fallback → (opt-in)
WhatsApp template → log every attempt to `Notification`. Failures are never silent.

### 6.8 `export.ts`

Vault export (zip of documents + structured JSON) and the **claim dossier PDF**: HTML
timeline template → Chromium print, includes every `ClaimEvent`, document, and draft
(§10 — "exportable as a single PDF dossier"). Both run as jobs, delivered by signed URL.

## 7. pg-boss Jobs (`src/lib/server/jobs/`)

| Job                          | Trigger              | Does                                                             |
| ---------------------------- | -------------------- | ---------------------------------------------------------------- |
| `extract-document.job`       | on capture           | §6.2 pipeline; retry ×3 backoff; terminal failure → NEEDS_REVIEW |
| `expiry-sweep.job`           | daily, per-tz        | 30d/7d expiry alerts; extended-warranty explainer for high-value |
| `claim-followup-sweep.job`   | hourly               | `nextFollowUpAt` due → nudge with pre-drafted follow-up (§6.3)   |
| `weekly-digest.job`          | weekly (opt-in)      | newly expiring + open claims summary (§5.4)                      |
| `send-notification.job`      | on demand            | layered delivery §6.7                                            |
| `export-vault.job` / `export-dossier.job` | on demand | §6.8                                                    |
| `wa-group-window.job`        | debounced 90s        | WhatsApp multi-image grouping (§3.6)                             |

All jobs: idempotent, stale-entity guarded, outcomes logged, registered + scheduled in
`setup.ts`, initialized from `hooks.server.ts`.

## 8. API Routes

All bodies zod-validated; routes use `locals.db`; webhooks use raw client + sender-derived
scoping.

- **Auth** — `/api/auth/*` (better-auth in-hook, Perch pattern)
- **Capture** — `POST /api/capture` (multipart files or manual fields → §6.1);
  `GET /api/items/[id]/extraction` (poll status for the confirmation screen)
- **Items** — `GET /api/items` (filter/sort/search: expiry, category, brand, price, q);
  `GET/PATCH /api/items/[id]`; `POST /api/items/[id]/confirm` (the one-tap confirm/edit —
  sets warranty fields, `USER_CONFIRMED`, derives `warrantyEndDate`);
  `POST /api/items/[id]/documents` (add pages/warranty card);
  `GET /api/documents/[id]/url` (short-lived signed URL)
- **Claims** — `POST /api/items/[id]/claims` (intake); `GET /api/claims` +
  `GET /api/claims/[id]`; `POST /api/claims/[id]/transition` (one-tap status updates);
  `POST /api/claims/[id]/drafts` (generate stage draft); `POST /api/claims/[id]/outcome`;
  `GET /api/claims/[id]/dossier` (enqueue + fetch export)
- **Household** — `GET /api/household`; `POST /api/household/invites` +
  `POST /api/household/join` (code-based); `DELETE /api/household/members/[id]` (leave)
- **Me** — `GET /api/me` (client-guard identity: `{ userID, householdID, tier }`);
  `PATCH /api/me/notifications` (prefs); `POST /api/devices`;
  `POST /api/me/export` · `DELETE /api/me` (full removal, §10)
- **Webhooks** — `POST /api/webhooks/email/[secret]` (Postmark inbound);
  `GET+POST /api/webhooks/whatsapp` (Meta verify + inbound; signature-checked)
- **WhatsApp linking** — `POST /api/whatsapp/link-code` · `POST /api/whatsapp/unlink`

## 9. Frontend Pages & Components

Route structure (mirrors Perch's client-guard convention — universal `+page.ts` fetches
`/api/me`, no server `load`):

```
src/routes/
  (auth)/sign-in, sign-up
  onboarding/            # value promise → camera-first capture → account → email/WA prompts (§8)
  vault/                 # home: item list + status chips + recovered-money hero
  vault/capture/         # camera / upload / manual tabs
  vault/items/[id]/      # item detail: documents, timeline bar, actions
  vault/items/[id]/confirm/  # extraction confirmation (the trust gate)
  claims/                # claim list
  claims/[id]/           # tracker: timeline, drafts, follow-up timer, escalate
  claims/[id]/intake/    # "Something broke" guided flow
  settings/              # household, notifications, WhatsApp link, export/delete, tier
```

Key components: `CaptureSheet` (camera/multi-page), `ExtractionConfirmCard` (per-field
edit, "estimated — tap to confirm" chips), `ItemRow` + `WarrantyChip`,
`WarrantyTimelineBar`, `RecoveredTotalHero`, `ClaimTimeline` (append-only event list),
`DraftComposer` (tone/language toggles, copy-to-send), `ChannelCard` (routing result),
`HouseholdPanel`. Layouts and states in UI_DESIGN §5–6.

## 10. UX & Trust Details

- Capture must feel instant: optimistic item row appears immediately with a shimmer
  "reading your receipt…" state; extraction completes in the background (§4.1, §4.2).
- Never silently trust extraction — the confirm screen is mandatory before an item counts
  as tracked; estimated warranties are visibly labeled (§5.2, §10).
- Notification permission is asked **at first warranty registration**, framed around that
  item (§8.5) — never on app open.
- The recovered-money total becomes the vault home hero once nonzero (§6.5).
- Every claim screen with legal drafts shows the document-preparation disclaimer (§10).
- Free-tier gate (one active claim) is enforced server-side in `claims.ts`, surfaced as an
  upgrade prompt, never as a data-hiding trick (§9).

## 11. File Tree (abbrev)

```
src/
  lib/
    server/
      auth.ts  db.ts  storage.ts
      services/ capture.ts extraction.ts warranty.ts claims.ts routing.ts
                drafting.ts notify.ts push.ts email.ts export.ts
      jobs/     setup.ts extract-document.job.ts expiry-sweep.job.ts
                claim-followup-sweep.job.ts weekly-digest.job.ts
                send-notification.job.ts export.job.ts
      whatsapp/ webhook.ts media.ts grouping.ts replies.ts   # separable module
      email-inbound/ parse.ts
    content/jurisdictions/ in.ts  types.ts
    contracts/            # zod schemas shared client/server (items, claims, capture)
    components/ ...
  routes/ ...             # §9
schema.zmodel
prisma/ (generated) · seed.ts (WarrantyDefault + BrandChannel seed rows, demo household)
```

## 12. Implementation Order

Follows PRODUCT_PLAN §13, sequenced in SPRINT_PLAN: foundation → auth/household → schema
& storage → capture+extraction+confirm → vault UI → warranty lifecycle & notifications →
email forwarding → claims (intake/routing/drafts, then tracker/escalation/outcome) →
household sharing → packaging & trust hardening → WhatsApp module → monetization gates.

## 13. Known Risks & Mitigations

1. **Extraction accuracy is the first impression.** Mitigate: confirmation screen always
   in the loop; per-field confidence surfaced; `ExtractionRun` logging from day one;
   `NEEDS_REVIEW` never loses the raw file.
2. **AI cost per capture.** One vision call per capture at Sonnet pricing is fine at MVP
   volume; `EXTRACTION_MODEL` env allows dropping to Haiku per-category later. Track
   latency/cost in `ExtractionRun`.
3. **iOS web push is unreliable** → expiry alerts (the retention engine) ride the
   Capacitor native shell + email fallback. Don't ship the "web-only" version to real
   users as the primary experience.
4. **Webhook trust boundaries.** Email/WhatsApp handlers bypass session auth — sender
   derivation + `InboundMessage` idempotency + signature checks are non-negotiable; a bug
   here writes into someone else's vault.
5. **Legal-content correctness (India CPA).** Statute references live in reviewed static
   content, not model output; the model only fills templates. Disclaimer everywhere.
6. **WhatsApp platform policy** (template approval, 24h window, opt-in records) — build
   the module last; the product must be whole without it (§7).
7. **Claim-timeline integrity is a product promise** — `@@deny` on update/delete plus
   transactional event writes; tested explicitly (attempted edit must fail).
