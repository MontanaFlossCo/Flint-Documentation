# Features and Actions

Flint supports conditional features that can be toggled programmatically at runtime or based on a purchase or a user-preference. Simply conform to the `ConditionalFeature` protocol instead:

```swift
/// Flint's own deep linking feature for URL Routes is conditional so that you can disable it if you don't want it.
public class DeepLinkingFeature: ConditionalFeature {
    public static var description: String = "Deep Linking and app-URL handling"

    /// Indicate that this needs `isAvailable` to be checked at runtime
    public static var availability: FeatureAvailability = .runtimeEvaluated
    
    /// Turned on by default, this can be turned off at runtime by setting it to `false`
    public static var isAvailable: Bool? = true

    /// The action to use to perform the URL
    public static let performIncomingURL = action(PerformIncomingURLAction.self)
    
    public static func prepare(actions: FeatureActionsBuilder) {
        actions.declare(performIncomingURL)
    }
}
```

To perform an action of a conditional feature you must first test if the feature is available. Flint uses’s Swift’s type system to enforce this: there is no way to `perform` an action of a conditional feature without first checking availability. You must obtain a `ConditionalActionRequest` by calling `request`:

```swift
if let request = DeepLinkingFeature.performIncomingURL.request() {
    request.perform(using: presenter, with: url)
} else {
    // This probably means there was a programmer error - 
    // your UI should not allow people to invoke actions from
    // features that are not enabled
    log.error("Attempt to use feature that is not enabled")
}
```

This type safety deliberately makes it painful to ignore the situations where a feature may not be available, and prevents confusing an always-available feature with one that isn’t. Your code that performs actions of conditional features always needs to be prepared to do something reasonable in case the feature is not available.

The previous code samples only declare the actions and allow you to perform them. Even at this level of simplicity, if an `ActionDispatchObserver` is registered, it will be able to do something whenever these actions are performed – such as emitting an analytics tracking event for each action. Flint provides such an observer called `AnalyticsReporting` which you can use to route analytics to whatever backend you use.

## Next steps

* Add [Routes](routes.md) to Actions
* Add [Activities](activities.md) support to some Actions
* Add [Analytics](analytics.md) tracking
* Use the [Timeline](timeline.md) to see what is going on in your app when things go wrong
* Start using [Focus](focus.md) to pare down your logging