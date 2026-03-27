# 14 — Common Errors

> **Reading time:** 12 minutes  
> **Previous:** [13 — UI](13-ui.md)  
> **Next:** [15 — Quick Reference](15-quick-reference.md)

---

## Error: offerings: 0, entitlements: 0

**What it looks like:**
```
RC init — offerings: 0, entitlements: 0
```

**What it means:** The SDK initialized but found nothing. This is almost always a dashboard configuration issue.

**Work through this checklist in order:**
1. Did you set an offering as **Current** in the RC dashboard? (The ★ star — this is the most common cause)
2. Do the product IDs in RC dashboard exist in App Store Connect / Google Play Console?
3. Is your iOS bundle ID in Xcode an exact match to the RC dashboard? (check for hyphens vs underscores)
4. On Android: is your app published to at least the **Internal Testing** track?
5. On Android: does the RC dashboard show the correct package name for your Android app?

---

## Error: CONFIGURATION_ERROR (code 23) — iOS

**What it looks like:**
```
RC raw fetch error: PlatformException(23,
"None of the products registered in the RevenueCat dashboard could be fetched
from App Store Connect"
```

**What it means:** The SDK is configured and talking to RC, but StoreKit cannot find the products. This is iOS-only.

**Causes:**
- The In-App Purchase capability is missing in Xcode — the most common cause. Open Xcode → Runner target → Signing & Capabilities → check for the In-App Purchase section
- Your bundle ID in Xcode does not match App Store Connect (check for `com.company.app-name` vs `com.company.app_name`)
- The products in App Store Connect are in "Missing Metadata" or "Waiting for Review" status (must be "Ready to Submit")
- The products were created under a different app in App Store Connect

---

## Error: PRODUCT_NOT_FOUND (statusCode=3) — Android

**What it looks like:**
```
Missing productDetails: UnfetchedProduct{productId='base_monthly', statusCode=3}
Product not found: base_monthly — Reason: PRODUCT_NOT_FOUND
```

**What it means:** Google Play returned the offerings from RC but could not find the products in your Play Store app.

**Causes:**
- The product IDs in RC dashboard do not exist in Google Play Console under your app's package name
- Your app has never been published (even to Internal Testing) — Google Play requires a published app for billing to work
- The RC dashboard has the wrong package name registered for your Android app

---

## Error: Empty user ID / anonymous user

**What it looks like:**
```
⚠️ Identifying with empty App User ID will be treated as anonymous.
👤 Setting new anonymous App User ID - $RCAnonymousID:510585fdb5...
```

**What it means:** You called `InitializeRC("")` — passing an empty string as the user ID.

**Cause:** You are initializing RC in `CheckAuthStatusEvent` or `_onSignIn`, where the user object may only have fields parsed from a JWT token. The `id` field may be empty at that point.

**Fix:** Initialize RC in `AccountBloc._onFetchProfile()`, after your `/users/me` API call returns the complete user profile with the confirmed UUID. See [12 — Initialization](12-initialization.md).

---

## Error: GetIt — Object with name 'rcApiKey' not registered

**What it looks like:**
```
Bad state: GetIt: Object/factory with name rcApiKey and type String is not registered
```

**What it means:** The Injectable code generator did not register your `rcApiKey` getter, so GetIt cannot inject it.

**Causes and fixes:**
1. You added `@lazySingleton` to the String getter — remove it, use only `@Named`
2. You forgot to re-run the code generator after changing `register_module.dart`
3. The code generator silently skipped the file due to a parse error

**Fix:**
```bash
flutter clean
dart run build_runner clean
dart run build_runner build --delete-conflicting-outputs
```

Then open `lib/core/di/injector.config.dart` and search for `rcApiKey`. Confirm it exists:
```dart
gh.registerLazySingleton<String>(() => registerModule.rcApiKey, instanceName: 'rcApiKey');
```

---

## Error: Runtime type cast — Future.wait

**What it looks like:**
```
type '() => List<dynamic>' is not a subtype of type '() => List<RCPackage>' of 'dflt'
```

**What it means:** You used `Future.wait` to run multiple typed futures in parallel. Dart erases the generic types when collecting results into a `List<dynamic>`, causing a runtime crash when you call `getOrElse`.

**Wrong:**
```dart
final results = await Future.wait([
  _getOfferings(NoParams()),
  _getEntitlements(NoParams()),
]);
final offerings = results[0].getOrElse(() => []); // CRASH
```

**Correct:**
```dart
final Either<Failure, List<RCPackage>> offeringsResult =
    await _getOfferings(NoParams());
final Either<Failure, Map<String, RCEntitlement>> entitlementsResult =
    await _getEntitlements(NoParams());

final offerings = offeringsResult.getOrElse(() => <RCPackage>[]);
final entitlements = entitlementsResult.getOrElse(() => <String, RCEntitlement>{});
```

---

## Error: purchasePackage — 'entitlements' isn't defined for PurchaseResult

**What it looks like:**
```
The getter 'entitlements' isn't defined for the type 'PurchaseResult'
```

**What it means:** In `purchases_flutter` v8+, `Purchases.purchasePackage()` returns a `PurchaseResult` object, not a `CustomerInfo` object directly.

**Wrong:**
```dart
final customerInfo = await rc.Purchases.purchasePackage(pkg);
customerInfo.entitlements.active; // error
```

**Correct:**
```dart
final result = await rc.Purchases.purchasePackage(pkg);
result.customerInfo.entitlements.active; // correct
```

---

## Error: PackageType naming collision

**What it looks like:**
```
The argument type 'PackageType (defined in purchases_flutter)' can't be assigned
to the parameter type 'PackageType (defined in rc_package_entity.dart)'
```

**What it means:** Both your entity file and the SDK export an enum named `PackageType`. Dart sees two different types with the same name.

**Fix:** Two steps:
1. Rename your entity enum to `RCPackageType` (not `PackageType`)
2. Import the SDK with an alias: `import 'package:purchases_flutter/purchases_flutter.dart' as rc;`

Now `rc.PackageType` is the SDK's enum and `RCPackageType` is yours. No collision.

---

## Error: Purchases instance already set

**What it looks like:**
```
Purchases instance already set. Did you mean to configure two Purchases objects?
```

**What it means:** `rc.Purchases.configure()` was called more than once. RC only needs to be initialized once per app session.

**Fix:** Use the `_rcInitialized` flag in your BLoC:
```dart
bool _rcInitialized = false;

Future<void> _onInitialize(...) async {
  // ...
  _rcInitialized = true; // set this BEFORE anything else
  // ...
}
```

And in every handler:
```dart
Future<void> _onLoadOfferings(...) async {
  if (!_rcInitialized) return; // exit early if RC not ready
  // ...
}
```

Also guard in AccountBloc:
```dart
if (_subscriptionsBloc.state.status == SubscriptionStatus.initial) {
  _subscriptionsBloc.add(InitializeRC(profile.id));
}
// Only fire if status is initial — prevents double initialization
```

---

**Next:** [15 — Quick Reference →](15-quick-reference.md)
