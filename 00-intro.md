# 💳 RevenueCat Flutter Playbook

> In-app subscriptions for Flutter devs.

---

## Who is this for?

This guide is for any Flutter developer who needs to add subscriptions to a mobile app using RevenueCat. It assumes you know basic Flutter but have never touched in-app purchases before and uses a simple bloc feature 

By the end you will have:
- A working subscription paywall
- Entitlement checking (know if a user is premium)
- Purchase + restore flows
- Clean Architecture / BLoC wiring that scales

---

## Read these in order

| # | File | What you'll learn |
|---|---|---|
| 01 | [What is RevenueCat](docs/01-what-is-revenuecat.md) | The concept, why it exists, the mental model |
| 02 | [Key Concepts](docs/02-key-concepts.md) | Product, Offering, Package, Entitlement — explained simply |
| 03 | [Dashboard Setup](docs/03-dashboard-setup.md) | Everything to configure before writing code |
| 04 | [Flutter Native Setup](docs/04-flutter-native-setup.md) | iOS Xcode + Android Manifest configuration |
| 05 | [Project Structure](docs/05-project-structure.md) | Clean Architecture file layout and why |
| 06 | [Entities](docs/06-entities.md) | Your app's data models, independent of the SDK |
| 07 | [Datasource](docs/07-datasource.md) | The only file that touches the RC SDK |
| 08 | [Repository](docs/08-repository.md) | Error handling layer between datasource and domain |
| 09 | [Use Cases](docs/09-use-cases.md) | One use case per action |
| 10 | [BLoC](docs/10-bloc.md) | State management — events, states, the bloc |
| 11 | [Dependency Injection](docs/11-dependency-injection.md) | Wiring everything together with Injectable |
| 12 | [Initialization](docs/12-initialization.md) | When and where to initialize RC in your app |
| 13 | [UI](docs/13-ui.md) | Paywall screen, feature gating, restore button |
| 14 | [Common Errors](docs/14-common-errors.md) | Every error you'll hit and how to fix it |
| 15 | [Quick Reference](docs/15-quick-reference.md) | Cheat sheet for when you just need a reminder |

---

## Tech stack used in this guide

- Flutter 3.x
- [purchases_flutter](https://pub.dev/packages/purchases_flutter) v8+
- [flutter_bloc](https://pub.dev/packages/flutter_bloc)
- [injectable](https://pub.dev/packages/injectable) + [get_it](https://pub.dev/packages/get_it)
- [dartz](https://pub.dev/packages/dartz) (Either type for error handling)
- [flutter_dotenv](https://pub.dev/packages/flutter_dotenv) (for API keys)

---

## The one thing to remember

RevenueCat is a middleman between your app and the stores. You never talk to Apple or Google directly — you talk to RevenueCat and it handles both platforms for you.

```
Your App  →  RevenueCat SDK  →  App Store (iOS)
                            →  Play Store (Android)
```

---

*Built by developer who spent way too long debugging empty offerings😂.*
