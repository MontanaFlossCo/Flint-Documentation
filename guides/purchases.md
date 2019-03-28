---
title: Purchases
subtitle: How to limit access to features with purchases
tags:
    - purchases
    - iap
    - conditional
    - store
---

#### In this article:
{:.no_toc}
* TOC
{:toc}

## Overview

Using Flint’s [`Conditional Features`](conditional_features.md) you can specify which purchases or subscriptions are required to unlock features in your app. The first step is to make the relevant features conform to [`ConditionalFeature`](conditional_features.md) and add constraints that specify which purchases are required. Because you are using the conditional feature request mechanism to perform actions, you have no code changes to make to control access to these features. If you want to also prompt the user to purchase, you just need to handle the case where the request is denied and show your store UI.

The purchase constraints allow you to specify the purchases that can unlock each feature in terms of the user purchasing product “X”, products “A or B” or “A and B” or “(A or B) and C”. 

Support is included for the different purchase types offered by StoreKit but is not restricted to that API so you can connect any purchase verification implementation you like. Flint does not attempt to provide a full implementation of purchase tracking, merely the wiring you need for the enabling of features based on purchase status. Specifically, Flint provides no APIs for implementing an in-app purchase store UI.

The types of purchases supported are:

* Non-Consumable: these are a simple `true`/`false` unlock that does not change once purchased — once you buy it you have it. Flint supplies a default StoreKit purchase tracker implementation that tracks this kind of purchase for you.
* Consumable: features can specify that a certain number of a “product” representing a type of credits is required to unlock the feature. This is purely informational and the app is always responsible for indicating whether this feature is unlocked at the current point in time. Your debug or store UI code can use the information about the number of credits for informational purposes.
* Subscriptions: features can specify the type of subscription products that will unlock the feature, but again the app must indicate whether or not the subscription is currently active and track expiration. This applies for non-renewing (season pass) and auto-renewing subscriptions.

Whenever you [make a request](conditional_features.md) to perform an action of a conditional feature, the constraints evaluator will evaluate all the constraints and check with your `PurchaseTracker` implementation to see if the feature’s purchase constraints are currently met.

## How purchases are detected

The information about which purchases have been made is provided by an implementation of [`PurchaseTracker`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Purchases/PurchaseTracker.swift) assigned to the [`Flint.purchaseTracker`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Core/Flint.swift) property.

By default, Flint will create an instance of `StoreKitPurchaseTracker` if you do not assign one yourself. This is a basic implementation that will observe the StoreKit payment queue for information about transactions. This works only for non-consumable in-app purchases but can serve as a base for your own implementations that handle consumables or subscriptions.

Whether or not you provide your own `PurchaseTracker` implementation, you can use a `DebugPurchaseTracker` instance to wrap your real tracker, so that you can easily override purchase status during testing.

Flint also provides a debug UI for iOS that lets you quickly view and override the status of purchases used by the features in your app.

## Defining purchase requirements

There is a base type [`Product`]() that contains the basic information about a product that you use as a purchase requirement. You will use the appropriate subclass of this to define your product. The base types to choose from are:

* [`NonConsumableProduct`]()
* [`ConsumableProduct`]()
* [`AutoRenewingSubscriptionProduct`]()
* [`NonRenewingSubscriptionProduct`]()

```swift
enum MyInAppPurchases {
    let oneOffPurchaseProduct = NonConsumableProduct(name: "💎 Premium",
                                                     description: "Unlock premium features",
                                                     productID: "PREM0001")
    
    let monthlySubscriptionProduct = AutoRenewingSubscriptionProduct(name: "💫 Monthly Subscription", 
                                                                     description: "Unlock everything",
                                                                     productID: "SUB0001")
    
    let creditsZBucksProduct = ConsumableProduct(name: "A Z Buck",
                                                 description: "We saw you coming, whale",
                                                 productID: "CREDIT-ZBUCK")
}
```

It is cleaner to specify these as static properties within and `enum` or other type so you can refer to them easily. 

Once you have these product defined, you can use them to create one or more [`PurchaseRequirement`]() in the constraints of your feature. There are various ways to add one or more purchases, with AND/OR combinations.

```swift
import FlintCore

final class PhotoAttachmentsFeature: ConditionalFeature {
    static func constraints(requirements: FeatureConstraintsBuilder) {
        requirements.iOSOnly = 10
        
        requirements.permissions(.photos, .camera)
        requirements.permission(.contacts(entity: .contacts))    
                
        // Require the one off purchase. Multiple calls to
        // `purchase` or passing a list of arguments will
        // create an "AND" of all the requirements,
        // so this is not a realistic example
        requirements.purchase(MyInAppPurchases.oneOffPurchaseProduct)

        // Require 5 credits. This is just returned as 
        // information about what is needed, they are not 
        // automatically deducted or the unlock tracked.
        requirements.purchase(MyInAppPurchases.creditsZBucksProduct, quantity: 5)

        // Alternatively, you could require either that or subscription
                
        requirements.purchase(anyOf:
            MyInAppPurchases.oneOffPurchaseProduct,
            MyInAppPurchases.monthlySubscriptionProduct
        )

        // You could also require the one off purchase plus some credits
        
        requirements.purchase(allOf:
            .init(MyInAppPurchases.oneOffPurchaseProduct),
            .init(MyInAppPurchases.creditsZBucksProduct, quantity: 5)
        )
    }

    ...
}
```

## Using StoreKitPurchaseTracker

This default purchase tracker observes the StoreKit purchase queue and stores the transaction status of non-consumable products in a JSON file on the device. There is no receipt validation provided by this implementation and support is only provided for non-consumable purchases. You can subclass `StoreKitPurchaseTracker` to provide implementations of the functions required for subscription and consumable purchase checking.

You have nothing else to do once this tracker is set up and your purchase constraints have been defined. You can see the status of the purchases using the debug UI, described in the next section.

If you are developing an app with Extensions that also require access to the information about active purchases, you must assign your app's shared group container ID to `FlintAppInfo.appGroupIdentifier` at startup, or construct the `StoreKitPurchaseTracker` instance yourself with the appropriate ID.

**NOTE**: If you really need receipt validation you can implement this yourself, as recommended by Apple. Our philosophy is that the out of the box experience should be simple for developers, and people who go to the effort to hack their devices or tamper with your app's files are never going to pay you whatever you do. You may disagree, and you can channel that energy into your own implementation — but you should also take care to ensure it is not trivial for hackers to bypass Flint's logic for feature availability. This notwithstanding, just implement your own `PurchaseTracker` and assign it to `Flint.purchaseTracker` at start up and you're good to go. However, support for subscriptions will require the extraction of the expiration date from the current receipt file so you must implement this yourself.

## Adding the Purchase Browser debug UI for iOS

Adding the `FlintUI` dependency to your app, you can then use the `PurchaseBrowserFeature` to show a view controller which will show purchase status:

![Purchase Browser screenshot](images/PurchaseTracker-unknown.png){:width="300px"}

If you have used the debug purchase tracker, you can also tap purchases and override their current status to faake the purchase, non-purchase or "unknown" status of each purchase.

## Adding the DebugPurchaseTracker for runtime toggling of purchase status

The debug tracker can be used standalone without a real tracker — useful for developing your purchase-based UI and testing how your feature/purchase gating feels — or you can use it to proxy a real tracker and provide runtime overrides for the actual purchase status of your active Apple ID.

![Purchase Browser overrides screenshot](images/PurchaseTracker-overrides.png){:width="300px"}
![Purchase Browser purchased screenshot](images/PurchaseTracker-purchased.png){:width="300px"}

To use the debug tracker to proxy the store tracker, you do something like this before setting up Flint in your app delegate:

```swift
let storeKitTracker = try! StoreKitPurchaseTracker(appGroupIdentifier: FlintAppInfo.appGroupIdentifier)
Flint.purchaseTracker = DebugPurchaseTracker(targetPurchaseTracker: storeKitTracker)
```

That's all there is to it!

## Next steps

* Using [Flint UI](flint_ui.md) for development debugging tools
