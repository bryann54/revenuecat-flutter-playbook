# 02 — Key Concepts

> **Reading time:** 8 minutes  
> **Previous:** [01 — What is RevenueCat](01-what-is-revenuecat.md)  
> **Next:** [03 — Dashboard Setup](03-dashboard-setup.md)

---

## The 4 words you must know

Everything in RevenueCat revolves around 4 concepts. If you understand these, the rest of the guide makes sense automatically.

---

### 1. Product

**What it is:** A subscription that exists inside Apple's or Google's system.

**Real-world analogy:** A physical item on a restaurant menu — it has a name, a price, and it lives in the kitchen's inventory.

**Where you create it:** App Store Connect (for iOS) or Google Play Console (for Android). NOT in RevenueCat.

**Examples:**
- `com.yourapp.premium.monthly` — $4.99/month
- `com.yourapp.premium.annual` — $39.99/year
- `com.yourapp.premium.lifetime` — $99.99 one-time

**Key rule:** The product ID must match exactly between the store (App Store / Play Store) and the RevenueCat dashboard. A typo means nothing shows up and you get no error message.

---

### 2. Offering

**What it is:** A group of products you want to show users at the same time.

**Real-world analogy:** Today's menu. The kitchen has hundreds of ingredients, but today the chef decided to serve three specific dishes.

**Where you create it:** RevenueCat dashboard — NOT in the stores.

**Why it exists:** It lets you change what plans you show users without updating your app. You just change the offering in the dashboard. On Monday you could show monthly + annual. On Tuesday you could run a promotion and show monthly + lifetime. Your app code stays the same.

**The "Current" offering:** You can have many offerings but only one is the "current" (default) one. Your app fetches the current offering. If you forget to mark one as current, your app gets `null` and shows nothing.

---

### 3. Package

**What it is:** A single product inside an offering. Each package has a type: monthly, annual, weekly, or lifetime.

**Real-world analogy:** One dish on today's menu. The menu (offering) might have 3 dishes (packages), and each dish (package) maps to a specific item in the kitchen's inventory (product).

**Reserved identifiers (use these):**
| Package identifier | Meaning |
|---|---|
| `$rc_monthly` | Billed every month |
| `$rc_annual` | Billed every year |
| `$rc_weekly` | Billed every week |
| `$rc_lifetime` | One-time purchase, never expires |

Using these reserved identifiers is important — RevenueCat uses them to understand your pricing for analytics.

---

### 4. Entitlement

**What it is:** Proof that a user has paid and is allowed to access specific features.

**Real-world analogy:** A VIP wristband at an event. The staff do not care HOW you got in (bought a ticket, won a contest, got a free pass) — they just check if you have the wristband. If yes, you get in.

**Where you create it:** RevenueCat dashboard.

**How it works in code:** After a user subscribes, RevenueCat attaches an entitlement to their account. Your app checks for it:

```dart
final isPremium = customerInfo.entitlements.active.containsKey('premium');
```

The string `'premium'` is the identifier you choose when you create the entitlement. It can be anything — just be consistent.

**Why entitlements are powerful:**
- A user might have `monthly`, `annual`, or `lifetime` subscriptions
- They might have gotten premium through a promo code or a gift
- They might be on a free trial
- You do not need to check all of these — you just check `entitlements['premium'].isActive`
- RevenueCat figures out whether they qualify, regardless of how they got there

---

## How they connect

Here is the relationship in plain English:

```
You create a Product in App Store Connect / Google Play Console.

You import that Product into RevenueCat.

You create an Entitlement in RevenueCat and attach the Product to it.
  → This means "anyone who buys this product gets this entitlement"

You create an Offering in RevenueCat.
You add Packages to the Offering, each Package linked to a Product.
  → This is what your app will show on the paywall screen

You set one Offering as Current.
  → This is the one your app fetches by default

When a user buys a Package:
  → The Product is purchased from the store
  → RevenueCat activates the Entitlement for that user
  → Your app checks the Entitlement and unlocks premium features
```

---

## The setup order matters

You must do these in this exact order. Doing them out of order is the most common source of confusion:

```
Step 1: Create Products in App Store Connect / Play Console
          ↓
Step 2: Import Products into RevenueCat dashboard
          ↓
Step 3: Create an Entitlement and attach the Products
          ↓
Step 4: Create an Offering and add Packages
          ↓
Step 5: Set the Offering as Current (the ★ star)
          ↓
Step 6: Write your Flutter code
```

> ⚠️ If you write Flutter code before completing Step 5, you will spend hours wondering why `offerings.current` is always null. Ask us how we know.

---

## Quick summary table

| Concept | Lives in | Created by | Purpose |
|---|---|---|---|
| Product | App Store / Play Store | You (the developer) | The actual thing the user buys |
| Offering | RevenueCat dashboard | You (the developer) | Groups products to show at once |
| Package | RevenueCat dashboard | You (the developer) | One product within an offering |
| Entitlement | RevenueCat dashboard | You (the developer) | What the user gets access to |

---

**Next:** [03 — Dashboard Setup →](03-dashboard-setup.md)
