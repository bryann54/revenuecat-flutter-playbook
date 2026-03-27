# 05 — Project Structure

> **Reading time:** 5 minutes  
> **Previous:** [04 — Flutter Native Setup](04-flutter-native-setup.md)  
> **Next:** [06 — Entities](06-entities.md)

---

## Why Clean Architecture?

You could write all your RevenueCat code directly in a widget. A lot of tutorials do this. The problem: when Apple releases a new StoreKit version, or you switch from RevenueCat to another provider, you have to change code everywhere.

Clean Architecture keeps the RevenueCat SDK locked in one file. Everything else in your app uses your own models and interfaces. Swapping providers means changing one file.

---

## The folder structure

```
lib/features/subscriptions/
│
├── data/
│   ├── datasources/
│   │   └── rc_subscription_datasource.dart
│   │       ← The ONLY file that imports purchases_flutter
│   │       ← Everything else is kept pure
│   │
│   └── repositories/
│       └── rc_subscription_repository_impl.dart
│           ← Converts exceptions to Failures
│           ← Calls the datasource, wraps in Either
│
├── domain/
│   ├── entities/
│   │   ├── rc_package_entity.dart
│   │   │   ← Your app's Package model (no SDK dependency)
│   │   └── rc_entitlement_entity.dart
│   │       ← Your app's Entitlement model (no SDK dependency)
│   │
│   ├── repositories/
│   │   └── rc_subscription_repository.dart
│   │       ← Abstract contract (interface)
│   │       ← What operations are available
│   │
│   └── usecases/
│       ├── rc_get_offerings_usecase.dart
│       ├── rc_purchase_package_usecase.dart
│       ├── rc_restore_purchases_usecase.dart
│       └── rc_get_entitlements_usecase.dart
│           ← One use case per action
│           ← Each does exactly one thing
│
└── presentation/
    └── bloc/
        ├── subscriptions_bloc.dart
        ├── subscriptions_event.dart
        └── subscriptions_state.dart
            ← State management
            ← The UI talks to the BLoC, never to use cases directly
```

---

## The flow of a purchase

To understand why the structure exists, trace a purchase from UI to SDK:

```
User taps "Subscribe"
    ↓
Widget fires: context.read<SubscriptionsBloc>().add(PurchasePackage(pkg))
    ↓
SubscriptionsBloc calls: _purchasePackage(RCPurchaseParams(event.package))
    ↓
RCPurchasePackageUseCase calls: _repository.purchasePackage(package)
    ↓
RCSubscriptionRepositoryImpl calls: _datasource.purchasePackage(package)
    ↓
RCSubscriptionDatasourceImpl calls: rc.Purchases.purchasePackage(sdkPackage)
    ↓  (RevenueCat SDK talks to App Store / Play Store)
App Store / Play Store processes payment
    ↓
Returns PurchaseResult with updated CustomerInfo
    ↓
Datasource maps to Map<String, RCEntitlement>
    ↓
Repository wraps in Right(entitlements)
    ↓
UseCase returns Either<Failure, Map<String, RCEntitlement>>
    ↓
BLoC emits SubscriptionState(status: success, entitlements: {...})
    ↓
Widget rebuilds: state.isPremium == true → show premium content
```

It looks long but each layer has one job. When something breaks, you know exactly where to look.

---

## The dependency rule

Dependencies only flow inward:

```
presentation → domain ← data
```

- `data` depends on `domain` (implements the repository interface)
- `presentation` depends on `domain` (uses entities and repository contract)
- `domain` depends on nothing — it is pure Dart

This means your domain layer (entities, use cases, repository contract) has zero third-party imports. The SDK is only in `data/datasources/`.

---

## What each layer knows about

| Layer | Knows about | Does NOT know about |
|---|---|---|
| `domain/entities` | Your own data models | RevenueCat, BLoC, Flutter |
| `domain/repositories` | Your entities, Either, Failure | How purchases actually happen |
| `domain/usecases` | Repository contract, entities | SDK, BLoC, UI |
| `data/datasources` | RC SDK, your entities | BLoC, UI, routing |
| `data/repositories` | Datasource, error types | BLoC, UI |
| `presentation/bloc` | Use cases, entities, state | SDK, HTTP, storage |
| `presentation/widgets` | BLoC state | Everything below BLoC |

---

**Next:** [06 — Entities →](06-entities.md)
