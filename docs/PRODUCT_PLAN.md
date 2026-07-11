# Product Plan: WarrantyVault — "The Receipt Vault That Fights For You"

> **Note for the builder (added at handoff, 2026-07-11):** this is the canonical product
> spec, copied verbatim from `../warranty-vault-product-plan.md`. One instruction below is
> now superseded: the stack **has** been decided — see [`DEV_PLAN.md`](./DEV_PLAN.md) §1.
> Start with [`HANDOFF.md`](./HANDOFF.md); build via [`SPRINT_PLAN.md`](./SPRINT_PLAN.md).

## Instructions for the Builder
This document is a complete product specification intended to be handed to an AI coding assistant. It deliberately contains **no technology stack decisions** — choose whatever stack fits best. Build the **Core Product (Phase 1)** first, then the **Claims Engine (Phase 2)**, then the **optional WhatsApp Integration module** described in its own section. Everything in this document describes *what* to build and *why*; implementation details are yours to decide.

---

## 1. Product Summary

WarrantyVault is a consumer app that stores purchase proof (receipts, invoices, warranty cards), automatically tracks warranty periods, and — its key differentiator — **actively helps the user fight warranty and service claims** when a product breaks. It drafts claim messages, locates the correct service channel, tracks the claim's progress, and escalates with consumer-protection-grade language if the brand stalls.

**One-line positioning:** "Never lose a receipt, never lose a claim."

**Core insight:** Every existing receipt app is a passive filing cabinet. Users don't need storage — they need an advocate. The product's personality is *persistent, organized, slightly relentless on the user's behalf*.

## 2. Target Users

**Primary persona — "The Household Manager":** An adult (25–55) who buys appliances, electronics, and furniture for the home, loses paper receipts, and gives up on warranty claims because the process is exhausting. Moderate tech comfort. Lives primarily on their phone; heavy WhatsApp user.

**Secondary persona — "The Small Business Owner":** Buys equipment (printers, ACs, kitchen equipment, tools) and needs warranty tracking across many items, but has no system beyond a drawer of papers.

Design all flows for the primary persona; the secondary persona is served by the same features with higher item volume.

## 3. Problem Statement

1. Receipts are lost, faded (thermal paper), or buried in email.
2. Users don't know when warranties expire, so they miss free-repair windows.
3. When something breaks, the claims process is deliberately high-friction: finding the right service center, writing complaints, following up repeatedly. Most users abandon claims worth real money.
4. When brands stall, users don't know their consumer rights or how to escalate credibly.

## 4. Product Principles

1. **Capture must take under 10 seconds.** If saving a receipt is slower than tossing it in a drawer, the product fails.
2. **The app works even if the user does nothing after capture.** All value (expiry alerts, claim readiness) flows automatically from a single capture action.
3. **Claims are guided, never DIY.** The user should feel like they hired someone, not like they downloaded a form-filler.
4. **Trust through transparency.** Users are storing financial documents; show them exactly what is stored, extracted, and why.

---

## 5. Phase 1 — Core Product (The Vault)

### 5.1 Receipt Capture
Users add a purchase in any of these ways:
- **Photo capture:** Snap a photo of a paper receipt, invoice, or warranty card. Support multi-page capture (long receipts, invoice + warranty card as one purchase record).
- **File upload:** PDF or image from the phone's files (for emailed invoices the user downloads).
- **Email forwarding:** Each user gets a unique forwarding address (e.g., username@inbox.warrantyvault.app). Forwarded purchase emails are parsed and filed automatically.
- **Manual entry:** A minimal form for purchases with no document (fields: product name, brand, purchase date, price, warranty length, seller).

### 5.2 Automatic Extraction (AI)
On capture, extract and store as structured data:
- Product name and brand
- Seller / store name
- Purchase date
- Price paid
- Warranty duration if stated on the document; otherwise infer a *suggested* warranty from a lookup of typical durations by product category and brand, clearly labeled as "estimated — tap to confirm"
- Invoice/receipt number
- Product category (electronics, appliance, furniture, vehicle, etc.)

Show the extracted fields to the user for a one-tap confirm/edit step. Never silently trust extraction; the confirmation screen is also where warranty length is confirmed.

### 5.3 The Vault (Home Screen)
- List of all purchases, each showing: product name, brand, purchase date, and a **warranty status chip** (Active — X months left / Expiring soon / Expired / Unknown).
- Sort and filter: by expiry date (default: soonest-expiring first), category, brand, price.
- Search across product names, brands, sellers.
- Tapping an item opens the **Item Detail** screen: original document images, extracted data, warranty timeline bar, and action buttons ("Something broke — start a claim", "Add document", "Edit details").

### 5.4 Warranty Lifecycle Notifications
- "Warranty expiring in 30 days" and "expiring in 7 days" notifications, with a nudge: "If anything's wrong with this product, claim now — repairs are free until [date]."
- "Extended warranty decision" notification near expiry for high-value items: a neutral explainer of whether extended warranty is typically worth it for that category (no upsell in Phase 1).
- A weekly "vault digest" (opt-in): what's newly expiring, any open claims.

### 5.5 Household Sharing (simple version)
A user can invite one or more family members to a shared vault so either spouse can capture receipts and both see everything. Keep roles simple: all members have equal rights in Phase 1.

---

## 6. Phase 2 — The Claims Engine (The Differentiator)

### 6.1 Starting a Claim
From any item, the user taps "Something broke." A short guided intake:
1. What's wrong? (free text or voice note; AI summarizes into a defect description)
2. When did it start?
3. Optional photos/video of the defect.

The app then assembles a **Claim File**: purchase proof + warranty status + defect description + defect evidence, all in one packet.

### 6.2 Claim Routing
Using the brand and product category, the app tells the user the correct claim channel and its details: brand's official support email, customer-care number, service-center locator, or seller-first policies (some warranties require going through the retailer). Maintain a growing internal knowledge base of brand support channels; when unknown, the app searches and confirms with the user before saving to the knowledge base.

### 6.3 Drafted Communications
The app generates ready-to-send messages at each stage, in the user's chosen tone (polite / firm) and language:
- **Initial claim:** references invoice number, purchase date, warranty terms, defect description; attaches the claim file.
- **Follow-up:** if no response within a user-visible timer (default 7 days), the app nudges the user with a pre-drafted follow-up referencing the earlier message and date.
- **Escalation:** if stalling continues, drafts escalate in seriousness: request for written denial → notice referencing applicable consumer-protection law and the intent to file a complaint with the consumer forum → a ready-to-file consumer complaint document with the full evidence timeline attached. Legal references must be region-appropriate (build with India's Consumer Protection Act as the first supported jurisdiction, structured so other jurisdictions can be added).

The user always sends messages themselves (from their email/phone); the app prepares everything and tracks the timeline. Do not send on the user's behalf in this phase.

### 6.4 Claim Tracker
Each claim has a status timeline: Drafted → Sent → Awaiting response → In service → Resolved / Denied / Escalated. The user updates status with one tap when things happen ("They replied", "Picked up for repair", "Got it back"). Every update is timestamped — this timeline itself becomes evidence if escalation is needed.

### 6.5 Outcome Capture
When a claim resolves, ask: repaired / replaced / refunded / denied, and the value recovered. Show the user a running "Money WarrantyVault helped you recover" total — this number is the retention engine and the centerpiece of the app's home screen once nonzero.

---

## 7. Optional Module — WhatsApp Integration

**Goal:** Users already receive and forward documents on WhatsApp constantly. Let a user send any receipt, invoice, or warranty document to WarrantyVault's WhatsApp number, and have it **automatically registered to their account** — so the item appears in their vault the next time they open the app, with zero extra steps.

This module is optional: the app must be fully functional without it, and it should be built as a cleanly separable component.

### 7.1 Account Linking
- The user links WhatsApp from the app's settings: the app deep-links into a WhatsApp chat with the product's business number, pre-filled with a one-time linking code. Sending that message binds that WhatsApp phone number to the user's account.
- Alternative linking path: the user enters their phone number in the app and confirms via a code sent to them on WhatsApp.
- One WhatsApp number maps to one account (the account may be a shared household vault). Allow unlinking from settings.

### 7.2 Inbound Document Flow
When a linked user sends a message to the WarrantyVault WhatsApp number:
- **Image or PDF received:** run the same extraction pipeline as in-app capture. Reply in the chat with a concise confirmation: product name, brand, purchase date, detected warranty, and a note like "Saved to your vault ✅ — reply EDIT if anything looks wrong."
- **Multiple images in sequence:** treat images sent within a short window as candidate pages of one purchase; ask a single clarifying question ("Are these one purchase? Reply 1 for yes, 2 to save separately.").
- **Text-only message:** support quick manual entry via chat (e.g., user types "Bought Samsung fridge yesterday, 45,000, 2 year warranty") — parse and register it, then confirm.
- **Unlinked sender:** reply with a friendly one-line explanation and the link/steps to connect their account. Do not store documents from unlinked numbers.

### 7.3 Sync Guarantees
- Anything registered via WhatsApp must appear in the app's vault with a source label ("via WhatsApp") and the original file preserved, exactly as if captured in-app.
- If extraction fails or the document is unreadable, still save the raw file to the vault in a "Needs review" state and tell the user in the chat that it's saved but needs a quick check in the app.
- Handle duplicates: if the same document is sent twice (or sent on WhatsApp after being captured in-app), detect and merge rather than creating a second item; confirm the merge in the reply.

### 7.4 Outbound Notifications on WhatsApp (opt-in)
If the user opts in, deliver warranty-expiry alerts and claim follow-up reminders on WhatsApp instead of (or in addition to) push notifications. Respect messaging-platform rules around business-initiated messages and user consent; make opt-out a single reply ("STOP").

### 7.5 Boundaries of This Module
- WhatsApp is a capture and notification channel only. Claims are managed in the app (a chat thread is a poor interface for a multi-step claim); the WhatsApp bot may link into the relevant claim screen.
- Never initiate marketing messages through this channel.

---

## 8. Onboarding Flow

1. Value promise screen: "Snap it once. We'll remember the warranty and fight your claims."
2. Immediate capture: onboarding's second screen is the camera — ask the user to capture their most recent receipt right away. First-run value beats a signup form.
3. Account creation after first capture (email or phone-based).
4. Prompt to forward one purchase email to their unique address, and (if the module is built) to link WhatsApp.
5. Ask notification permission *at the moment* the first warranty is registered, framed as "Want us to warn you before this warranty expires?"

## 9. Monetization (design hooks now, charge later)

- **Free tier:** unlimited captures, warranty tracking, expiry alerts, and one active claim at a time.
- **Pro tier (paid):** unlimited simultaneous claims, escalation-grade legal drafting, household sharing beyond 2 members, WhatsApp channel, and priority extraction review.
- Future revenue experiments (do not build yet, but don't architect them out): affiliate extended-warranty offers presented neutrally; a "done-for-you claims" concierge taking a percentage of recovered value; a small-business tier with bulk capture and asset reports.

## 10. Trust, Privacy, and Safety Requirements

- All stored documents are private to the account; sharing only via explicit household invites.
- The user can export their entire vault (documents + structured data) and delete their account with full data removal.
- Extraction confidence must be surfaced honestly ("estimated" labels); never fabricate warranty terms.
- Legal-escalation drafts must include a visible disclaimer that the app provides document preparation and general information, not legal advice.
- Log every claim-timeline event immutably from the user's perspective (their evidence must be tamper-evident and exportable as a single PDF dossier).

## 11. Success Metrics

- **Activation:** % of new users who capture ≥1 receipt in their first session (target: >60%).
- **Habit:** median receipts captured per user per month (target: ≥3 by month 2).
- **Core value:** % of expiring warranties where the user viewed or acted on the expiry alert.
- **Differentiator:** number of claims started, claim resolution rate, and total value recovered per user.
- **WhatsApp module (if built):** % of linked users, and share of captures arriving via WhatsApp (expect this channel to dominate once linked).

## 12. Explicit Non-Goals (v1)

- No expense tracking, budgeting, or spending analytics — this is not a finance app.
- No automatic sending of emails/messages on the user's behalf.
- No marketplace, price comparison, or shopping features.
- No support for jurisdictions beyond the first (India) in the escalation legal content — but structure the content so jurisdictions are pluggable.

## 13. Suggested Build Order

1. Capture (photo, upload, manual) → extraction → confirmation screen → vault list and item detail.
2. Warranty status logic and expiry notifications.
3. Email-forwarding capture.
4. Claims Engine: intake → claim file → drafted initial message → claim tracker → follow-ups → escalation drafts.
5. Household sharing.
6. WhatsApp module: linking → inbound capture → confirmations → duplicate handling → opt-in notifications.
7. Monetization gates and Pro tier.
