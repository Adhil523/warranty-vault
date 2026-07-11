# WarrantyVault — UI Design System

> Direction + tokens for the product app. The builder establishes `global.css` `@theme`
> from §2 in Sprint 0 and **never invents new hex or fonts after that**. Components
> consume only the semantic aliases (§2.5). Personality target (PRODUCT_PLAN §1):
> *persistent, organized, slightly relentless on the user's behalf* — a competent
> advocate, not a cute filing app.

---

## 1. Principles

1. **Calm archive, urgent advocate.** The vault is quiet, warm, paper-like — documents at
   rest. The moment something is expiring or a claim is live, the UI sharpens: high
   contrast, clear deadlines, one obvious next action.
2. **Status is the spine.** Warranty state and claim state drive nearly every surface;
   the chip/timeline system (§3) must be legible at a glance and never color-only.
3. **Capture is sacred.** Anything on the capture path is optimized for one-handed,
   sub-10-second use: huge targets, no optional questions, progress always visible.
4. **Numbers earn trust.** Dates, amounts, invoice numbers, and the recovered-money total
   render in the mono numeric face — precise things should look precise.
5. **Mobile-first, phone-real.** Design at 390px; the Capacitor app is the primary
   surface. Desktop is a stretched, not separate, layout (§8).

## 2. Foundations

### 2.1 Color primitives (define verbatim in `@theme`)

Light-first (receipts live on paper); full dark theme from the same aliases.

| Token       | Hex       | Use                                        |
| ----------- | --------- | ------------------------------------------ |
| `paper`     | `#FAF7F1` | app background (warm off-white)            |
| `card`      | `#FFFFFF` | raised surfaces                            |
| `ink`       | `#1C221E` | primary text (near-black, green-warm)      |
| `slate`     | `#5C6660` | secondary text, Unknown status             |
| `line`      | `#E6E1D6` | borders, dividers                          |
| `pine`      | `#175D4C` | brand primary — actions, links, Active     |
| `pine-deep` | `#0E3F33` | pressed/hover, dark-theme primary surface  |
| `mint`      | `#DDEFE7` | Active-chip tint, success wash             |
| `amber`     | `#B45309` | Expiring soon, follow-up due               |
| `amber-wash`| `#FDF0DC` | expiring tint                              |
| `rust`      | `#B3261E` | Expired, Denied, destructive               |
| `rust-wash` | `#FBEAE8` | expired/danger tint                        |
| `brass`     | `#8A6D1D` | recovered-money accent (money, not danger) |

Dark theme: `paper→#141815`, `card→#1D2320`, `ink→#EDEAE2`, `line→#2C332E`, keep hues,
raise washes' lightness. Both themes ship from Sprint 0 via the alias layer.

### 2.2 Typography

- **Display / headings:** `Sora` (600/700) — sturdy, modern, advocate-voiced.
- **Body / UI:** `Inter` (400/500/600).
- **Numeric / evidence:** `IBM Plex Mono` (400/500) — dates, prices, invoice numbers,
  countdowns, the recovered total, timeline timestamps.
- Scale (mobile): display 28/34 · h1 22/28 · h2 18/24 · body 15/22 · caption 12/16.
  Minimum body on any surface: 13px.

### 2.3 Spacing, radius, elevation

- 4px base grid; screen gutter 16px; card padding 16px; section gap 24px.
- Radius: cards 16px · chips/inputs 10px · buttons 12px · document thumbnails 8px.
- Elevation: one soft shadow level for raised cards (`0 1px 3px rgb(0 0 0 / 0.07)`);
  everything else separates by `line` borders and background steps. No glassmorphism.

### 2.4 Motion

- Durations 150–250ms, ease-out; one signature moment: the **capture settle** (photo
  thumbnail flies into the vault list row) and the **recovered-total count-up**.
- Every animation respects `prefers-reduced-motion` (swap for opacity fades).

### 2.5 Semantic aliases (mandatory — components use only these)

`primary` `on-primary` `surface` `surface-raised` `border` `text` `text-muted` `success`
`success-wash` `warning` `warning-wash` `danger` `danger-wash` `accent-money` — mapped
from §2.1 per theme. A grep for primitive names (`pine`, `amber`, `rust`…) inside
`src/lib/components` must return nothing.

## 3. Status system (the spine)

### 3.1 Warranty chips (`WarrantyChip`)

| State          | Look                                   | Label                  |
| -------------- | -------------------------------------- | ---------------------- |
| Active         | `success-wash` bg, `success` text, ● icon | "Active · 14 mo left" |
| Expiring soon  | `warning-wash` bg, `warning` text, ◔ icon | "Expires in 23 days"  |
| Expired        | `danger-wash` bg, `danger` text, ○ icon   | "Expired May 2026"    |
| Unknown        | dashed `border`, `text-muted`, ? icon      | "Warranty unknown"    |
| Estimated      | any of the above + dotted underline + "est." suffix; tapping opens confirm |

Icon + words always accompany color (a11y §7). The chip is the same component everywhere:
list rows, item detail, claim header, notifications.

### 3.2 Claim status (`ClaimTimeline` + status pill)

Drafted → Sent → Awaiting response → In service → Resolved / Denied / Escalated.
Timeline is a vertical, timestamped (mono), append-only list — visually communicates
"this is evidence": no edit affordances, subtle lock glyph on past events. Follow-up
timer renders as a countdown pill ("Nudge ready in 3d") that flips to `warning` when due.

## 4. Core components

- `CaptureSheet` — full-screen camera with page counter ("Page 2"), gallery + file + manual
  tabs, giant shutter (≥72px). Post-shot: instant thumbnail + "reading your receipt…"
  shimmer row lands in the vault list.
- `ExtractionConfirmCard` — the trust gate: per-field rows (label, value, confidence-aware
  edit affordance), estimated-warranty row highlighted with the est. treatment, one
  primary "Looks right" button + per-field edit.
- `ItemRow` — thumbnail, product name (body-600), brand + purchase date (caption,
  mono date), `WarrantyChip` right-aligned.
- `WarrantyTimelineBar` — horizontal purchase→expiry bar, today marker, months-left in
  mono; `warning`/`danger` fill as expiry nears.
- `RecoveredTotalHero` — brass accent, mono count-up, "recovered with WarrantyVault";
  hidden until nonzero, then owns the top of the vault home.
- `ChannelCard` — routing result: channel icon, name, value (mono), "verified" tick or
  "confirm this is right" prompt.
- `DraftComposer` — draft body, tone (polite/firm) + language toggles, prominent **Copy**
  / **Share** (never "Send" — the user sends everything themselves), disclaimer footer on
  legal stages.
- `HouseholdPanel`, `NotificationPrefs`, `NeedsReviewBanner` ("Saved, needs a quick
  check" — item safe, extraction failed).

## 5. Key screens

### 5.1 Vault home (`/vault`)
`RecoveredTotalHero` (when nonzero) → search + filter row → item list sorted
soonest-expiring. Empty state: warm illustration + "Snap your first receipt" straight
into `CaptureSheet`. FAB or bottom-bar center action = capture, always one tap away.

### 5.2 Item detail (`/vault/items/[id]`)
Document pager (pinch-zoomable pages, source label "via WhatsApp/email") →
`WarrantyTimelineBar` → extracted fields (mono values) → actions: **Something broke —
start a claim** (primary), Add document, Edit details.

### 5.3 Capture + confirm (`/vault/capture`, `.../confirm`)
The sacred path (§1.3). Camera opens pre-warmed; multi-page is "add page", never a
settings toggle. Confirm screen is one scroll, one primary button.

### 5.4 Claim intake + tracker (`/claims/[id]/intake`, `/claims/[id]`)
Intake: three steps max (what/when/photos), progress dots, AI-summarized defect shown
back for approval. Tracker: status pill + `ClaimTimeline` + current `DraftComposer` +
`ChannelCard`; one-tap status updates ("They replied", "Picked up for repair") as large
list buttons. Escalation entry is deliberate: "They're stalling — escalate" with a short
explainer of the ladder.

### 5.5 Onboarding
Value promise (one screen, display type) → camera immediately → account creation after
first capture → forwarding-address + WhatsApp prompts → notification permission asked at
first warranty registration, framed on that item.

## 6. States (every interactive surface defines all five)

loading (skeletons matching final layout — no spinners on lists) · empty (helpful, points
to the next action) · error (plain-language + retry, never a bare code) · offline (vault
reads from cache with a stale banner; capture queues) · success. The extraction pipeline
adds two domain states: **reading** (shimmer) and **needs review** (banner, never a dead
end).

## 7. Accessibility (WCAG 2.2 AA — acceptance, not polish)

44px minimum targets; visible focus ring (`primary`, 2px offset); full keyboard path on
web; chips/status never color-only (§3); contrast ≥4.5:1 body / 3:1 large text in both
themes (§2.1 pairs are chosen for this — verify, don't trust); reduced-motion honored;
document viewer images carry meaningful `alt` (product + doc type); form errors
programmatically associated.

## 8. Responsive

Design at 390px. ≥768px: vault becomes a two-column master-detail; capture stays
full-screen; claim tracker gains a side-by-side timeline+draft layout. Never horizontal
scroll; tables/timelines scroll within their own container.

## 9. Do / Don't

- **Do** use mono for every date, amount, and identifier. **Don't** mix numeric faces.
- **Do** show source labels (via WhatsApp/email). **Don't** hide provenance — it's trust.
- **Do** keep destructive/legal actions deliberate (confirm + explainer).
  **Don't** add friction to capture — it gets zero confirmation dialogs.
- **Do** use `accent-money` only for recovered value. **Don't** spend it elsewhere; the
  green-money association is the retention engine.
- **Don't** invent hex, shadows, or fonts outside §2. **Don't** ship a color-only status.
