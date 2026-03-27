# 07 — Datasource

> **Reading time:** 12 minutes  
> **Previous:** [06 — Entities](06-entities.md)  
> **Next:** [08 — Repository](08-repository.md)

---

## The only file that touches the SDK

This is the most important file in the whole feature. Every `purchases_flutter` call happens here and nowhere else. The rest of your codebase never imports the RC SDK.

Notice the import alias: `import 'package:purchases_flutter/purchases_flutter.dart' as rc;`

The `as rc` alias means all SDK types are prefixed: `rc.Purchases`, `rc.PackageType`, `rc.EntitlementInfo`. This prevents the naming collision with your own `RCPackageType` enum.

---

## The abstract interface

```dart
// lib/features/subscriptions/data/datasources/rc_subscription_datasource.dart

abstract class RCSubscriptionDatasource {
  Future<void> initialize(String userId);
  Future<List<RCPackage>> getOfferings();
  Future<Map<String, RCEntitlement>> purchasePackage(RCPackage package);
  Future<Map<String, RCEntitlement>> restorePurchases();
  Future<Map<String, RCEntitlement>> getEntitlements();
  Future<void> logOut();
}
```

---

## The full implementation

```dart
import 'package:flutter/services.dart';
import 'package:injectable/injectable.dart';
import 'package:purchases_flutter/purchases_flutter.dart' as rc;
import 'package:legal_defender/core/errors/exceptions.dart';
import 'package:legal_defender/features/subscriptions/domain/entities/rc_entitlement_entity.dart';
import 'package:legal_defender/features/subscriptions/domain/entities/rc_package_entity.dart';

@LazySingleton(as: RCSubscriptionDatasource)
class RCSubscriptionDatasourceImpl implements RCSubscriptionDatasource {
  final String _apiKey;

  RCSubscriptionDatasourceImpl(@Named('rcApiKey') this._apiKey);

  @override
  Future<void> initialize(String userId) async {
    await rc.Purchases.setLogLevel(rc.LogLevel.debug);
    await rc.Purchases.configure(
      rc.PurchasesConfiguration(_apiKey)..appUserID = userId,
    );
  }

  @override
  Future<List<RCPackage>> getOfferings() async {
    try {
      final offerings = await rc.Purchases.getOfferings();
      final current = offerings.current;
      if (current == null) return [];
      return current.availablePackages.map(_mapPackage).toList();
    } on PlatformException catch (e) {
      _throwMapped(e);
    }
  }

  @override
  Future<Map<String, RCEntitlement>> purchasePackage(RCPackage package) async {
    try {
      final offerings = await rc.Purchases.getOfferings();
      final sdkPackage = offerings.current?.availablePackages.firstWhere(
        (p) => p.storeProduct.identifier == package.productIdentifier,
      );
      if (sdkPackage == null) throw NotFoundException('Package not found');

      // IMPORTANT: purchasePackage returns PurchaseResult in SDK v8+
      // You must access .customerInfo from the result — it is NOT CustomerInfo directly
      final result = await rc.Purchases.purchasePackage(sdkPackage);
      return _mapEntitlements(result.customerInfo.entitlements.active);
    } on PlatformException catch (e) {
      _throwMapped(e);
    }
    return {};
  }

  @override
  Future<Map<String, RCEntitlement>> restorePurchases() async {
    try {
      final customerInfo = await rc.Purchases.restorePurchases();
      return _mapEntitlements(customerInfo.entitlements.active);
    } on PlatformException catch (e) {
      _throwMapped(e);
    }
    return {};
  }

  @override
  Future<Map<String, RCEntitlement>> getEntitlements() async {
    try {
      final customerInfo = await rc.Purchases.getCustomerInfo();
      return _mapEntitlements(customerInfo.entitlements.active);
    } on PlatformException catch (e) {
      _throwMapped(e);
    }
    return {};
  }

  @override
  Future<void> logOut() async {
    try {
      await rc.Purchases.logOut();
    } on PlatformException catch (e) {
      _throwMapped(e);
    }
  }

  // ── Mappers ────────────────────────────────────────────────────────────────
  // These translate SDK types to your app's types.
  // Change the SDK → update these. Nothing else in your app needs to change.

  RCPackage _mapPackage(rc.Package p) {
    return RCPackage(
      identifier: p.identifier,
      packageType: _mapPackageType(p.packageType),
      productIdentifier: p.storeProduct.identifier,
      localizedPriceString: p.storeProduct.priceString,
      localizedTitle: p.storeProduct.title,
      localizedDescription: p.storeProduct.description,
    );
  }

  // The SDK has rc.PackageType, you have RCPackageType — different types, no collision
  RCPackageType _mapPackageType(rc.PackageType type) {
    switch (type) {
      case rc.PackageType.monthly:  return RCPackageType.monthly;
      case rc.PackageType.annual:   return RCPackageType.annual;
      case rc.PackageType.weekly:   return RCPackageType.weekly;
      case rc.PackageType.lifetime: return RCPackageType.lifetime;
      default:                      return RCPackageType.unknown;
    }
  }

  Map<String, RCEntitlement> _mapEntitlements(
    Map<String, rc.EntitlementInfo> active,
  ) {
    return active.map(
      (key, info) => MapEntry(
        key,
        RCEntitlement(
          identifier: info.identifier,
          isActive: info.isActive,
          expirationDate: info.expirationDate != null
              ? DateTime.parse(info.expirationDate!)
              : null,
          productIdentifier: info.productIdentifier,
          store: info.store.name,
        ),
      ),
    );
  }

  // Maps RevenueCat SDK errors to your app's exception types.
  // The repository will catch these and convert to Failure objects.
  Never _throwMapped(PlatformException e) {
    final code = rc.PurchasesErrorHelper.getErrorCode(e);
    switch (code) {
      case rc.PurchasesErrorCode.purchaseCancelledError:
        throw ValidationException('Purchase cancelled');
      case rc.PurchasesErrorCode.networkError:
        throw NetworkException();
      case rc.PurchasesErrorCode.purchaseNotAllowedError:
        throw ValidationException('Purchase not allowed');
      case rc.PurchasesErrorCode.receiptAlreadyInUseError:
        throw ValidationException('Receipt already in use by another account');
      default:
        throw ServerException(e.message ?? 'RevenueCat error');
    }
  }
}
```

---

## Key things to understand

### Why `.storeProduct.identifier` not `.storeProduct.productIdentifier`?

In `purchases_flutter` v8, the property is `.identifier` on the `StoreProduct` object. Using `.productIdentifier` (which was correct in earlier versions) will cause a compile error.

### Why re-fetch offerings inside purchasePackage?

The datasource receives your `RCPackage` entity (your type), but the SDK needs its own `rc.Package` type. The simplest way to get the SDK's package object is to fetch offerings again and find the matching one by product identifier. The result is cached by the SDK so this is fast.

### What is `Never` as a return type?

`_throwMapped` always throws — it never returns normally. The `Never` return type tells Dart this and prevents "missing return" warnings in the calling methods.

---

**Next:** [08 — Repository →](08-repository.md)
