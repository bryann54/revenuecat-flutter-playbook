# 11 — Dependency Injection

> **Reading time:** 6 minutes  
> **Previous:** [10 — BLoC](10-bloc.md)  
> **Next:** [12 — Initialization](12-initialization.md)

---

## Register the API key

Add `rcApiKey` to your existing `RegisterModules`. The `@Named` annotation tells Injectable to inject it wherever `@Named('rcApiKey')` appears in constructors.

Do not add `@lazySingleton` to a String — only use `@Named`. Primitives cannot be singletons in Injectable.

```dart
// lib/core/di/register_module.dart

import 'dart:io';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:injectable/injectable.dart';
import 'package:shared_preferences/shared_preferences.dart';

@module
abstract class RegisterModules {
  @preResolve
  Future<SharedPreferences> get prefs => SharedPreferences.getInstance();

  @Named('BaseUrl')
  String get baseUrl => dotenv.env['BASE_URL'] ?? '';

  @lazySingleton
  FlutterSecureStorage get secureStorage => const FlutterSecureStorage();

  // Platform-aware RC key — automatically uses the right key for iOS or Android
  // @Named only, no @lazySingleton — primitives cannot be registered as singletons
  @Named('rcApiKey')
  String get rcApiKey => Platform.isIOS
      ? (dotenv.env['PUBLIC_REVENUECAT_API_KEY_IOS'] ?? '')
      : (dotenv.env['PUBLIC_REVENUECAT_API_KEY_ANDROID'] ?? '');
}
```

---

## Regenerate after any change

Every time you add, remove, or change an `@injectable`, `@LazySingleton`, or `@module` annotation, you must regenerate the DI config:

```bash
dart run build_runner build --delete-conflicting-outputs
```

---

## Verify the generated config

After regenerating, open `lib/core/di/injector.config.dart` and search for `rcApiKey`. You should see:

```dart
gh.registerLazySingleton<String>(
  () => registerModule.rcApiKey,
  instanceName: 'rcApiKey',
);
```

And the datasource wired with it:

```dart
gh.registerLazySingleton<RCSubscriptionDatasource>(
  () => RCSubscriptionDatasourceImpl(gh<String>(instanceName: 'rcApiKey')),
);
```

If either of these is missing, the code generator did not pick up your changes. Try a full clean:

```bash
flutter clean
dart run build_runner clean
dart run build_runner build --delete-conflicting-outputs
```

---

## Provider order in main.dart

`SubscriptionsBloc` is injected into both `AuthBloc` and `AccountBloc` as a constructor parameter. This means `SubscriptionsBloc` must be provided first in the widget tree so it exists when the others are created.

```dart
// lib/main.dart — the order matters

MultiProvider(
  providers: [
    ChangeNotifierProvider(create: (_) => localeProvider),
    BlocProvider(create: (context) => getIt<SubscriptionsBloc>()),  // FIRST
    BlocProvider(create: (context) => getIt<AuthBloc>()),           // second
    BlocProvider(create: (context) => getIt<AccountBloc>()),        // third
    // ... rest of your blocs
  ],
)
```

---

**Next:** [12 — Initialization →](12-initialization.md)
