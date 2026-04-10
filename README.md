# Billing & payment options — team reference
 
**Purpose:** Align product, finance, and engineering on **how customers pay** and **how workspace access (`BillingSubscription`) stays correct**.
 
**Audience:** Product, finance, engineering leads.
 
---
 
## 1. Three payment models (at a glance)
 
| # | Model | Who initiates | Where money is recorded | How the app knows they paid |
|---|--------|----------------|-------------------------|-----------------------------|
| **1** | **Self-serve Stripe subscription** | Customer in-app | Stripe (subscription + invoices) | Webhooks → MongoDB `BillingSubscription` |
| **2A** | **Manual / offline entitlement** | Finance / ops after wire, PO, contract | Outside Stripe (bank, etc.) | **Manual update** to MongoDB (or internal admin tool) |
| **2B** | **Stripe Invoice (no self-serve checkout)** | Finance / ops in Stripe | Stripe (invoice + payment) | **`invoice.paid` webhook** → MongoDB (requires metadata; see §5) |
 
**Note:** “2A” and “2B” are both **exceptions** when subscription checkout is not used. Choose based on whether payment must appear in Stripe or not.
 
---
 
## 2. Model 1 — Self-serve Stripe subscription (default SaaS path)
 
**Fit:** Standard customers who can pay by card through the product.
 
**Flow (simplified):**
 
1. User starts checkout from **Settings → Billing** (`/api/billing/create-checkout-session`).
2. Stripe Checkout collects payment and creates a **Subscription**.
3. Stripe sends webhooks (e.g. `checkout.session.completed`, `customer.subscription.updated`).
4. Backend syncs **`BillingSubscription`** (plan, seats, period, Stripe IDs, status).
 
**Source of truth:**
 
- **Money & contract in Stripe:** Subscription and related invoices.
- **App access:** MongoDB `BillingSubscription`, kept in sync via webhooks and overview safety sync.
 
**Engineering notes:** Plan and price IDs come from server config (`lib/billing/plans.js`); Stripe customer is tied to `accountId`. Redirect URLs use configured app base URL (`APP_URL` / `NEXT_PUBLIC_APP_URL`), not the browser `Origin` header.
 
---
 
## 3. Model 2A — Manual / offline entitlement
 
**Fit:** Enterprise deals, wire transfers, purchase orders, or any case where **payment does not go through Stripe** (or not yet).
 
**Flow:**
 
1. Finance confirms payment (or signed contract) through your normal process.
2. Ops updates **`BillingSubscription`** for that workspace’s `accountId`:
   - `planKey`, `seats`, `status` (e.g. `active`)
   - `currentPeriodStartAt` / `currentPeriodEndAt` (contract term)
   - Optional: `metadata` e.g. `billingSource: "manual"`, PO number, notes.
3. If the customer previously had a Stripe subscription you are replacing, cancel it in **Stripe Dashboard** and align DB fields (e.g. clear `stripeSubscriptionId` if you no longer use a subscription for them).
 
**Source of truth:**
 
- **Payment:** Your finance process + bank records (not Stripe).
- **App access:** MongoDB only for that entitlement path.
 
**Trade-off:** Stripe will **not** automatically mirror offline revenue; reporting must combine finance data + optional manual tags in CRM/ops.
 
---
 
## 4. Model 2B — Stripe Invoice (money in Stripe, not subscription checkout)
 
**Fit:** Customer should pay **through Stripe** (invoice, card/ACH on invoice) but **not** through the in-app subscription checkout (e.g. negotiated deal, annual lump sum, procurement prefers invoice).
 
**Flow:**
 
1. Ensure a **Stripe Customer** exists and carries **`metadata.accountId`** (and ideally workspace identifiers) so Stripe objects map to the correct tenant.
2. Create a **Stripe Invoice** (Dashboard or API) with line items for the deal.
3. Add **metadata the webhook can read**, e.g. on the invoice: `accountId`, `planKey`, `seats`, optional period end or duration.
4. Customer pays; Stripe fires **`invoice.paid`**.
5. Backend webhook handler updates **`BillingSubscription`** for that `accountId` (same logical fields as 2A: plan, seats, dates, `stripeCustomerId`, `lastInvoiceId`).
 
**Source of truth:**
 
- **Money:** Stripe (invoice + payment).
- **App access:** MongoDB, updated when **`invoice.paid`** runs (not when the user lands on a “success” page).
 
**Engineering note (current codebase):** The existing `invoice.paid` handler is primarily used for **subscription-related** behavior (e.g. scheduled downgrades). **Standalone invoice entitlements** need an explicit branch: read `accountId` / plan / seats from **invoice or customer metadata** and `$set` on `BillingSubscription`. Plan this as a small follow-up if 2B is adopted.
 
---
 
## 5. Consistency rules (all models)
 
1. **Never rely on the browser alone** to write billing state; prefer **webhooks** (Stripe) or **controlled admin updates** (manual).
2. **Always key by `accountId`** when connecting Stripe objects to MongoDB (metadata on Customer / Invoice / Subscription).
3. **Avoid two conflicting writers** for the same account (e.g. active subscription + conflicting manual row) without a clear rule (e.g. “invoice deal replaces subscription”).
 
---
 
## 6. Decision guide for sales / ops
 
| Situation | Recommended model |
|-----------|-------------------|
| Standard SMB / self-serve | **1 — Subscription checkout** |
| Wire / PO / no Stripe | **2A — Manual DB** (or move to 2B later if you invoice through Stripe) |
| Must show revenue in Stripe, no app checkout | **2B — Stripe Invoice + webhook** |
 
---
 
*Document version: internal draft for team discussion. Update when product policy or implementation changes.*
