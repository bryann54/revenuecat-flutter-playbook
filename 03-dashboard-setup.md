# 03 — Dashboard Setup

> **Reading time:** 10 minutes  
> **Previous:** [02 — Key Concepts](02-key-concepts.md)  
> **Next:** [04 — Flutter Native Setup](04-flutter-native-setup.md)

---

## Do this before writing any Flutter code

Seriously. The most common reason subscriptions do not work is skipping or half-completing the dashboard setup. The SDK will not give you a useful error — it will just silently return empty. Set everything up first, then write code.

---

## Step 1 — Create your RevenueCat project

1. Go to [app.revenuecat.com](https://app.revenuecat.com) and sign up or log in
2. Click **+ New Project** and give it your app's name
3. Inside the project, go to **Apps & providers**
4. Add your **iOS app** — enter your bundle ID exactly (e.g. `com.yourcompany.yourapp`)
5. Add your **Android app** — enter your package name exactly (e.g. `com.yourcompany.yourapp`)

> ⚠️ The bundle ID / package name must match your app exactly. A mismatch here means RevenueCat cannot connect your purchases to your app. Check for typos, hyphens vs underscores, and uppercase vs lowercase.

After adding apps, go to **API Keys** in the left sidebar and save both keys:
- iOS key starts with `appl_`
- Android key starts with `goog_`

You will need these later in your `.env` file.

---

## Step 2 — Create products in the stores

RevenueCat does NOT create products. You create them in Apple's and Google's systems, then import them into RevenueCat.

### iOS — App Store Connect

1. Go to [appstoreconnect.apple.com](https://appstoreconnect.apple.com)
2. Select your app
3. Go to **Monetization** → **Subscriptions**
4. Click **+** to create a **Subscription Group** (e.g. "Premium")
5. Inside the group, click **+** to add a subscription:
   - Reference Name: `Monthly Premium`
   - Product ID: `com.yourapp.premium.monthly` (you choose this — remember it)
   - Duration: 1 Month
   - Price: set your price
6. Add more products for annual, weekly, etc.
7. Save each one — status must show **Ready to Submit**

> ⚠️ Products in a "Missing Metadata" or "Developer Action Needed" status will NOT be fetchable by the SDK.

### Android — Google Play Console

1. Go to [play.google.com/console](https://play.google.com/console)
2. Select your app
3. Go to **Monetize** → **Subscriptions**
4. Click **Create subscription**:
   - Product ID: `com.yourapp.premium.monthly` (use the same ID as iOS if possible)
   - Name: `Monthly Premium`
   - Billing period: Monthly
   - Price: set your price
5. Click **Save** — it will be created as **Active**

> ⚠️ For Google Play Billing to work at all, your app must be published to at least the **Internal Testing** track. An app that was never published cannot query products, even in development.

---

## Step 3 — Import products into RevenueCat

Now that the products exist in the stores, tell RevenueCat about them:

1. In the RC dashboard, go to **Product catalog** → **Products**
2. Click **+ New**
3. Enter the Product ID exactly as you created it in the store (e.g. `com.yourapp.premium.monthly`)
4. Select the platform (iOS or Android)
5. Click **Save**

Repeat for every product. If you have both iOS and Android versions of the same product, add them separately (once for iOS, once for Android).

---

## Step 4 — Create an Entitlement

This is the "VIP wristband" from the concepts guide. Your app code will check this to decide if a user gets premium access.

1. Go to **Product catalog** → **Entitlements**
2. Click **+ New**
3. Identifier: `premium` (this string is what you use in your Dart code — keep it simple)
4. Description: `Access to all premium features`
5. Click **Add**
6. On the entitlement detail page, click **Attach** and attach ALL your products to it

> 💡 Any user who purchases ANY of the attached products will have this entitlement activated. You are saying: "buying monthly OR annual OR lifetime should all give the user the `premium` entitlement."

---

## Step 5 — Create an Offering

1. Go to **Product catalog** → **Offerings**
2. Click **+ New**
3. Identifier: `default` (important — RC uses this for the current offering)
4. Description: anything
5. Click **Add**

Now add packages to the offering:

1. On the offering detail page, scroll to **Packages**
2. Click **+ New Package** for each billing period:

   **Monthly package:**
   - Identifier: `$rc_monthly` (use RC's reserved identifier)
   - Display name: `Monthly`
   - Click Save, then attach your monthly product for each platform

   **Annual package:**
   - Identifier: `$rc_annual`
   - Display name: `Annual`
   - Attach your annual product

> ⚠️ When attaching products to a package, you will see a green icon for connected platforms and a grey icon for unconnected ones. Both iOS and Android should be green. If iOS shows grey, your iOS products are not attached and iOS users will see no plans.

---

## Step 6 — Set the Offering as Current ★

This is the step everyone forgets.

1. Go back to **Product catalog** → **Offerings**
2. You will see your offerings listed
3. Find `default` and click the **★ star icon** to make it the current offering

Without this, `offerings.current` in your app will always return `null`.

---

## Checklist before moving on

Work through this before going to Step 4:

- [ ] RevenueCat project created with iOS and Android apps
- [ ] API keys saved (iOS: `appl_...`, Android: `goog_...`)
- [ ] Products created in App Store Connect (status: Ready to Submit)
- [ ] Products created in Google Play Console (app published to Internal Testing)
- [ ] All products imported into RC dashboard
- [ ] Entitlement `premium` created with all products attached
- [ ] Offering `default` created with packages
- [ ] Each package has both iOS and Android products attached (green icons)
- [ ] Offering `default` is set as Current (★ star)

If you cannot tick every box, do not proceed — the SDK will return empty results and debugging it is very frustrating.

---

**Next:** [04 — Flutter Native Setup →](04-flutter-native-setup.md)
