# 08 — Repository

> **Reading time:** 8 minutes  
> **Previous:** [07 — Datasource](07-datasource.md)  
> **Next:** [09 — Use Cases](09-use-cases.md)

---

## What the repository does

The repository sits between the datasource and your use cases. Its job is:

1. Call the datasource
2. Catch any exceptions thrown
3. Return `Right(result)` on success or `Left(failure)` on error

This is where the `Either` type from `dartz` comes in. `Either<Failure, T>` means "this operation either failed (Left) or succeeded with a value of type T (Right)". Your use cases and BLoC only deal with `Either` — they never see raw exceptions.

---

## The abstract contract

```dart
// lib/features/subscriptions/domain/repositories/rc_subscription_repository.dart

import 'package:dartz/dartz.dart';
import 'package:your_app/core/errors/failures.dart';
import '../entities/rc_package_entity.dart';
import '../entities/rc_entitlement_entity.dart';

abstract class RCSubscriptionRepository {
  /// Call once after the user logs in, with their real user ID
  Future<Either<Failure, Unit>> initialize(String userId);

  /// Fetch available subscription plans to show on the paywall
  Future<Either<Failure, List<RCPackage>>> getOfferings();

  /// Buy a plan — returns updated entitlements on success
  Future<Either<Failure, Map<String, RCEntitlement>>> purchasePackage(RCPackage package);

  /// Restore previous purchases (required by App Store guidelines)
  Future<Either<Failure, Map<String, RCEntitlement>>> restorePurchases();

  /// Get what the user currently has active
  Future<Either<Failure, Map<String, RCEntitlement>>> getEntitlements();

  /// Check one specific entitlement by key
  Future<Either<Failure, bool>> hasEntitlement(String entitlementKey);

  /// Reset RC identity when user signs out
  Future<Either<Failure, Unit>> logOut();
}
```

---

## The implementation

```dart
// lib/features/subscriptions/data/repositories/rc_subscription_repository_impl.dart

import 'package:dartz/dartz.dart';
import 'package:injectable/injectable.dart';
import 'package:your_app/core/errors/exceptions.dart';
import 'package:your_app/core/errors/failures.dart';
import '../datasources/rc_subscription_datasource.dart';
import '../../domain/entities/rc_entitlement_entity.dart';
import '../../domain/entities/rc_package_entity.dart';
import '../../domain/repositories/rc_subscription_repository.dart';

@LazySingleton(as: RCSubscriptionRepository)
class RCSubscriptionRepositoryImpl implements RCSubscriptionRepository {
  final RCSubscriptionDatasource _datasource;
  const RCSubscriptionRepositoryImpl(this._datasource);

  @override
  Future<Either<Failure, Unit>> initialize(String userId) =>
      _guardUnit(() => _datasource.initialize(userId));

  @override
  Future<Either<Failure, List<RCPackage>>> getOfferings() =>
      _guard(() => _datasource.getOfferings());

  @override
  Future<Either<Failure, Map<String, RCEntitlement>>> purchasePackage(
    RCPackage package,
  ) => _guard(() => _datasource.purchasePackage(package));

  @override
  Future<Either<Failure, Map<String, RCEntitlement>>> restorePurchases() =>
      _guard(() => _datasource.restorePurchases());

  @override
  Future<Either<Failure, Map<String, RCEntitlement>>> getEntitlements() =>
      _guard(() => _datasource.getEntitlements());

  @override
  Future<Either<Failure, bool>> hasEntitlement(String entitlementKey) async {
    final result = await getEntitlements();
    return result.map(
      (entitlements) =>
          entitlements.containsKey(entitlementKey) &&
          entitlements[entitlementKey]!.isActive,
    );
  }

  @override
  Future<Either<Failure, Unit>> logOut() =>
      _guardUnit(() => _datasource.logOut());

  // ── Error guards ──────────────────────────────────────────────────────────
  // These catch every exception type and convert to Failure objects.
  // Use _guard for operations that return a value.
  // Use _guardUnit for operations that return nothing (void).

  Future<Either<Failure, T>> _guard<T>(Future<T> Function() fn) async {
    try {
      return Right(await fn());
    } on ServerException catch (e) {
      return Left(ServerFailure(error: e.message));
    } on NetworkException {
      return const Left(NetworkFailure());
    } on UnauthorizedException {
      return Left(UnauthorizedFailure());
    } on NotFoundException catch (e) {
      return Left(NotFoundFailure(error: e.message));
    } on ValidationException catch (e) {
      return Left(ValidationFailure(error: e.message));
    } catch (e) {
      return Left(GeneralFailure(error: e.toString()));
    }
  }

  Future<Either<Failure, Unit>> _guardUnit(Future<void> Function() fn) async {
    try {
      await fn();
      return const Right(unit);
    } on ServerException catch (e) {
      return Left(ServerFailure(error: e.message));
    } on NetworkException {
      return const Left(NetworkFailure());
    } on ValidationException catch (e) {
      return Left(ValidationFailure(error: e.message));
    } catch (e) {
      return Left(GeneralFailure(error: e.toString()));
    }
  }
}
```

---

**Next:** [09 — Use Cases →](09-use-cases.md)
