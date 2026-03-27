# 15 — Quick Reference

> **Previous:** [14 — Common Errors](14-common-errors.md)  
> **Start over:** [README](../README.md)

---

## The full lifecycle in one place

```
App starts
  → AuthBloc: CheckAuthStatusEvent
  → User authenticated (shell object, may have empty ID)
  → App navigates to home

Home screen loads
  → AccountBloc: FetchProfileEvent
  → GET /users/me → returns real user UUID

Profile loaded
  → SubscriptionsBloc: InitializeRC(profile.id)  ← real UUID here
  → rc.Purchases.configure(apiKey, appUserID: userId)
  → getOfferings() → packages from RC dashboard
  → getEntitlements() → what user currently has active
  → status: ready

User taps Subscribe
  → SubscriptionsBloc: PurchasePackage(pkg)
  → App Store / Play Store payment sheet appears
  → User pays
  → status: success, entitlements: { premium: { isActive: true } }
  → state.isPremium == true → unlock premium features

User signs out
  → AuthBloc: SignOutEvent
  → SubscriptionsBloc: RCLogOut()
  → rc.Purchases.logOut()
  → BLoC resets to initial state
```

---

## Key methods at a glance

```dart
// Check if user is premium:
context.read<SubscriptionsBloc>().state.isPremium

// Check in a builder:
BlocBuilder<SubscriptionsBloc, SubscriptionState>(
  builder: (context, state) => state.isPremium
      ? PremiumWidget()
      : UpgradeWidget(),
)

// Trigger a purchase:
context.read<SubscriptionsBloc>().add(PurchasePackage(pkg));

// Restore purchases:
context.read<SubscriptionsBloc>().add(RestorePurchases());

// Sign out (resets RC):
context.read<SubscriptionsBloc>().add(RCLogOut());

// Check a specific entitlement:
state.entitlements['premium']?.isActive == true
```

---

## Environment variables

```bash
# .env
PUBLIC_REVENUECAT_API_KEY_IOS=appl_xxxxxxxxxxxxxxxx
PUBLIC_REVENUECAT_API_KEY_ANDROID=goog_xxxxxxxxxxxxxxxx
```

---

## After any DI annotation change

```bash
dart run build_runner build --delete-conflicting-outputs
```

---

## After any native change (Podfile, AndroidManifest, Xcode settings)

```bash
flutter clean
flutter pub get
cd ios && pod install && cd ..
flutter run
```

Hot restart is not enough for native changes.

---

## Dashboard setup quick order

```
1. App Store Connect / Play Console → create products
2. RC Dashboard → Products → import product IDs
3. RC Dashboard → Entitlements → create "premium", attach products
4. RC Dashboard → Offerings → create "default", add packages, attach products
5. RC Dashboard → Offerings → click ★ star on "default" to set as Current
```

---

## Debugging checklist when offerings are empty

```
[ ] Offering set as Current in RC dashboard? (★ star)
[ ] Product IDs in RC match App Store Connect / Play Console exactly?
[ ] iOS bundle ID in Xcode matches RC dashboard? (hyphens vs underscores)
[ ] In-App Purchase capability added in Xcode?
[ ] Android app published to Internal Testing?
[ ] Android package name in RC dashboard matches build.gradle?
[ ] BILLING permission in AndroidManifest outside <application>?
[ ] minSdkVersion 21 in build.gradle?
[ ] Run flutter clean + pod install after native changes?
```

---

## The definitive sign everything works

```
RC init — offerings: 3, entitlements: 0
```

- `offerings: 3` means the SDK found 3 packages in the current offering. Plans will show.
- `entitlements: 0` means this user has not subscribed yet. That is correct and expected.

Once a user subscribes:
```
RC init — offerings: 3, entitlements: 1
{ "premium": RCEntitlement(isActive: true, expirationDate: 2026-12-31) }
```

---

*You made it. Go ship your subscriptions.*
