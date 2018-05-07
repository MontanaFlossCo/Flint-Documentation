# Features and Actions

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

You declare a precondition constraint by calling the builder function `precondition` with a value of the [`FeaturePrecondition`]() enum type.

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

For purchase preconditions, purchase requirements (see [`PurchaseRequirement`]()) allow you to define rules based on one or more products, so that requirements can be fulfilled by several different purchases (say "Premium subscription" or "Generous supporter"), or require specific combinations. 

The user toggling precondition will read the value from the user's defaults, and if there is no current value will use the default value provided.

The `.runtimeEnabled` test will always check the static `isEnabled` property on the feature to verify it is `true`. You can use this for any kind of app-supplied runtime determination of the feature availability. You can also declare it as a writable property and just assign it `true`/`false` in the source file or at runtime to flag internal features that are perhaps not yet ready to ship.

Only if all the preconditions are `true` and all the other constraints are also met will the Feature be available.

## Defining required permissions

Permissions are particularly fiddly to deal with in apps. You shouldn't spam users at startup with authorisation requests for every permission your app might ever need, and you need to encourage them to approve permission alerts by creating a smooth experience that explains why they need to approve it.

You also need to gracefully handle the case where they don't approve the permission and tell them how to enable it in the Settings app if they later need to use the feature. 

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

The permissions you pass in are any values from the [`SystemPermission`]() enum. The kinds of permissions currently supported are:

* **.photos** — Photos access
* **.camera** — Camera access
* **.location(usage:)** — Location access

All the other permissions including those such as HealthKit, Bluetooth etc. are coming soon.
	
Once you add multiple permissions into the mix, and some features require different combinations of those permissions things get complicated quickly. Using conditional features with permission constraints solves this problem in a way that makes it easy for you to provide the best experience for the user. It also makes it very clear what your app's permissions behaviours should be for testing and QA of the various features in all permutations with and without the permissions granted.

By specifying the permissions that each feature requires and not the superset of all the permissions your app requires, Flint can check these permissions for you at the time the user tries to use the feature. The user is not bothered by permission alerts until they need them, and then only the ones they need right then.

Permissions support enables Flint to hide all the details of verifying the different permissions as well as how you request authorisation. You just don't need to worry about any of that any more, as all you do is check if a feature is available to use by calling `MyFeature.isAvailable` or more often — use `MyFeature.request()` to try to perform an action.

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

To perform an action of a conditional feature you must first test if the feature is available. Flint uses Swift’s type system to enforce this: there is no way to `perform` an action of a conditional feature without first checking availability. You must obtain a `ConditionalActionRequest` by calling `request()`, and if you get a result back you can then call `perform` on that:

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

This is a lot better than requiring simple boolean feature checks through your code, where these are easily missed, leading to undefined behaviours where some interactions not gated on the feature flag by accident. 

This is a **fundamental tenet of Flint**, that all your feature-related code should be triggered by action invocations, and the action invocations are protected by type safety so you cannot make these kinds of mistakes.

## Handling system permissions that require authorization

If you do not get a `request` instance back when you want to perform an action of a conditional feature, you need to check if this was because there are missing permissions.

We will be simplifying this process very soon, but for now, you can do something like this:

```swift
func selectPhoto() {
    if let request = PhotoAttachmentsFeature.request(PhotoAttachmentsFeature.showPhotoSelection) {
        request.perform(using: self)
    } else {
        // Check if it failed because of permissions
        let constraints = Flint.constraintsEvaluator.evaluate(for: PhotoAttachmentsFeature.self)
        if constraints.unsatisfied.permissions.count > 0 {
            
        	// For simplicity we'll just ask for the first one
            let permission = constraints.unsatisfied.permissions.first!
            let status = Flint.permissionChecker.status(of: permission)

            if status == .denied {
                let alertController = UIAlertController(title: "Permission required!", message: "You need to grant Photos access. Please go to Settings, Flint Demo and enable photos access.", preferredStyle: .alert)
                alertController.addAction(UIAlertAction(title: "OK", style: .default, handler: nil))
                present(alertController, animated: true)
            } else if status == .restricted {
                let alertController = UIAlertController(title: "Access is restricted!", message: "You need Photos access but this is restricted on your device.", preferredStyle: .alert)
                alertController.addAction(UIAlertAction(title: "OK", style: .default, handler: nil))
                present(alertController, animated: true)
            } else if status == .notDetermined {
            	// Ask Flint to request permission
                Flint.permissionChecker.requestAuthorization(for: permission)
            }
        }
    }
}
```

This simple example uses alerts to tell the user what is wrong with the permission, and will authorise only the first unfulfilled permission. In a real app you would handle this with non-modal UI, and you would show some instructive UI before requesting authorization to increase the chance of the user granting it.

## Next steps

* Add [Routes](routes.md) to Actions
* Add [Activities](activities.md) support to some Actions
* Add [Analytics](analytics.md) tracking
* Use the [Timeline](timeline.md) to see what is going on in your app when things go wrong
* Start using [Focus](focus.md) to pare down your logging
