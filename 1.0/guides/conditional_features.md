# Conditional Features

[Back to the Documentation index](../index.md)

Flint supports conditional features that may not always be available to users. Your app almost always includes features that are not enabled by default, or those that can even be manually disabled by the user. On top of this, in-app purchases and feature-flagging are a classic case of conditional features, where the user first has to pay or meet some other criteria to see or use the feature.

Imagine a podcast player app with a feature hierarchy like this:

* App Features
	* Podcast playback
		* Dynamic Speed — requires purchase "Premium subscription", and user must turn it on
		* Sound enhancement  — requires purchase "Premium subscription", and user must turn it on
	* Podcast downloading
		* Turbo downloads — experimental feature currently for internal use only 
	* Podcast favourites
		* Smart playlists — requires location access and purchase "Premium subscription"


We use the concept of *constraints* on features to limit when they are available. There are many constraints you can apply, including platform versions, system permissions like Location or Camera access, purchases, and build-specific or user-toggled feature "flagging" (turning features on or off explicitly).

When using them in code, what makes conditional features special is that you cannot perform actions of conditional features directly. You must first check if the feature is available. If the feature is not available you should take appropriate action — perhaps prompting the user for any required permissions. Flint helps with that part too.

Using Flint to handle conditional features for you makes it much easier to write cross-platform code as you do not have to handle the case where the feature's definition is not even compiled in to the binary for a platform. Code paths that might involve a couple of features, one of which is not available on all platforms, are no longer a problem.

## Defining a conditional feature with constraints

You start by declaring the feature as conforming to the protocol `ConditionalFeature`. You then declare the constraints on the feature by implementing the `constraints` function.

Some constraints are evaluated once at startup – such as the minimum operating system version — and others which are evaluated at runtime when required, as these can change their value while the app is running. For example if a user makes an in-app purchase, once it is verified the purchase is valid, the features reliant on that purchase will now be available.

Here's an example from Flint's own condition feature for deep linking support"

```swift
/// Flint's deep linking feature for URL Routes is conditional so that you can disable
/// it if you don't want it.
public class DeepLinkingFeature: ConditionalFeature {
    public static var description: String = "Deep Linking and app-URL handling"

    public static func constraints(requirements: FeatureConstraintsBuilder) {
    	requirements.precondition(.runtimeEnabled)
    }

    // It's on by default. 
    public static var isEnabled: Bool? = true

    /// The action to use to perform the URL
    public static let performIncomingURL = action(PerformIncomingURLAction.self)
    
    public static func prepare(actions: FeatureActionsBuilder) {
        actions.declare(performIncomingURL)
    }
}
```

The `constraints(requirements:)` function definition uses a Domain-Specific Language (DSL) provided by the "builder" passed to the function. This DSL lets you define the constraints in a convenient way. The above constraint `.runtimeEnabled` means that the `isEnabled` property of the feature must return `true` for the feature to be available.. The default implementation of this provided by Flint returns `true`. You can override the property in your own features to allow assignments to it at compile or runtime, or make it call into other code to find out if it should be enabled.

Here are the kinds of constraints the DSL currently supports:

* **Preconditions**: This includes user feature toggling, purchases and runtime enabling.
* **System Permissions**: The permissions your app needs for the feature to work, such as Photos, Contacts or Location access
* **Platform restrictions**: You can set the minimum required version for each platform operating system, or indicate "any" or "unsupported". 

Flint allows you to combine all of these as appropriate to declare a simple set of requirements that handle all of the complexities of these disparate factors affecting feature availability at runtime.

## Defining preconditions

You declare a precondition constraint by calling the builder function `precondition` with a value of the [`FeaturePrecondition`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Constraints/FeaturePrecondition.swift) enum type.

At the time of writing there are three kinds of precondition supported:

* `.purchase(requirement: PurchaseRequirement)` — The feature requires one or more purchases before it is available.
* `.runtimeEnabled` — Whenever a request is made to use the feature, the value of `YourFeature.isEanbled` will be checked.
* `.userToggled(defaultValue: Bool)` — The Feature will check the user's current settings to see if this feature is enabled.

So if we wanted to have a somewhat contrived conditional feature that required a purchase but also had to be enabled by the user, say a level editor mode in a game, and the user could only access the level editor they paid for once they had completed in-game "training", you would set it up like this:

```swift
let premiumSubscription = Product(name: "💎 Premium Subscription",
								  description: "Unlock the level builder",
								  productID: "SUB0001")

public class LevelBuilderFeature: ConditionalFeature {
    public static var description: String = "Premium Level Builder"

    public static func constraints(requirements: FeatureConstraintsBuilder) {
    	requirements.preconditions(.userToggled(defaultValue: true),
    							   .runtimeEnabled,
    							   .purchase(PurchaseRequirement(premiumSubscription)))
    }

    static var isEnabled: Bool? = MyPlayerProgressTracker.shared.tutorialCompleted

    ...
}
```

Note that there are both `precondition()` and the plural `preconditions(...)` forms of this function. You can call these functions as many times as makes sense for your requirements, but currently `.purchase` is the only type that can be declared more than once, with different parameters.

For purchase preconditions, purchase requirements (see [`PurchaseRequirement`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Purchases/PurchaseRequirement.swift)) allow you to define rules based on one or more products, so that requirements can be fulfilled by several different purchases (say "Premium subscription" or "Generous supporter"), or require specific combinations. 

The user toggling precondition will read the value from the user's defaults, and if there is no current value will use the default value provided.

The `.runtimeEnabled` test will always check the static `isEnabled` property on the feature to verify it is `true`. You can use this for any kind of app-supplied runtime determination of the feature availability. You can also declare it as a writable property and just assign it `true`/`false` in the source file or at runtime to flag internal features that are perhaps not yet ready to ship.

Only if all the preconditions are `true` and all the other constraints are also met will the Feature be available.

## Defining required permissions

Permissions can be fiddly to deal with in apps. Your goal is to get people to authorise permissions so that they can use what you've implemented, but to improve the probability of this happening you have to explain what is happening and why. You shouldn't spam users at startup with authorisation requests for every permission your app might ever need. 

Associating permissions with the features that require them instantly solves the "don't spam the user with requests" part. If you only prompt for permissions when the feature is used, and only the permissions that feature needs, they already have some idea of what might be required, e.g. a mapping feature requires location access. They are primed to allow the permissions.

To declare a permission constraint, you use the `permission()` or `permissions(...)` functions:

```swift
public class SelfieFeature: ConditionalFeature {
    public static var description: String = "Selfie Posting"

    public static func constraints(requirements: FeatureConstraintsBuilder) {
    	requirements.permissions(.camera,
    							 .photos,
    							 .location(usage: .whenInUse))
    }

    ...
}
```

The permissions you pass in are any values from the [`SystemPermission`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Constraints/Permissions/SystemPermission.swift) enum. The kinds of permissions currently supported are:

* **.photos** — Photos access
* **.camera** — Camera access
* **.location(usage:)** — Location access

All the other permissions including those such as HealthKit, Bluetooth etc. are coming soon.

Once you add multiple permissions into the mix, and some features require different combinations of those permissions things get complicated quickly. Using conditional features with permission constraints solves this problem in a way that makes it easy for you to provide the best experience for the user, using Flint's tools for requesting the permissions. It also makes it very clear what your app's permission behaviours should be for testing and QA of the various features in all permutations with and without the permissions granted.

Permissions support enables Flint to hide all the details of verifying the different permissions as well as how you request authorisation. 

We'll see how to do that shortly, and after that we'll see how to handle the case where there are missing permissions.

## Defining platform restrictions

The constraints DSL also provides convenient ways to specify per-platform minimum version requirements.

Using the `.iOS`, `.watchOS`, `.tvOS`, `macOS` properties on the builder you can set a minimum version required for this feature to be available. You can alternatively set it to `.any`, or `.unsupported` if you want to prevent a feature on a certain platform where it doesn't make sense. The default for all platforms is `.any`.

There is also support for limiting a feature to a single platform by assigning a value to `.iOSOnly`, `.watchOSOnly`, `.macOSOnly` or `.tvOSOnly`. Assigning a version to any of these will set all the other platforms to `.unsupported` automatically.

Note that you can set version constraints either as an `Int` like "11", or a `String` like `"10.13.4"`:

```swift
public class ExampleFeature: ConditionalFeature {
    public static func constraints(requirements: FeatureConstraintsBuilder) {
    	requirements.iOS = 9
    	requirements.macOS = "10.12.1"
    	requirements.tvOS = .any
    	requirements.watchOS = .unsupported
    }

    ...
}

public class ExampleiOSOnlyFeature: ConditionalFeature {
    public static func constraints(requirements: FeatureConstraintsBuilder) {
    	// Availble on any iOS version, and nothing else
    	requirements.iOSOnly = .any
    }

    ...
}
```

## Performing actions of a conditional feature

To perform an action of a conditional feature you must be sure the feature is available. Flint uses Swift’s type system to enforce this: there is no way to `perform` an action of a conditional feature without first checking availability, unlike non-conditional features where you just call `perform`.

Instead you must first obtain a `ConditionalActionRequest` by calling `request()` on the action binding, and if you get a result back you can then call `perform` on that:

```swift
if let request = DeepLinkingFeature.performIncomingURL.request() {
    request.perform(using: presenter, with: url)
} else {
    // This often means there was a programmer error - 
    // your UI probably should not allow people to invoke actions from
    // features that are not enabled. A common exception is the case of permissions
    // and purchases where you want to let the user choose the action but then show
    // them UI explaining what they have to do to be able to use it.
    log.error("Attempt to use feature that is not enabled")
}
```

This type safety deliberately makes it painful to ignore the situations where a feature may not be available, and prevents confusing an always-available feature with one that isn’t. Your code that performs actions of conditional features always needs to be prepared to do something reasonable in case the feature is not available.

This is a far better than testing boolean feature flags through your code, where these can be easily missed, leading to undefined behaviours where some interactions are not gated on the feature flag by accident.

This is a **fundamental tenet of Flint**, that all your feature-related code should be triggered by action invocations, and the action invocations are protected by type safety so you cannot make these kinds of mistakes.

## What to do when your conditional feature indicates that it is not available

If you do not get a `request` instance back when you want to perform an action of a conditional feature, you need to check if this was because there are missing permissions.

There are many reasons why a feature may not be available, and Flint provides the information you need to deal with this. Some of the conditions you can help the user deal with — permissions and purchases for example — whereas others are programmer error such as a feature being disabled programmatically but the UI elements not being hidden or disabled.

One of the most common challenges is prompting the user to authorise permissions that are required. The various APIs of iOS, tvOS and watchOS that require permissions have subtle differences in their authorisation requirements, but Flint wraps all these up into a common interface. It also provides a controller and coordinator mechanism so that you don't have to do anything about authorisation except provide any onboarding UI that you want to include to help the user understand what is happening.

### Prompting for required permissions

When the feature `request()` call comes back with a nil, if it requires system permissions you'll need to see if there are any permissions that the user has not yet been asked to authorise. You do this using the `permissions.notDetermined` property on your feature type. Then you can use Flint's controller mechanism to request all the required authorisations.

```swift
if PhotoAttachmentsFeature.permissions.notDetermined.count > 0 {
    permissionController = PhotoAttachmentsFeature.permissionAuthorisationController(using: self)
    permissionController?.begin(retryHandler: retryHandler)
    return
}
```

This fragment does exactly this – it asks whether there are any permissions that the feature requires that have the `.notDetermined` status. If the count is non-zero, it calls Flint's `permissionAuthorisationController(using:)` function to get a controller instance that you can use to request all the authorisations, one by one. This controller takes a single optional argument of type `PermissionAuthorisationCoordinator`, which you can use to affect the authorisation process.

You can also pass `nil` for the coordinator and the user will see the system permission requests one by one without any hints about what is happening. That's fine during development but we don't recommend you do this in real apps. Use the coordinator to add some onboarding cards along the lines of:

> "Hey, we're going to need permission to access the camera because, well, you're going to be taking a photo!"

You would probably include buttons for "OK", "Ask me later" and "Cancel" so they can get out of the flow.

The coordinator receives callbacks at every stage in the cycle of authorisations and can do the following:

* Show UI at the start and end of the entire batch of required authorisations, including information about all the permissions required so you can adapt your content to this.
* Show some UI before and after the controller shows the system permission alert, so that the alert is not shown until the user has seen your UI and indicated they are ready to proceed.
* Set the order in which permissions will be requested — perhaps your onboarding wants to get the most important permissions requested first to avoid "prompt fatigue", or at least you want the ordering to be deterministic for automated testing.
* Skip authorising a specific permission - e.g. if your onboarding UI has an "Ask me later" button you can instruct the controller to skip a single permission.
* Cancel all the authorisations.

These are all very important for avoiding problems where users, through fatigue or confusion, deny one or more of the permissions when the system alert is displayed.
Remember that once permissions are denied you cannot prompt for them again in-app and at best you are forced to show UI that tells the user to open the "Settings" app and find the switch to enable the permissions for your app. In many cases if you end up in that situation you have "lost" the user. Flint also helps deal with these situations.

### Dealing with the wider range of permission and purchases issues

Whether your features require permissions and/or in-app purchases, things get more complication than you may at first think.

In terms of permissions, there are several authorisation statuses that do not grant access (only `.authorized` does).

The status `.denied` indicates that the user was previously asked for permission but denied it, and the app cannot recover from this or request permission again. So you need to show UI when you find some required permissions are denied, to tell them to go to the Settings app to enable them. Flint provides access to the denied permissions that your feature requires in the `permissions.denied` property on the feature.

Another status that requires special handling is `.restricted`. This indicates that for some reason the permission is simply not available and cannot be granted. Usually this means there are parental restrictions or a custom mobile device management profile installed that has denied access to the requested resource. For example some corporate managed devices may not permit camera or location access. You can't do anything about this, and often nor can the user, but you need to show UI to explain to them why they can't use the feature that requires the permission. Flint also provides access to the restricted permissions that your feature requires in the `permissions.restricted` property on the feature.

With in-app purchases, you can easily ask Flint whether there are purchases required to unlock the feature, using the `purchases.requiredToUnlock` property on the feature:

```swift
if feature.purchases.requiredToUnlock.count > 0 {
	// Show your in-app store onboarding UI
}
```

This is straightforward enough. When combined with the possibility of missing permissions, care should be taken to handle the possible remedies in a sensible way.

One example is that the user may not have granted camera access, but may also not have purchased the in-app purchase required for the feature. In this case, you don't want to prompt for permission unless they have actually purchased the feature.

What's more, if any of the permissions required are returning "restricted" you should not let them purchase the feature.

Regardless, you should only ask for one thing at a time and avoid confusing the user. Here's an example of how you might approach this:

```swift
func selectPhoto() {
    if let request = PhotoAttachmentsFeature.request(PhotoAttachmentsFeature.showPhotoSelection) {
        request.perform(using: self)
    } else {
        handleUnsatisfiedConstraints(for: PhotoAttachmentsFeature.self, retry: { [weak self] in self?.selectPhoto() })
    }
}

func handleUnsatisfiedConstraints<T>(for feature: T.Type, retry retryHandler: (() -> Void)?) where T: ConditionalFeature {
    // Check for required permissions that are restricted on this device through parental controls or a profile.
    // We must ask for these first in case the user purchases a feature they cannot use
    if feature.permissions.restricted.count > 0 {
        let permissions = feature.permissions.restricted.map({ String(describing: $0) }).joined(separator: ", ")
        let alertController = UIAlertController(title: "Permissiones are restricted!",
                                                message: "\(feature.name) requires permissions that are restricted on your device: \(permissions)",
                                                preferredStyle: .alert)
        alertController.addAction(UIAlertAction(title: "OK", style: .default, handler: nil))
        present(alertController, animated: true)
        return
    }
    
    // Check for required purchases next, only if there are no permissions that are restricted
    if feature.purchases.requiredToUnlock.count > 0 {
        let alertController = UIAlertController(title: "Purchase required!",
                                                message: "Sorry but \(feature.name) is a premium feature. Please make a purchase to unlock this feature.",
                                                preferredStyle: .alert)
        alertController.addAction(UIAlertAction(title: "OK", style: .default, handler: nil))
        present(alertController, animated: true)
        return
    }

    // Check for required permissions that are already denied
    if feature.permissions.denied.count > 0 {
        let permissions = feature.permissions.denied.map({ String(describing: $0) }).joined(separator: ", ")
        let alertController = UIAlertController(title: "Permissiones are denied!",
                                                message: "\(feature.name) requires permissions that you have denied. Please go to Settings to enable them: \(permissions)",
                                                preferredStyle: .alert)
        alertController.addAction(UIAlertAction(title: "OK", style: .default, handler: nil))
        present(alertController, animated: true)
        return
    }

    // Start the flow of requesting authorisation for any permissions not determined
    if feature.permissions.notDetermined.count > 0 {
        permissionController = feature.permissionAuthorisationController(using: self)
        permissionController?.begin(retryHandler: retryHandler)
        return
    }
}
```

This simple example uses alerts to tell the user what is wrong with the permissions. In a real app you would usually handle this with non-modal UI provided by the coordinator passed to `feature.permissionAuthorisationController`.

## Next steps

* Add [Routes](routes.md) to Actions
* Add [Activities](activities.md) support to some Actions
* Add [Analytics](analytics.md) tracking
* Use the [Timeline](timeline.md) to see what is going on in your app when things go wrong
* Start using [Focus](focus.md) to pare down your logging
 