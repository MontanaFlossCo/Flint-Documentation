# Activities

[Back to the Documentation index](../index.md)

Apple platforms use `NSUserActivity` for a variety of purposes to tell the operating system about something the user is doing. It is used across the platforms to make the user experience more efficient. This includes support for Handoff, Siri Suggestions (AKA Siri Proactive), Spotlight Search, deep linking and even ClassKit for education apps.

Flint’s Activities feature can automatically register `NSUserActivity` for you when users perform actions in your app. You can determine which actions qualify for this (“Save” is not something that makes sense for a Handoff action), and control the attributes passed to the operating system.

## Enabling Activities on an Action

We can enable the Activities feature of Flint itself, and we get Siri suggestions, Handoff and Spotlight integration with a tiny amount of code. Let’s add the declaration for a new “Document Open” action that supports just Handoff and Siri Suggestions/Pro-active:

```swift
final class DocumentOpenAction: Action {
    typealias InputType = DocumentRef
    typealias PresenterType = DocumentPresenter

    static var description = "Open a document"
    
    /// Declare the types of activity we want published.
    static var activityTypes: Set<ActivityEligibility> = [.perform, .handoff]
    
    static func perform(with context: ActionContext<DocumentRef>, using presenter: DocumentPresenter, completion: ((ActionPerformOutcome) -> ())) {
        // … not important for this example
    }
}
```

If you have already mapped this action with URL Routes, when the app is run it will automatically publish an `NSUserActivity` when that action occurs, and if you run the app on two devices, it will show a Handoff icon when you have the document open. Flint uses your URL routes to automatically map the `NSUserActivity` to the action.

You’ll need to add a [single function call from Flint.swift](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Core/Flint.swift) to `application(:continue:restorationHandler:)` to pass the incoming activity to Flint, but that’s it! Of course you can customise the other properties of the `NSUserActivity` easily if you need to.

## Updating your app so it handles incoming activities

When the operating system asks your app to conitnue an activity from Handoff, Siri suggestions, Deep Linking or similar, it will call into
your application delegate and pass in the activity. You need to wire this up to Flint like so:

```swift
func application(_ application: UIApplication, continue userActivity: NSUserActivity, restorationHandler: @escaping ([Any]?) -> Void) -> Bool {
    return Flint.continueActivity(activity: userActivity, with: presentationRouter) == .success
}
```

You do need to update your `Info.plist` and the key `NSUserActivityTypes` to include the identifier for every activity you support. Flint generates automatic activity types using the app's bundle ID and action name (split from camel case into tokens), lowercased. In future we'll have a helper to output all the IDs for you so you can enter them into `Info.plist` but for now here's some examples of how the IDs map:

* A `BeepAction` in app with bundle id "co.montanafloss.test" would have ID `co.montanafloss.test.beep`
* A `SaveDocumentAction` in app with bundle id "co.montanafloss.test" would have ID `co.montanafloss.test.save document`

Note that Flint will crash with a `preconditionFailure` if it finds the activity ID for an automatic activity is not in the `Info.plist`. We are tryingto save you from yourself!

You can also explicitly set the ID yourself by implementing `prepare(:)` on your `Action`. Either way all the types have to be listed in your app’s `Info.plist`, and sadly Flint cannot do this for you.

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

Now when run, any document that is opened will automatically be registered for search indexing. Go to the home screen, pull down for search and type “Flint Demo” and you will find the document. Tap it and the app will open at that document. This is a slightly odd arrangement — you’d normally submit these items to Core Spotlight for indexing separately when loading your data store, and reference that from the `NSUserActivity` — but it demonstrates the possibilities, and Apple has support for this.

## Troubleshooting and testing

More docs coming soon

* Activities that also register the item for Spotlight search *must* have a title. Flint will alert you to this if you forget.
* If Handoff or other activity doesn't happen - what to check

## Next

* Add [Analytics](analytics.md) tracking
* Use the [Timeline](timeline.md) to see what is going on in your app when things go wrong
* Start using [Focus](focus.md) to pare down your logging
