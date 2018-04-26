# Activities

Apple platforms is `NSUserActivity` for a variety of purposes to tell the operating system about something the user is doing. It is used across the platforms to make the user experience more efficient. This includes support for Handoff, Siri Suggestions (AKA Siri Proactive), Spotlight Search, and even ClassKit for education apps.

Flint’s Activities feature can automatically register `NSUserActivity` for you when users perform actions in your app. You can determine which actions qualify for this (“Save” is not something that makes sense for a Handoff action), and control the attributes passed to the operating system.

## Enabling Activities on an Action

Building on this, we can enable the [Activities](guides/activities.md) feature of Flint itself, and we get Siri suggestions, Handoff and Spotlight integration with a tiny amount of code. This is a smart wrapper around Apple’s `NSUserActivity` functionality. Let’s add the declaration for a new “Document Open” action that supports just Handoff and Siri Suggestions/Pro-active:

```swift
final class DocumentOpenAction: Action {
    typealias InputType = DocumentRef
    typealias PresenterType = DocumentPresenter

    static var description = "Open a document"
    
    /// Declare the types of activity we want published.
    /// This is all we have to do, aside from add `NSUserActivityTypes` to Info.plist
    /// and list the activity IDs. See docs for details
    static var activityTypes: Set<ActivityEligibility> = [.perform, .handoff]
    
    static func perform(with context: ActionContext<DocumentRef>, using presenter: DocumentPresenter, completion: ((ActionPerformOutcome) -> ())) {
        // … not important for this example
    }
}
```

Now when the app is run, it will automatically publish an `NSUserActivity` when that action occurs, and if you run the app on two devices, it will show a Handoff icon when you have the document open. You’ll need to add a [single function call from Flint.swift](FlintCore/Core/Flint.swift) to `application(:continue:restorationHandler:)` to pass the incoming activity to Flint, but that’s it! Of course you can customise the other properties of the `NSUserActivity` easily if you need to.

## Adding code so your app handles incoming activities

Docs coming soon

## Setting custom attributes on an Activity

Suppose now that we want to add Spotlight Search integration. We just need to tell Flint how to describe the action’s input to Spotlight. We add a `prepare` function:

```swift
final class DocumentOpenAction: Action {
    /// Change `activityTypes` to include search
    static var activityTypes: Set<ActivityEligibility> = [.perform, .handoff, .search]
    
    /// Prepare the attributes for the activity
    static func prepare(activity: NSUserActivity, with input: InputType?) -> NSUserActivity? {
        guard let input = input else {
            preconditionFailure("Input was expected")
        }
        let searchAttributes: CSSearchableItemAttributeSet = CSSearchableItemAttributeSet(itemContentType: kUTTypeText as String)
        searchAttributes.addedDate = Date()
        searchAttributes.kind = "Flint Demo Document"
        searchAttributes.contentDescription = "A test document for Spotlight indexing support"
        activity.contentAttributeSet = searchAttributes
        activity.keywords = "decentralized internet patent"
        activity.title = input.name
        return activity
    }
}
```

Now when run, any document that is opened will automatically be registered for search indexing. Go to the home screen, pull down for search and type “Flint Demo” and you will find the document. Tap it and the app will open at that document. This is a slightly odd arrangement — you’d normally submit these items for indexing separately when loading your data store, and reference that from the `NSUserActivity` — but it demonstrates the possibilities.

This is all just scratching the surface of what is possible. For more details see the documentation for [Features and Actions](guides/features_and_actions.md), [Timeline](guides/timeline.md), [Focus](guides/focus.md), [Activities](guides/activites.md), [Routes](guides/routes.md) and [Action Stacks](guides/action_stacks.md).

For a more detailed working example you can check out and build the [FlintDemo-iOS](https://github.com/MontanaFlossCo/FlintDemo-iOS) project which demonstrates many of the capabilities of Flint.

## Things Flint cannot do for you

You need to update Info.plist `NSUserActivityTypes`. Flint generates automatic activity types using the pattern XXXXXXXXXXXX, but you can also explicitly set the ID yourself. Either way all the types have to be listed in your app’s `Info.plist`

## Troubleshooting and testing

* Activities that also register the item for Spotlight search *must* have a title. Flint will alert you to this if you forget.
* If Handoff or other activity

