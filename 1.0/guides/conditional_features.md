# Features and Actions

Flint supports conditional features that can may not always be available to users. We use constraints on features to limit when they are available. There are numerous constraints you can apply, but they include platform versions, system permissions like Location or Camera access, and build-specific or user-toggled feature "flagging" (turning features on or off explicitly).

When using them in code, what makes conditional features special is that your code cannot perform actions of conditional features directly. You must first check if the feature is available. If the feature is not available you should take appropriate action — perhaps prompting the user for any required permissions. Flint helps with that part too.

Using Flint to handle conditional features for you makes it much easier to write cross-platform code as you do not have to handle the case where the feature's definition is not even compiled in to the binary for a platform. These types are extremely lightweight so there is no need to exclude them from platform targets that don't actually need to support them. Code paths that might involve a couple of features, one of which is not available on all platforms, become easy to deal with.

## Defining a ConditionalFeature with constraints

As the developer you set the constraints on the feature, some of which are evaluated once at startup – such as the minimum operating system version — and others which are evaluated at runtime when required, and these can change their value at runtime. For example if a user makes an in-app purchase, once it is verified the features reliant on that purchase will now be available.

You start by declaring the feature as conforming to the protocol `ConditionalFeature`:

```swift
/// Flint's own deep linking feature for URL Routes is conditional so that you can disable it if you don't want it.
public class DeepLinkingFeature: ConditionalFeature {
    public static var description: String = "Deep Linking and app-URL handling"

    public static func constraints(requirements: FeatureConstraintsBuilder) {
    	requirements.precondition(.runtimeEnabled)
    }

    /// The action to use to perform the URL
    public static let performIncomingURL = action(PerformIncomingURLAction.self)
    
    public static func prepare(actions: FeatureActionsBuilder) {
        actions.declare(performIncomingURL)
    }
}
```

The `constraints(requirements:)` function definition uses a Domain-Specific Language (DSL) provided by the "builder" passed to the function. This DSL lets you define different kinds of constraints in convenient ways. The above constraint `.runtimeEnabled` means that the `isEnabled` property of the feature must return `true`. The default implementation of this provided by Flint returns `true`. You can override it to turn this feature off, or make it call into other code to find out if it should be enabled.

Here are the kinds of constraints the DSL currently supports:

* **Preconditions**: This includes user toggling, purchases and runtime enabling.
* **System Permissions**: The permissions your app needs for the feature to work, such as Photos, Contacts or Location access
* **Platform restrictions**: You can set the minimum required version for each platform operating system, or indicate "any" or "unsupported". 

Flint allows you to combine all of these as appropriate to declare a simple set of requirements that handle all of the complexities of these disparate factors affecting feature availability at runtime.

## Defining preconditions

You declare a precondition requirement by calling the builder function `precondition` with a value of the [`FeaturePrecondition`]() enum type.

At the time of writing there are three supported preconditions:

* `.purchase(requirement: PurchaseRequirement)` — The feature requires one or more purchases before it is available.
* `.runtimeEnabled` — Whenever a request is made to use the feature, the value if `YourFeature.isEanbled` will be checked.
* `.userToggled(defaultValue: Bool)` — The Feature will check for the user's current settings to see if this feature is enabled.

So if we wanted to have an obscure feature condition that required a purchase but also had to be enabled by the user, say some a "level builder" mode in a game, and the user could only access the level builder they paid for once they had completed "training", you would set it up like this:

```swift
let premiumSubscription = Product(name: "💎 Premium Subscription", description: "Unlock the level builder", productID: "SUB0001")

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

Note that there are both `precondition()` and plural `preconditions()` variadic forms of this function. You can call these functions as many times as makes sense for your requirements, but currently `.purchase` is the only type that can exist with multiple parameters.

For purchase preconditions, purchase requirements (see [`PurchaseRequirement`]()) allow you to define rules based on multiple products, so that requirements can be fulfilled by many different purchases, or require specific combinations. 

The user toggling will read and write the value from the user's defaults, and if there is no current value will use the default value provided.

The `.runtimeEnabled` test will always check the static `isEnabled` property on the feature to verify it is `true`. You can use this for any kind of app-supplied runtime determination of the feature availability. You can also declare it as a writable property and just assign it `true`/`false` in the source file to flag internal features that are perhaps not yet ready to ship.

Only if all the preconditions are `true` will the Feature be available — and only if all the other constraints are also met.

## Defining required permissions

Permissions are particularly difficult to deal with in apps. You shouldn't spam users at startup with authorisation requests for every permission your app might ever need, and you need to encourage them to approve permission alerts by creating a smooth experience that explains why they need to approve it. You also need to gracefully handle the case where they don't approve the permission and tell them how to enable it in the Settings app if they later need to use the feature. 

Once you add multiple permissions into the mix, and only some features required different combinations of those features things get complicated quickly. Using conditional features with permission constraints solves this problem in a way that makes it easy for you to provide the best experience for the user.

By specifying the permissions each feature requires and not the superset of all the permissions your app requires, Flint can check these permissions only at the time the user tries to use the feature. The user is not bothered by permission alerts until they need them, and then only the ones they need right then.

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

The permission you pass in is any value from the [`SystemPermission`]() enum. The kinds of permissions currently supported are:

* **.photos** — Photos access
* **.camera** — Camera access
* **.location(usage:)** — Location access

All the other permissions including those such as HealthKit, Bluetooth etc. are coming soon!
	
Using this permission mechanism means Flint can hide all the details of verifying the different permissions as well as how you request authorisation. You just don't need to worry about any of that any more, as all you do is check if a feature is available to use by calling `MyFeature.isAvailable` or most often `MyFeature.request()` to try to perform an action.

Let's see how we do that next.

## Performing actions of a conditional feature

To perform an action of a conditional feature you must first test if the feature is available. Flint uses’s Swift’s type system to enforce this: there is no way to `perform` an action of a conditional feature without first checking availability. You must obtain a `ConditionalActionRequest` by calling `request`:

```swift
if let request = DeepLinkingFeature.performIncomingURL.request() {
    request.perform(using: presenter, with: url)
} else {
    // This often means there was a programmer error - 
    // your UI probably should not allow people to invoke actions from
    // features that are not enabled. An exception is the case of permissions and purchases
    // where you want to let the user choose the action but then show them UI explaining
    // what they have to do to be able to use it.
    log.error("Attempt to use feature that is not enabled")
}
```

This type safety deliberately makes it painful to ignore the situations where a feature may not be available, and prevents confusing an always-available feature with one that isn’t. Your code that performs actions of conditional features always needs to be prepared to do something reasonable in case the feature is not available.

The previous code samples only declare the actions and allow you to perform them. Even at this level of simplicity, if an `ActionDispatchObserver` is registered, it will be able to do something whenever these actions are performed – such as emitting an analytics tracking event for each action. Flint provides such an observer called `AnalyticsReporting` which you can use to route analytics to whatever backend you use.

## Handling system permissions that require authorization

Improvements to this coming very soon. Currently, you can do something like this:

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
            	// Request permission!
                Flint.permissionChecker.requestAuthorization(for: permission)
            }
        }
    }
}
```

## Defining platform restrictions

The DSL provides convenient ways to specify per-platform minimum version requirements.

Using the `.iOS`, `.watchOS`, `.tvOS`, `macOS` properties on the builder you can set a minimum version required for this feature to be available, . You can alternatively set it to `.any`, or `.unsupported` if you want to prevent a feature on a certain platform where it doesn't make sense but your common code still needs to reference it. The default for all platforms is `.any`.

There is also support for limiting support to a single platform by assigning a value to `.iOSOnly`, `.watchOSOnly`, `.macOSOnly` or `.tvOSOnly`. Assigning a version to any of these will set all the other platforms to `.unsupported` automatically.

Note that you can set version constraints as either as an `Int` like "11", a `String` like `10.13.4`:

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

## Next steps

* Add [Routes](routes.md) to Actions
* Add [Activities](activities.md) support to some Actions
* Add [Analytics](analytics.md) tracking
* Use the [Timeline](timeline.md) to see what is going on in your app when things go wrong
* Start using [Focus](focus.md) to pare down your logging
