# 01 — What is RevenueCat?

> **Reading time:** 5 minutes  
> **Next:** [02 — Key Concepts](02-key-concepts.md)

---

## The problem it solves

Say you built a great app and you want to charge $4.99/month for premium features.

Sounds simple. Here is what you would have to do **without** RevenueCat:

1. Learn Apple's StoreKit framework (complex, poorly documented)
2. Learn Google's Billing Library (completely different API, different quirks)
3. Build your own backend server to track who paid for what
4. Write receipt validation logic (Apple and Google do this differently)
5. Handle subscription renewals, cancellations, refunds, upgrades, downgrades
6. Handle edge cases: expired cards, family sharing, free trials, promo codes
7. Keep all of this working as Apple and Google change their APIs every year

That is months of work. It breaks constantly. It is a full-time job.

**RevenueCat does all of that for you.** You call three methods:

```dart
// Get available plans
final offerings = await Purchases.getOfferings();

// Buy a plan
await Purchases.purchasePackage(package);

// Check if user is premium
final customerInfo = await Purchases.getCustomerInfo();
final isPremium = customerInfo.entitlements.active.containsKey('premium');
```

RevenueCat handles everything else in the background.

---

## The mental model — think of a restaurant

This analogy will stick with you throughout the guide:

```
Your app (customer)
    ↓  "What's on the menu?"
RevenueCat SDK (the waiter)
    ↓  fetches and translates
App Store / Play Store (the kitchen)
    ↓  returns products and prices
RevenueCat SDK
    ↑  gives you clean data
Your app shows the plans to the user
```

When the user buys something:

```
Your app: "I'll have the monthly plan"
RevenueCat: processes with Apple or Google
Apple / Google: charges the user's card
RevenueCat: gives your app a receipt (called an Entitlement)
Your app: unlocks premium features
```

---

## What RevenueCat does NOT do

It is important to understand the boundaries:

| RevenueCat does | RevenueCat does NOT do |
|---|---|
| Process payments | Create products (you do that in the stores) |
| Track subscriptions | Build your paywall UI |
| Validate receipts | Handle user authentication |
| Sync across platforms | Store your app's data |
| Give you a revenue dashboard | Replace your backend entirely |

---

## How RevenueCat makes money

RevenueCat is free up to $2,500 monthly tracked revenue. After that they take a small percentage. They are not in the payments flow — Apple and Google still pay you directly. RevenueCat just charges for their tracking service.

---

## What you will build

By the end of this guide, your app will:

1. Show a paywall with your subscription plans and their prices (fetched live from the stores)
2. Let users subscribe with one tap (the native App Store / Play Store payment sheet appears)
3. Check if a user is premium anywhere in the app with `state.isPremium`
4. Restore purchases when a user reinstalls the app
5. Reset correctly when the user logs out

---

## Before you continue

You need:
- [ ] A RevenueCat account — [sign up free at app.revenuecat.com](https://app.revenuecat.com)
- [ ] An Apple Developer account (to create iOS products)
- [ ] A Google Play Developer account (to create Android products)
- [ ] A Flutter project to add this to

---

**Next:** [02 — Key Concepts →](02-key-concepts.md)
