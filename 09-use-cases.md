# 09 — Use Cases

> **Reading time:** 5 minutes  
> **Previous:** [08 — Repository](08-repository.md)  
> **Next:** [10 — BLoC](10-bloc.md)

---

## One use case, one job

Use cases are tiny classes that each do one thing. They exist so your BLoC is not directly coupled to the repository. This makes testing easier and keeps things readable.

```dart
// lib/features/subscriptions/domain/usecases/rc_get_offerings_usecase.dart

import 'package:dartz/dartz.dart';
import 'package:injectable/injectable.dart';
import 'package:your_app/common/helpers/base_usecase.dart';
import 'package:your_app/core/errors/failures.dart';
import '../entities/rc_package_entity.dart';
import '../repositories/rc_subscription_repository.dart';

@injectable
class RCGetOfferingsUseCase implements UseCase<List<RCPackage>, NoParams> {
  final RCSubscriptionRepository _repository;
  const RCGetOfferingsUseCase(this._repository);

  @override
  Future<Either<Failure, List<RCPackage>>> call(NoParams params) =>
      _repository.getOfferings();
}
```

```dart
// lib/features/subscriptions/domain/usecases/rc_purchase_package_usecase.dart

class RCPurchaseParams {
  final RCPackage package;
  const RCPurchaseParams(this.package);
}

@injectable
class RCPurchasePackageUseCase
    implements UseCase<Map<String, RCEntitlement>, RCPurchaseParams> {
  final RCSubscriptionRepository _repository;
  const RCPurchasePackageUseCase(this._repository);

  @override
  Future<Either<Failure, Map<String, RCEntitlement>>> call(
    RCPurchaseParams params,
  ) => _repository.purchasePackage(params.package);
}
```

```dart
// lib/features/subscriptions/domain/usecases/rc_restore_purchases_usecase.dart

@injectable
class RCRestorePurchasesUseCase
    implements UseCase<Map<String, RCEntitlement>, NoParams> {
  final RCSubscriptionRepository _repository;
  const RCRestorePurchasesUseCase(this._repository);

  @override
  Future<Either<Failure, Map<String, RCEntitlement>>> call(NoParams params) =>
      _repository.restorePurchases();
}
```

```dart
// lib/features/subscriptions/domain/usecases/rc_get_entitlements_usecase.dart

@injectable
class RCGetEntitlementsUseCase
    implements UseCase<Map<String, RCEntitlement>, NoParams> {
  final RCSubscriptionRepository _repository;
  const RCGetEntitlementsUseCase(this._repository);

  @override
  Future<Either<Failure, Map<String, RCEntitlement>>> call(NoParams params) =>
      _repository.getEntitlements();
}
```

---

## NoParams

If your `base_usecase.dart` does not already have `NoParams`, add it:

```dart
class NoParams extends Equatable {
  @override
  List<Object?> get props => [];
}
```

---

**Next:** [10 — BLoC →](10-bloc.md)
