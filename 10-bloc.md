# 10 — BLoC

> **Reading time:** 15 minutes  
> **Previous:** [09 — Use Cases](09-use-cases.md)  
> **Next:** [11 — Dependency Injection](11-dependency-injection.md)

---

## Events

Events are things that happen in your app. The BLoC reacts to them.

```dart
// lib/features/subscriptions/presentation/bloc/subscriptions_event.dart
part of 'subscriptions_bloc.dart';

abstract class SubscriptionEvent extends Equatable {
  const SubscriptionEvent();
  @override
  List<Object?> get props => [];
}

// Fired after user logs in — starts the whole RC lifecycle
class InitializeRC extends SubscriptionEvent {
  final String userId;
  const InitializeRC(this.userId);
  @override List<Object?> get props => [userId];
}

// Fired when the paywall screen opens to refresh plans
class LoadOfferings extends SubscriptionEvent {}

// Fired when user taps a Subscribe button
class PurchasePackage extends SubscriptionEvent {
  final RCPackage package;
  const PurchasePackage(this.package);
  @override List<Object?> get props => [package];
}

// Fired when user taps "Restore Purchases"
class RestorePurchases extends SubscriptionEvent {}

// Fired to refresh what the user currently has
class LoadEntitlements extends SubscriptionEvent {}

// Fired on user sign-out to reset RC identity
class RCLogOut extends SubscriptionEvent {}
```

---

## State

The state is what the UI reads to decide what to show.

```dart
// lib/features/subscriptions/presentation/bloc/subscriptions_state.dart
part of 'subscriptions_bloc.dart';

enum SubscriptionStatus {
  initial,    // BLoC just created, nothing has happened yet
  loading,    // RC is initializing or fetching data
  ready,      // RC is initialized and data is loaded
  purchasing, // A purchase is in progress (show spinner)
  restoring,  // Restore is in progress (show spinner)
  success,    // Purchase or restore completed successfully
  error,      // Something went wrong
}

class SubscriptionState extends Equatable {
  final SubscriptionStatus status;
  final List<RCPackage> offerings;
  final Map<String, RCEntitlement> entitlements;
  final String? errorMessage;
  final String? successMessage;

  const SubscriptionState({
    this.status = SubscriptionStatus.initial,
    this.offerings = const [],
    this.entitlements = const {},
    this.errorMessage,
    this.successMessage,
  });

  // The most important getter in the whole feature.
  // This is what your UI checks to decide whether to show premium content.
  // Replace 'premium' with whatever you named your entitlement in the RC dashboard.
  bool get isPremium => entitlements['premium']?.isActive == true;

  SubscriptionState copyWith({
    SubscriptionStatus? status,
    List<RCPackage>? offerings,
    Map<String, RCEntitlement>? entitlements,
    String? errorMessage,
    String? successMessage,
  }) => SubscriptionState(
    status: status ?? this.status,
    offerings: offerings ?? this.offerings,
    entitlements: entitlements ?? this.entitlements,
    errorMessage: errorMessage,       // intentionally NOT null-coalesced
    successMessage: successMessage,   // null clears the message
  );

  @override
  List<Object?> get props => [
    status, offerings, entitlements, errorMessage, successMessage,
  ];
}
```

---

## The BLoC

```dart
// lib/features/subscriptions/presentation/bloc/subscriptions_bloc.dart

import 'package:bloc/bloc.dart';
import 'package:equatable/equatable.dart';
import 'package:injectable/injectable.dart';
import 'package:your_app/common/helpers/base_usecase.dart';
import 'package:your_app/common/utils/functions.dart';
import 'package:your_app/core/errors/failures.dart';
import '../../domain/entities/rc_entitlement_entity.dart';
import '../../domain/entities/rc_package_entity.dart';
import '../../domain/repositories/rc_subscription_repository.dart';
import '../../domain/usecases/rc_get_entitlements_usecase.dart';
import '../../domain/usecases/rc_get_offerings_usecase.dart';
import '../../domain/usecases/rc_purchase_package_usecase.dart';
import '../../domain/usecases/rc_restore_purchases_usecase.dart';

part 'subscriptions_event.dart';
part 'subscriptions_state.dart';

@injectable
class SubscriptionsBloc extends Bloc<SubscriptionEvent, SubscriptionState> {
  final RCSubscriptionRepository _repository;
  final RCGetOfferingsUseCase _getOfferings;
  final RCPurchasePackageUseCase _purchasePackage;
  final RCRestorePurchasesUseCase _restorePurchases;
  final RCGetEntitlementsUseCase _getEntitlements;

  // Guard flag — prevents any RC call before initialize() completes.
  // If the UI fires LoadOfferings before RC is initialized, it is silently ignored.
  bool _rcInitialized = false;

  SubscriptionsBloc(
    this._repository,
    this._getOfferings,
    this._purchasePackage,
    this._restorePurchases,
    this._getEntitlements,
  ) : super(const SubscriptionState()) {
    on<InitializeRC>(_onInitialize);
    on<LoadOfferings>(_onLoadOfferings);
    on<PurchasePackage>(_onPurchasePackage);
    on<RestorePurchases>(_onRestorePurchases);
    on<LoadEntitlements>(_onLoadEntitlements);
    on<RCLogOut>(_onLogOut);
  }

  Future<void> _onInitialize(
    InitializeRC event,
    Emitter<SubscriptionState> emit,
  ) async {
    emit(state.copyWith(status: SubscriptionStatus.loading));

    final initResult = await _repository.initialize(event.userId);
    await initResult.fold(
      (failure) async => emit(state.copyWith(
        status: SubscriptionStatus.error,
        errorMessage: mapFailure(failure),
      )),
      (_) async {
        _rcInitialized = true;

        // CRITICAL: Use typed separate awaits — NOT Future.wait
        // Future.wait erases generic types at runtime and causes:
        // "type '() => List<dynamic>' is not a subtype of '() => List<RCPackage>'"
        final Either<Failure, List<RCPackage>> offeringsResult =
            await _getOfferings(NoParams());
        final Either<Failure, Map<String, RCEntitlement>> entitlementsResult =
            await _getEntitlements(NoParams());

        emit(state.copyWith(
          status: SubscriptionStatus.ready,
          offerings: offeringsResult.getOrElse(() => <RCPackage>[]),
          entitlements: entitlementsResult.getOrElse(() => <String, RCEntitlement>{}),
        ));
      },
    );
  }

  Future<void> _onLoadOfferings(
    LoadOfferings event,
    Emitter<SubscriptionState> emit,
  ) async {
    if (!_rcInitialized) return;
    emit(state.copyWith(status: SubscriptionStatus.loading));
    final result = await _getOfferings(NoParams());
    result.fold(
      (f) => emit(state.copyWith(
        status: SubscriptionStatus.error,
        errorMessage: mapFailure(f),
      )),
      (pkgs) => emit(state.copyWith(
        status: SubscriptionStatus.ready,
        offerings: pkgs,
      )),
    );
  }

  Future<void> _onPurchasePackage(
    PurchasePackage event,
    Emitter<SubscriptionState> emit,
  ) async {
    if (!_rcInitialized) return;
    emit(state.copyWith(status: SubscriptionStatus.purchasing));
    final result = await _purchasePackage(RCPurchaseParams(event.package));
    result.fold(
      (f) => emit(state.copyWith(
        status: SubscriptionStatus.error,
        errorMessage: mapFailure(f),
      )),
      (entitlements) => emit(state.copyWith(
        status: SubscriptionStatus.success,
        entitlements: entitlements,
        successMessage: 'Welcome to premium!',
      )),
    );
  }

  Future<void> _onRestorePurchases(
    RestorePurchases event,
    Emitter<SubscriptionState> emit,
  ) async {
    if (!_rcInitialized) return;
    emit(state.copyWith(status: SubscriptionStatus.restoring));
    final result = await _restorePurchases(NoParams());
    result.fold(
      (f) => emit(state.copyWith(
        status: SubscriptionStatus.error,
        errorMessage: mapFailure(f),
      )),
      (entitlements) => emit(state.copyWith(
        status: SubscriptionStatus.success,
        entitlements: entitlements,
        successMessage: entitlements.isNotEmpty
            ? 'Purchases restored!'
            : 'No active purchases found.',
      )),
    );
  }

  Future<void> _onLoadEntitlements(
    LoadEntitlements event,
    Emitter<SubscriptionState> emit,
  ) async {
    if (!_rcInitialized) return;
    final result = await _getEntitlements(NoParams());
    result.fold(
      (f) => emit(state.copyWith(
        status: SubscriptionStatus.error,
        errorMessage: mapFailure(f),
      )),
      (entitlements) => emit(state.copyWith(
        status: SubscriptionStatus.ready,
        entitlements: entitlements,
      )),
    );
  }

  Future<void> _onLogOut(
    RCLogOut event,
    Emitter<SubscriptionState> emit,
  ) async {
    await _repository.logOut();
    _rcInitialized = false;
    emit(const SubscriptionState()); // full reset to initial state
  }
}
```

---

**Next:** [11 — Dependency Injection →](11-dependency-injection.md)
