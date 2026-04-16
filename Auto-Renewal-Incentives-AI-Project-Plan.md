# Auto Renewal Incentives — Feature Documentation

---

## What We Built

Members who stay on **auto-renewal** are automatically rewarded with **bonus home points** when their membership renews. This incentivises members to remain on auto-renewal rather than switching to manual renewal, improving retention and reducing churn.

---

## How It Works

**The core loop is simple:**

1. An admin sets the bonus home points value per tier via the Admin Tools
2. When a member's membership auto-renews, they receive those bonus points as a credit
3. Members who renew manually do **not** get the bonus
4. Members are informed of this benefit, creating a reason to stay on auto-renewal

---

## User Flows

### Flow 1 — Admin Sets the Bonus

```
Admin logs into Admin Tools
         │
         ▼
Navigates to Marketing CMS → Site Config
         │
         ▼
Sees "Auto Renewal Bonus Home Points" field
  ┌──────────────────────────────────────────┐
  │  Auto Renewal Bonus Home Points          │
  │  ┌──────┐  [Save]  [Reset]              │
  │  │  5   │                                │
  │  └──────┘                                │
  │  "The number of bonus home points a      │
  │   member receives when their membership  │
  │   auto-renews. Set to 0 to disable."     │
  └──────────────────────────────────────────┘
         │
         ▼
Value saved to Redis, change is audit-logged
```

- Each tier (Standard and Premium Lite) has its own independent value
- Setting the value to **0** disables the bonus for that tier
- Every change records who made it and what the new value is

---

### Flow 2 — Member Auto-Renews and Gets Bonus Points

```
Member's renewal date arrives
         │
         ▼
Scheduled job runs daily (Hangfire)
         │
         ▼
Finds members due to renew today
         │
         ▼
Checks eligibility:
  ✓ Auto-renew is ON
  ✓ Membership expires today
  ✓ Account is active
  ✓ Not a PMG+ club member
         │
         ▼
Renews the finance agreement
         │
         ▼
Processes the renewal transaction
         │
         ▼
Transaction includes:
  ┌──────────────────────────────────────┐
  │  Payment (debit)                     │
  │  Home Points (credit) — from bundle  │
  │  Away Points (credit) — from bundle  │
  │  ★ Bonus Home Points (credit)        │  ◄── THE INCENTIVE
  │    e.g. +5 Home Points               │
  └──────────────────────────────────────┘
         │
         ▼
Member sees extra home points in their account
```

**Members who renew manually get the standard bundle points only — no bonus.**

---

### Flow 3 — Admin Disables Auto-Renew for a Member

```
Admin views a member's profile
         │
         ▼
Clicks "Disable Auto Renew"
         │
         ▼
Warning modal appears:
  ┌──────────────────────────────────────┐
  │  ⚠ Bonus Points Warning             │
  │                                      │
  │  Members who renew automatically     │
  │  receive bonus home points.          │
  │                                      │
  │  By disabling auto-renewal, this     │
  │  member will NOT receive those       │
  │  bonus points if they renew          │
  │  manually.                           │
  │                                      │
  │        [Cancel]  [Disable Auto Renew]│
  └──────────────────────────────────────┘
         │
         ▼
If confirmed:
  • DontAutoRenew = true
  • AutoRenewTurnedOffDate recorded
  • Audit log created
  • HubSpot CRM updated
```

---

### Flow 4 — Member Self-Service (Premium Portal)

```
Member logs into the Premium app
         │
         ▼
Goes to Manage Subscription
         │
         ├─── Auto-Renew is ON ──────────────────────┐
         │                                            │
         │  "This subscription will automatically     │
         │   renew on {date}. If you don't want       │
         │   this to happen, you can Disable           │
         │   Auto Renew."                              │
         │                                            │
         │  [Disable auto-renew]  ◄── red button      │
         │                                            │
         ├─── Auto-Renew is OFF ─────────────────────┐
         │                                            │
         │  "This subscription will not auto-renew,   │
         │   to continue after expiry you'll have     │
         │   to make a payment manually. You can      │
         │   turn auto-renew back on."                │
         │                                            │
         │  [Enable Auto-renew]                       │
         │                                            │
         └────────────────────────────────────────────┘
```

---

### Flow 5 — Welcome / Checkout Messaging

```
During checkout (for auto-renewing members):

  "Your membership will automatically renew
   for {price} on your renewal date.
   You can cancel online at any time."
```

This messaging reinforces that the member is on auto-renewal and sets expectations.

---

## What Gets Stored

| Where | What | Example |
|-------|------|---------|
| **Redis** | Bonus points per tier | `AutoRenewalBonusHomePoints = 5` |
| **Redis** | Premium Lite bonus | `IX_AutoRenewalBonusHomePoints = 3` |
| **SQL (AspNetUsers)** | Auto-renew flag | `DontAutoRenew = false` |
| **SQL (AspNetUsers)** | Turn-off timestamp | `AutoRenewTurnedOffDate = 2026-03-15` |
| **HubSpot CRM** | Auto-renew status | `auto_renew_enabled = true` |
| **HubSpot CRM** | Turn-off date | `auto_renew_turned_off_date = 2026-03-15` |
| **Audit Log** | Config changes | Admin X set bonus to 5 for Standard tier |
| **Audit Log** | Auto-renew toggles | Admin Y disabled auto-renew for Member Z |

---

## Where It Lives in the Codebase

### Backend (.NET — MembersWebAzure)

| Component | File | What It Does |
|-----------|------|--------------|
| API | `SiteConfigController.cs` | GET/SET bonus points endpoints |
| API | `MemberAdminController.cs` | Disable auto-renew endpoint |
| Service | `SiteConfigService.cs` | Reads/writes bonus + creates audit log |
| Service | `PurchaseService.cs` | Adds bonus points line to renewal transaction |
| Service | `MonthlyPayAutoRenewalService.cs` | Daily job that processes auto-renewals |
| Service | `CachingMemberService.cs` | Toggles auto-renew flag + updates DB |
| Service | `PaymentLifecycleService.cs` | Handles payment events, triggers purchase |
| Service | `HubSpotService.cs` | Syncs auto-renew status to CRM |
| Store | `SiteConfigStore.cs` | Redis read/write for bonus values |
| Model | `AlterAutoRenewalBonusPointsAuditData.cs` | Audit record shape |
| Migration | `20260331120000_AddAutoRenewTurnedOffDate.cs` | Added turn-off date column |

### Frontend — Admin Tools (Nuxt 2)

| Component | File | What It Does |
|-----------|------|--------------|
| Page | `siteConfig/index.vue` | Bonus points config UI |
| Modal | `DisableAutoRenewConfirmModal.vue` | Warning when disabling A/R |

### Frontend — Premium Member App (Vue 2)

| Component | File | What It Does |
|-----------|------|--------------|
| Page | `manageSubscription.vue` | Shows renewal status + toggle |
| Button | `enableAutorenew.vue` | Re-enables auto-renewal |
| Button | `disableAutorenew.vue` | Disables auto-renewal |
| Store | `userStore.js` | Tracks `autoRenewOn` state |

### Shared JS Library (pmg-services)

| Component | File | What It Does |
|-----------|------|--------------|
| Service | `siteconfigService.js` | `getAutoRenewalBonusHomePoints(tier)` / `setAutoRenewalBonusHomePoints(amount, tier)` |
| Constants | `apiConstants.js` | API route definitions |

---

## Key Business Rules

| Rule | Detail |
|------|--------|
| Auto-renew only | Bonus points are **only** credited on auto-renewal transactions — never on manual renewals |
| Per-tier config | Standard and Premium Lite have separate bonus values |
| Zero = off | Setting the bonus to 0 disables it — no bonus line is added to the transaction |
| PMG+ excluded | Members at PMG+ clubs are excluded from auto-renewal entirely |
| Inactive excluded | Locked-out or inactive accounts are skipped during renewal |
| System attribution | Auto-renewal transactions are attributed to a system user, not the member |
| Promo codes stack | Tier-specific renewal promo codes apply alongside the bonus (not instead of) |
| CRM visibility | Auto-renew status is pushed to HubSpot so marketing can segment and target accordingly |

---

## End-to-End Summary

```
┌──────────┐     ┌──────────────┐     ┌────────────────┐     ┌──────────────┐
│  Admin   │────▶│  Sets bonus  │────▶│  Stored in     │────▶│  Applied at  │
│  Tools   │     │  per tier    │     │  Redis         │     │  renewal     │
└──────────┘     └──────────────┘     └────────────────┘     └───────┬──────┘
                                                                     │
                                                                     ▼
┌──────────┐     ┌──────────────┐     ┌────────────────┐     ┌──────────────┐
│  HubSpot │◀────│  CRM sync    │◀────│  Member gets   │◀────│  Bonus home  │
│  CRM     │     │  A/R status  │     │  extra points  │     │  points      │
└──────────┘     └──────────────┘     └────────────────┘     │  credited    │
                                                              └──────────────┘
```

**Admin configures → Redis stores → Renewal job triggers → Bonus applied → Member rewarded → CRM updated**

---

*PlayMoreGolf Platform — Auto Renewal Incentives Feature*
*Last updated: April 2026*
