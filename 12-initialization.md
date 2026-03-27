# 12 — Initialization

> **Reading time:** 8 minutes  
> **Previous:** [11 — Dependency Injection](11-dependency-injection.md)  
> **Next:** [13 — UI](13-ui.md)

---

## When to initialize RevenueCat

This is the most common source of bugs after dashboard setup. RevenueCat must be initialized with the correct user ID, at the right moment.

**The wrong way:** Initialize RC when the auth state changes, passing the user object from the token check. The user object at that point is a shell — it only has fields extracted from the JWT, which may not include the full user ID.

**The right way:** Initialize RC after the user's profile loads from your API (`/users/me`). At that point you have the confirmed, real user UUID.

---

## Where to put the initialization

In `AccountBloc`, after the profile fetch succeeds:

```dart
// lib/features/account/presentation/bloc/account_bloc.dart

@injectable
class AccountBloc extends Bloc<AccountEvent, AccountState> {
  final GetProfileUsecase _getProfile;
  final UpdateProfileUseCase _updateProfile;
  final ChangeLanguageUsecase _changeLanguage;
  final DeleteAccountUseCase _deleteAccount;
  final SubscriptionsBloc _subscriptionsBloc; // injected via constructor

  AccountBloc(
    this._getProfile,
    this._updateProfile,
    this._changeLanguage,
    this._deleteAccount,
    this._subscriptionsBloc,
  ) : super(const AccountState()) {
    on<FetchProfileEvent>(_onFetchProfile);
    // ... other handlers
  }

  Future<void> _onFetchProfile(
    FetchProfileEvent event,
    Emitter<AccountState> emit,
  ) async {
    emit(state.copyWith(status: AccountStatus.loading));
    final result = await _getProfile(NoParams());
    result.fold(
      (f) => emit(state.copyWith(
        status: AccountStatus.error,
        errorMessage: mapFailure(f),
      )),
      (profile) {
        // Initialize RC only if not already done.
        // profile.id is the real UUID from your API — guaranteed non-empty.
        if (_subscriptionsBloc.state.status == SubscriptionStatus.initial) {
          _subscriptionsBloc.add(InitializeRC(profile.id));
        }
        emit(state.copyWith(status: AccountStatus.loaded, profile: profile));
      },
    );
  }
}
```

---

## Sign out

When the user signs out, reset the RC identity so the next user starts fresh:

```dart
// lib/features/auth/presentation/bloc/auth_bloc.dart

Future<void> _onSignOut(SignOutEvent event, Emitter<AuthState> emit) async {
  emit(state.copyWith(status: AuthStatus.loading));
  final result = await _signOutUseCase();
  result.fold(
    (failure) => emit(
      state.copyWith(status: AuthStatus.error, errorMessage: mapFailureToMessage(failure)),
    ),
    (_) {
      _subscriptionsBloc.add(RCLogOut()); // resets RC identity + BLoC state
      emit(state.copyWith(status: AuthStatus.unauthenticated, user: null));
    },
  );
}
```

---

## The full lifecycle visualized

```
App starts
    ↓
AuthBloc: CheckAuthStatusEvent fires
    ↓
User is authenticated → AuthStatus.authenticated emitted
    ↓
App navigates to home screen
    ↓
AccountBloc: FetchProfileEvent fires
    ↓
GET /users/me → returns { id: "27fb0973-...", email: "...", ... }
    ↓
RC init NOT done yet → fires InitializeRC("27fb0973-...")
    ↓
SubscriptionsBloc: _onInitialize
    → rc.Purchases.configure(apiKey, appUserID: "27fb0973-...")
    → getOfferings() → 3 packages returned
    → getEntitlements() → { "premium": { isActive: true } }
    ↓
status: ready, offerings: [monthly, annual, lifetime], entitlements: { premium: active }
    ↓
state.isPremium == true → app unlocks premium features

User signs out
    ↓
RCLogOut() → rc.Purchases.logOut() → BLoC resets to initial
    ↓
Next user who logs in gets a fresh RC identity
```

---

## What happens if you initialize with an empty ID?

```
⚠️ Identifying with empty App User ID will be treated as anonymous.
👤 Setting new anonymous App User ID - $RCAnonymousID:510585fdb5...
```

An anonymous user's purchases are not linked to your user account. If the user reinstalls the app, they lose their subscription. Always pass the real user ID.

---

**Next:** [13 — UI →](13-ui.md)
