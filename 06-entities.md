# 06 — Entities

> **Reading time:** 5 minutes  
> **Previous:** [05 — Project Structure](05-project-structure.md)  
> **Next:** [07 — Datasource](07-datasource.md)

---

## What are entities?

Entities are your app's data models. They look like the SDK's models but have zero imports from `purchases_flutter`. This matters because:

- If RevenueCat changes their SDK, you update one file (the datasource), not every widget
- You can write unit tests without the SDK
- The rest of your app never needs to import `purchases_flutter` at all

---

## The Package entity

Notice the enum is named `RCPackageType`, not `PackageType`. This is intentional — `purchases_flutter` also exports a `PackageType` enum. If both have the same name, Dart gets confused. Naming yours `RCPackageType` eliminates the collision entirely.

```dart
// lib/features/subscriptions/domain/entities/rc_package_entity.dart

import 'package:equatable/equatable.dart';

// Named RCPackageType to avoid collision with purchases_flutter's PackageType
enum RCPackageType { monthly, annual, weekly, lifetime, unknown }

class RCPackage extends Equatable {
  final String identifier;
  final RCPackageType packageType;
  final String productIdentifier;
  final String localizedPriceString; // e.g. "$4.99" — already formatted
  final String localizedTitle;       // e.g. "Monthly Premium"
  final String localizedDescription;

  const RCPackage({
    required this.identifier,
    required this.packageType,
    required this.productIdentifier,
    required this.localizedPriceString,
    required this.localizedTitle,
    required this.localizedDescription,
  });

  @override
  List<Object?> get props => [identifier, productIdentifier];
}
```

---

## The Entitlement entity

```dart
// lib/features/subscriptions/domain/entities/rc_entitlement_entity.dart

import 'package:equatable/equatable.dart';

class RCEntitlement extends Equatable {
  final String identifier;        // e.g. "premium"
  final bool isActive;            // true = user has access right now
  final DateTime? expirationDate; // null for lifetime purchases
  final String? productIdentifier;
  final String? store;            // "app_store" or "play_store"

  const RCEntitlement({
    required this.identifier,
    required this.isActive,
    this.expirationDate,
    this.productIdentifier,
    this.store,
  });

  @override
  List<Object?> get props => [identifier, isActive, expirationDate];
}
```

---

## Why `localizedPriceString` and not `price`?

The price from the store is already formatted for the user's locale and currency. `"$4.99"` in the US, `"€4,99"` in Germany, `"KES 650"` in Kenya. You should display this string directly — do not try to format it yourself.

---

**Next:** [07 — Datasource →](07-datasource.md)
