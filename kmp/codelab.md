author: Jaewoong Eum
summary: Kotlin Multiplatform Purchases & Paywalls
id: codelab-3-kmp-sdk
categories: codelab,markdown
environments: Web
status: Published
feedback link: https://github.com/revenuecat/codelab/issues/new
analytics_ga4_account: G-MMFEH1TP0C

# KMP In-App Purchases & Paywalls

## Kotlin Multiplatform Purchases & Paywalls Overview
Duration: 0:02:00

Welcome to [RevenueCat](https://www.revenuecat.com/)'s KMP (Kotlin Multiplatform) SDK Codelab!

In this codelab, you will:

* Integrate the **[RevenueCat KMP SDK](https://www.revenuecat.com/docs/getting-started/installation/kotlin-multiplatform)** into your Kotlin Multiplatform project
* Implement **in-app purchases** in your KMP application
* Learn how to **distinguish between paying and non-paying users**
* Build a **paywall screen**, which is based on **server-driven UI** approach

By the end of this codelab, you’ll be able to successfully implement in-app purchases in your KMP app and display dynamic paywalls using RevenueCat's KMP SDK.

![overview](img/1.overview.png)

## Import RevenueCat SDK
Duration: 0:07:00

First things first, before implementing in-app purchases, you’ll need to import the RevenueCat SDK into your existing or new project. To get started, add the following dependency to your `build.gradle.kts` file:

You can check out the [latest release version on GitHub](https://github.com/RevenueCat/purchases-kmp/releases).

```gradle
[versions] 
purchases-kmp = "1.8.6+14.0.2"

[libraries]
purchases-core = { module = "com.revenuecat.purchases:purchases-kmp-core", version.ref = "purchases-kmp" }
purchases-ui = { module = "com.revenuecat.purchases:purchases-kmp-ui", version.ref = "purchases-kmp" }
```

You can now add the dependency to your `commonMain` source set in your module's build.gradle.kts.

```gradle
kotlin {
    // ...
    sourceSets {
        // ...
        commonMain.dependencies {  
            // Add the purchases-kmp dependencies.
            implementation(libs.purchases.core)  
            implementation(libs.purchases.ui)  
        }
    }
}
```

### Opt in to ExperimentalForeignApi

Since the SDK leverages Kotlin-generated bindings for native iOS code, you’ll need to opt in to the `ExperimentalForeignApi` in your iOS source sets. To enable this, add the following configuration to your module’s `build.gradle.kts` file:

```kotlin
kotlin {
    // ...
    sourceSets {
        // ...
        named { it.lowercase().startsWith("ios") }.configureEach {
            languageSettings {
                optIn("kotlinx.cinterop.ExperimentalForeignApi")
            }
        }
    }
}
```

### Link the native iOS SDK

Since the SDK depends on a native RevenueCat iOS framework, PurchasesHybridCommon, this will need to be linked to your existing iOS project. In this codelab, you'll configure the native iOS SDK by using CocoaPods.

You can include a complete Kotlin project as a CocoaPods dependency. To do this, specify the dependency in your project's `Podfile` by providing the name and the path to the directory containing the generated `.podspec` file.

1. Add the Kotlin CocoaPods Gradle plugin and the `purchases-common` version to your version catalog.
In the `gradle/libs.versions.toml` file, add the following to the `[plugins]` block:

```gradle
[versions] 
# This version must be equal to everything after the '+' in the purchases-kmp version.
purchases-common = "13.3.0"

[plugins]
kotlin-cocoapods = { id = "org.jetbrains.kotlin.native.cocoapods", version.ref = "kotlin" }
```

2. In your root `build.gradle.kts` file, add the following alias to the plugins {} block:

```gradle
alias(libs.plugins.kotlin.cocoapods) apply false
```

3. In the module where you want to integrate CocoaPods (e.g., composeApp), add the same alias to its `plugins {}` block:

```gradle
alias(libs.plugins.kotlin.cocoapods)
```

4. Next, configure cocoapods environments like the below:

```kotlin
kotlin {
    cocoapods {
        // Required properties
        // Specify the required Pod version here
        // Otherwise, the Gradle project version is used
        version = "1.0"
        summary = "Some description for a Kotlin/Native module"
        homepage = "Link to a Kotlin/Native module homepage"

        // Optional properties
        // Configure the Pod name here instead of changing the Gradle project name
        name = "MyCocoaPod"

        // .. other Cocoapods options are here. 

        // Add the PurchasesHybridCommon dependency.
        pod("PurchasesHybridCommon") {  
            version = libs.versions.purchases.common.get()  
            extraOpts += listOf("-compiler-option", "-fmodules")
        }        
    }
}
```

5. Finally, add the following `post_install` script to your iOS app target in the `Podfile`. For example, if your Kotlin Multiplatform module is named `shared`, your script should look like this:

```
target 'iosApp' do
  use_frameworks!
  platform :ios, '16.0'
  pod 'shared', :path => '../shared', :platforms => :ios

  # Add the following post_install script:
  post_install do |installer|
    installer.pods_project.targets.each do |target|
      if target.name == 'shared'
        target.build_configurations.each do |config|
          config.build_settings.delete('ASSETCATALOG_COMPILER_APPICON_NAME')
        end
      end
    end
  end
end 
```

You’ve successfully imported the RevenueCat SDK. Now, let’s move on to the initialization step.

## Initialization
Duration: 0:02:00

Now, it’s time to initialize the Purchases SDK in your project. Be sure to configure Purchases using **only your public API key**, which you can find by navigating to your app’s settings under **Project Settings > Platforms**.

```kotlin
import com.revenuecat.purchases.kmp.LogLevel
import com.revenuecat.purchases.kmp.Purchases
import com.revenuecat.purchases.kmp.configure

// If you have common initialization logic, call configure() there. If not,
// call it early in the app's lifecycle on the respective platforms.
// Note: make sure you use the correct api key for each platform. You could
// use Kotlin Multiplatform's expect/actual mechanism for this. 

Purchases.logLevel = LogLevel.DEBUG
Purchases.configure(apiKey = "<google_or_apple_api_key>") {
    appUserId = "<app_user_id>"
    // Other configuration options.
}
```

The `app_user_id` field in the `.configure` method is used by RevenueCat to identify users in your app. You can either provide a custom user identifier or omit it to let RevenueCat automatically generate an anonymous ID. For more details, refer to our [Identifying Users guide](https://www.revenuecat.com/docs/user-ids).

Yes! You’ve now completed 50% of the implementation.

## Validating Entitlements
Duration: 0:03:00

Now let’s move on to validating user entitlements.

As mentioned earlier, an entitlement represents the level of access or features a user unlocks after making a purchase. This makes it useful for determining things like whether to display an ad banner or grant premium access.

You can easily check if a user has an active entitlement using the code snippet below:

```kotlin
val ENTITLEMENT_IDENTIFIER = ".." // get specific entitlement identifier from your RevenueCat dashboard
val customerInfo = Purchases.sharedInstance.awaitCustomerInfo()
val isEntitled = customerInfo?.entitlements[ENTITLEMENT_IDENTIFIER]?.isActive == true
```

Once you’ve checked whether the user has a specific entitlement, you can decide how to proceed based on your app’s business model.

For example, if your app is ad-supported, you might choose to show or hide an AdMob banner. Alternatively, you could choose to display a paywall or purchase dialog, allowing users to unlock advanced features or content.

Here’s an example of how you might implement that logic:

```kotlin
@Composable
fun ContentScreen(isEntitled: Boolean) {
    if (isEntitled) {
      // if the user is granted access to this entitlement. don't need to display a banner.
    } else {
      // display a banner UI here or display a paywall
      ..
    }
}
```

## Implement In-App Purchases
Duration: 0:04:00

Now, let’s implement in-app purchases to offer an ad-free experience. To get started, you’ll first need to fetch the relevant product information from your RevenueCat dashboard. This product data will be used to present purchase options to your users.

You can retrieve the available products by calling `Purchases.sharedInstance.awaitGetProducts()`, as shown in the example below:

```kotlin
// fetches the product information from the RevenueCat server
val products = Purchases.sharedInstance.awaitGetProducts(
  productIds = listOf("paywall_tester.subs"),
)

// proceed in-app purchases
val purchaseResult = Purchases.sharedInstance.awaitPurchase(
  storeProduct = products.first()
)
```

If you offer multiple product variations, such as `paywall_tester.subs:weekly`, `paywall_tester.subs:monthly`, and `paywall_tester.subs:yearly`, you can simplify product retrieval by using the **base product identifier**, `paywall_tester.subs`, as the value for the `productIds` field. This tells RevenueCat to fetch **all related product variations** as a list, so you can dynamically present them in your paywall UI.

Once you've retrieved the product data, you can initiate the **in-app purchase flow** by calling `Purchases.sharedInstance.awaitPurchase(product)`. This will automatically trigger the **Google Play purchase dialog**, enabling the user to complete the transaction within your app.

Just like that, you've integrated a fully functional **in-app purchase flow** with just a few lines of code—no need to deal with the complexity of handling receipts, store APIs, or purchase validation manually.

## Implement Paywalls
Duration: 0:07:00

Now, it's time to impement Paywalls in your KMP project using Compose Multiplatform.

### Business Logic

To begin, you’ll need to **fetch the current offering** from the RevenueCat dashboard. An offering defines the available purchase options presented to the user, such as monthly, yearly, or lifetime plans. This can be done effortlessly using the `Purchases.sharedInstance.awaitOfferings()` method, as demonstrated in the example below.

```kotlin
internal class DetailsRepositoryImpl : DetailsRepository {

  override fun fetchOffering(): Flow<ApiResponse<Offering>> = flow {
    try {
      val offerings = Purchases.sharedInstance.awaitOfferings()
      offerings.current?.let { currentOffering ->
        val response = ApiResponse.of { currentOffering }
        emit(response)
      }
    } catch (e: PurchasesException) {
      ApiResponse.exception(e)
    }
  }
}
```

The RevenueCat KMP SDK has **native support for Kotlin coroutines**, making it a seamless fit for projects using a coroutine-based architecture. The current offering is also exposed as a **Flow**, allowing you to observe changes reactively and update your UI accordingly.

### Paywalls UI With Compose Multiplatform

At this point, everything should be ready to go. If you’ve already added the `com.revenuecat.purchases:purchases-kmp-ui` SDK, you can **easily build paywall UIs using Compose Multiplatform**.

RevenueCat’s UI library provides built-in components, such as `PaywallDialog` that allow you to **quickly display a paywall screen or dialog**. These components are fully customizable, offering a variety of configuration options to tailor the look and behavior to match your app’s design.

Here’s an example showing how simple it is to implement and customize a paywall using Compose Multiplatform:

```kotlin
val currentOffering by viewModel.offering.collectAsState()

val options = remember {
    PaywallOptions(dismissRequest = { TODO("Handle dismiss") }) {
        offering = currentOffering
        shouldDisplayDismissButton = true
    }
}

Paywall(options)
```

Configuration! 🥳 Now, you’ll be able to display paywalls whenever a user doesn’t have the required entitlement, using the exact same design you configured in the Paywall Editor. 

As you’ve already seen in the [Codelab: RevenueCat Google Play Integration (Create Paywalls)](https://revenuecat.github.io/codelab-internal-testing/google-play/codelab-1-google-play-integration/index.html#7), the paywall system is built on a **server-driven UI**. This means you can dynamically update the paywall’s content and design directly from the dashboard without needing to push app updates or go through the review process.

## Conclusion

In this codelab, you’ve learned how to integrate RevenueCat’s KMP SDK, implement in-app purchases, and build paywalls in Compose Multiplatform. Now it’s time to ship your app and make more money! 💰

You can also learn more about using the RevenueCat SDK with the resources below:

- [Product Tutorials](https://www.revenuecat.com/tutorials/): Video tutorials to help you get started and get the most out of RevenueCat.