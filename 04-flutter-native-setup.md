# 04 — Flutter Native Setup

> **Reading time:** 8 minutes  
> **Previous:** [03 — Dashboard Setup](03-dashboard-setup.md)  
> **Next:** [05 — Project Structure](05-project-structure.md)

---

## Overview

Before writing any Dart code, the native iOS and Android projects need specific configuration. Skipping these steps causes the SDK to silently return empty results — no error messages, just blank data.

---

## Add the package

```yaml
# pubspec.yaml
dependencies:
  purchases_flutter: ^8.0.0
```

```bash
flutter pub get
```

---

## iOS setup

### 1. Add the In-App Purchase capability in Xcode

This is the most critical step for iOS. Without it, StoreKit cannot fetch products and everything returns empty.

1. Open `ios/Runner.xcworkspace` in Xcode (not the `.xcodeproj` file)
2. In the left panel, click **Runner** (the blue icon at the top)
3. Select the **Runner** target (not the project)
4. Click the **Signing & Capabilities** tab
5. Click **+ Capability**
6. Search for **In-App Purchase** and double-click it

You should see it appear as a new section with a shopping cart icon. This automatically updates your `Runner.entitlements` file.

**How to verify it worked:**

```bash
cat ios/Runner/Runner.entitlements
```

You should see:

```xml
<key>com.apple.developer.in-app-payments</key>
<array/>
```

If that key is missing, the capability was not saved properly. Try again.

### 2. Set minimum iOS version

`purchases_flutter` v8 requires iOS 13 minimum.

```ruby
# ios/Podfile — this must be the very first line
platform :ios, '13.0'
```

### 3. Reinstall pods

```bash
cd ios
pod deintegrate
pod install
cd ..
```

### 4. Verify your bundle ID

Your Xcode bundle ID must match what you registered in the RC dashboard exactly.

```bash
# Check what bundle ID Xcode is using:
cd ios && grep -r "PRODUCT_BUNDLE_IDENTIFIER" Runner.xcodeproj/project.pbxproj
```

You will see three entries (Debug, Release, Profile) — all three should match.

> ⚠️ Watch for hyphens vs underscores. `com.company.legal-defender` and `com.company.legal_defender` are different bundle IDs. Apple and RevenueCat treat them as completely separate apps.

To change the bundle ID: open Xcode → Runner target → Signing & Capabilities → change the Bundle Identifier field.

---

## Android setup

### 1. Add the billing permission

This goes **outside** the `<application>` tag. A very common mistake is putting it inside.

```xml
<!-- android/app/src/main/AndroidManifest.xml -->

<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- CORRECT: outside <application> -->
    <uses-permission android:name="com.android.vending.BILLING" />

    <application
        android:label="Your App Name"
        android:name="${applicationName}"
        android:icon="@mipmap/launcher_icon">

        <!-- your app content -->

    </application>

</manifest>
```

If you put it inside `<application>`, Android silently ignores it and billing will not work.

### 2. Set minimum SDK version

```gradle
// android/app/build.gradle
android {
    defaultConfig {
        minSdkVersion 21        // required by purchases_flutter
        targetSdkVersion 35
        compileSdkVersion 35
    }
}
```

### 3. Verify your package name

```bash
# Check your package name:
grep "applicationId" android/app/build.gradle
```

This must match what you registered in the RC dashboard exactly.

---

## Store your API keys

```bash
# env/.env  (for production)
PUBLIC_REVENUECAT_API_KEY_IOS=appl_xxxxxxxxxxxxxxxx
PUBLIC_REVENUECAT_API_KEY_ANDROID=goog_xxxxxxxxxxxxxxxx

# env/.dev.env  (for development)
PUBLIC_REVENUECAT_API_KEY_IOS=appl_xxxxxxxxxxxxxxxx
PUBLIC_REVENUECAT_API_KEY_ANDROID=goog_xxxxxxxxxxxxxxxx
```

> ⚠️ Never commit API keys to git. Add `env/` to your `.gitignore`.

---

## Clean build after setup

After making all native changes, always do a clean build:

```bash
flutter clean
flutter pub get
cd ios && pod install && cd ..
flutter run
```

A hot restart is not enough after native configuration changes.

---

## Checklist before moving on

- [ ] `purchases_flutter: ^8.0.0` added to `pubspec.yaml`
- [ ] In-App Purchase capability added in Xcode
- [ ] `Runner.entitlements` contains `com.apple.developer.in-app-payments`
- [ ] `ios/Podfile` starts with `platform :ios, '13.0'`
- [ ] `pod install` run after Podfile change
- [ ] iOS bundle ID verified and matches RC dashboard
- [ ] `BILLING` permission in Android Manifest (outside `<application>`)
- [ ] `minSdkVersion 21` in `android/app/build.gradle`
- [ ] Android package name verified and matches RC dashboard
- [ ] API keys stored in `.env` files, `.env` in `.gitignore`
- [ ] `flutter clean && flutter pub get` run

---

**Next:** [05 — Project Structure →](05-project-structure.md)
