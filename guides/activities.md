---
title: Activities
subtitle: Flint's automatic Activities feature will publish actions as the current NSUserActivity and will continue these activities later, dispatching your actions for you
tags:
    - integration
    - useractivity
    - actions
    - featured
    - siri
---

#### In this article:
{:.no_toc}
* TOC
{:toc}

## Overview

Apple platforms use `NSUserActivity` to tell the operating system about something the user is doing in your app. This information is used across the platforms to improve the user experience. This includes support for Handoff, Siri Suggestions (AKA Siri Pro-active), Spotlight Search, Siri Intents, deep linking and even ClassKit for education apps.

From iOS 12 onward it also feeds this information into Siri Shortcuts, which will use machine learning to suggest appropriate activities you have performed in the past based on your location, time of day and other data.

When you have actions defined using Flint, you get support for all of these with very little extra code required.

The Activities feature of Flint will automatically register an `NSUserActivity` for you when users perform a qualifying action in your app. Flint observes the execution of actions and does the right thing for you so you no longer have to think about this at the call site where actions are performed. What's more it leverages your existing Action(s) and input types, with type safety when performing incoming activities and increased decoupling of your code.

With Activities you can:

* Automatically publish `NSUserActivity`
* Set the `NSUSerActivity` eligibility including options for  `handoff`, `search` and `prediction` (iOS 12 only)
* Define a suggested invocation phrase for activities that you would like to expose as Siri Shortcuts (iOS 12 only)
* Override activity attributes per Action and input the action receives
* Define activity attributes specific to the value of the `InputType` to your Actions, so input types can be reused across actions that publish activities in different ways
* Easily set custom Spotlight search attributes

You determine which actions are published as activities. As an example, “Save” is not something that makes sense for a Handoff action, but opening a document is.

Flint will not automatically start publishing all your actions as activities. To enable this behaviour you need to declare one or more `activityTypes` on an action and it will have an `NSUserActivity` published. Your action’s `InputType` if there is one also needs to conform to `ActivityCodable`.

The other side of activities is receiving them in your app and performing the appropriate action. You need to provide a way to describe how to invoke your action from the `NSUserActivity` at a later a time. This means Flint needs to know:

1. Which `Action` to invoke
2. How to reconstruct an instance of the `InputType` for your action from the `NSUserActivity`'s `userInfo`
3. How to obtain an instance of the `PresenterType` for the action to use, so the UI can be put into the correct state for your action

In fact you only need to do parts (1) and (3) if your action is already mapped to a URL using [URL Routes](routes.md) because in that case Flint already knows how to represent your action’s input as a URL, and can embed the URL in the activity.

To start using Activities in the most basic way, all you need to do is update actions to indicate what kind of activity they should expose.

Note that the `Activities` feature of Flint is on by default but can be disabled if you do not want it, or you need to debug scenarios where `NSUserActivity` is getting in the way somehow — just set `ActivitiesFeature.isEnabled = false`.

## Enabling Activities on an Action

Let’s add the declaration for a new “Document Open” action that supports just Handoff and Siri Prediction. Note that registering an activity implies pre-iOS 12 Siri Suggestions support, so it is implied by declaring `handoff`. Adding `prediction` is a more specific indication to Siri that it should use your activity and userInfo, location and time of day to provide useful recommendations to users.

Flint currently supports the following values for activity eligibility, which you can combine as you wish:

* `.perform` - Minimal support, just register the activity. This is implied by all other types so is not required normally.
* `.handoff` - Handoff support between devices
* `.search` - Include the activity in Spotlight search indexing
* `.prediction` - Use the activity for iOS 12 Siri predictions

So we’ll define a value for the static `activityTypes` property:

```swift
final class DocumentOpenAction: UIAction {
    typealias InputType = DocumentRef
    typealias PresenterType = DocumentPresenter

    static var description = "Open a document"
    
    /// Declare the types of activity we want published.
    /// This is the bare minimum we need to add.
    static var activityTypes: Set<ActivityEligibility> = [.handoff, .prediction]
    
    static func perform(with context: ActionContext<InputType>, using presenter: PresenterType, Completion) -> Completion.Status {
        // … here we would open the document as normal
    }
}
```

If you have already mapped this action with URL Routes in a `URLMapped` feature, when the app is run it will automatically publish an `NSUserActivity` when that action occurs, and if you run the app on two devices, it will show a Handoff icon on the other device when you have the document open. Flint uses your URL routes to automatically map the `NSUserActivity` to the action in this scenario.

We will see a little later how to customise properties of the activity such as the `title`, `subtitle` and `thumbnail` which are used for some of the eligibility types — e.g. search results should have a subtitle and thumbnail for good results.

For these actions to be invoked when activities come in to the app, you need to add a call into Flint to your app delegate. First we'll look at how to support activities even if you don't want to map your actions to URLs. 

## Activities for Actions that don't have URL mappings

In the case where you don't need or want URL mappings for your actions but you still want to support `NSUserActivity`, it is easy to express your action's input as `userInfo` set on the activity instance by making your action's `InputType` conform to `ActivityCodable`.

In our example the action's `InputType` is `DocumentRef`. We can add an extension to that type to implement the three requirements for expressing the input as `userInfo`:


```swift
extension DocumentRef: ActivityCodable {
    static let documentKey = "document"
    
    enum DocumentError: Error {
        case missingUserInfoKey(name: String)
    }
    
    // Specify the info keys that need to be persisted by
    // the OS, and given back to us later.
    var requiredUserInfoKeys: Set<String> {
        return [DocumentRef.documentKey]
    }
    
    init(activityUserInfo: [AnyHashable:Any]?) throws {
        guard let name = activityUserInfo?[DocumentRef.documentKey] as? String else {
            throw DocumentError.missingUserInfoKey(name: DocumentRef.documentKey)
        }
        self.init(name: name)
    }
    
    func encodeForActivity() -> [AnyHashable:Any]? {
        return [DocumentRef.documentKey: name]
    }
}
```

This simple mechanism allows Flint to read and write this input type to and from an `NSUserActivity`, and you can reuse this input type across multiple action types.

Flint will automatically detect if your action's input type conforms to this protocol and use it to set the `userInfo` on the activity when publishing it, and to create the input from `userInfo` when asked to perform your action when your app receives an `NSUserActivity` for the `activityType` that maps your action.

With the above changes the initial example would now work without any URL mappings. You can also then reuse the same input type in other actions, without any more code to support this.

Note that Flint applies sanity checks to make sure you specify values for every `requiredUserInfoKey`, eliminating a class of hard to identify bugs where activities just don't work reliably.

## Updating your app so it handles incoming activities

When the operating system asks your app to continue an activity from Handoff, Siri suggestions, Deep Linking or similar, it will call into your application delegate and pass in the activity to `application(:continue:restorationHandler:)`.

All that you need to add is a [call to a single function on the Flint type](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Core/Flint.swift#L283) to `application(:continue:restorationHandler:)` to pass the incoming activity to Flint.

Here's how you do this in your app delegate:

```swift
func application(_ application: UIApplication, continue userActivity: NSUserActivity, restorationHandler: @escaping ([Any]?) -> Void) -> Bool {
    return Flint.continueActivity(activity: userActivity, with: presentationRouter) == .success
}
```

You may be wondering what the `presentationRouter` is in this code. This is a value you provide that conforms to Flint's `PresentationRouter` type. It is responsible for looking at the type of action to invoke and returning the correct type of presenter the action requires. This is how we handle the problem of "my app in currently showing some UI and this action requires a completely different state". See the example in the [Routes guide](https://github.com/MontanaFlossCo/Flint-Documentation/blob/master/guides/routes.md#adding-the-code-your-app-needs-to-handle-urls-and-present-the-ui) which also leverages the same mechanism.

Other than this, you do also need to update your `Info.plist` to update the key `NSUserActivityTypes` to include the identifiers for every activity you support. When you start supporting many kinds of activity because it is now so simple to do so, keeping on top of this can be difficult.

However, Flint also helps with this. First, it generates automatic activity type IDs using the app's bundle ID, feature and action name (split from camel case into tokens), all using lowercased "snake case". In future we'll have a helper to output all the IDs for you so you can enter them into `Info.plist` but for now here's some examples of how the IDs map:

* A `BeepAction` of feature `SoundsFeature` in an app with bundle id "co.montanafloss.test" would have ID `co.montanafloss.test.sounds.beep`
* A `SaveDocumentAction` of feature `DocumentManagementFeature` in an app with bundle id "co.montanafloss.test" would have ID `co.montanafloss.test.document-management.save-document`

Note that Flint will deliberately crash at startup with a helpful message if it finds the activity ID for an automatic activity is not in the `Info.plist`, so you don't forget to keep this updated.

## Customising the attributes on an Activity

There are two options for customising the attribute values passed to the system when activities are published, and you can even mix the two.

The first is to implement `prepareActivity(_ activity: ActivityBuilder<InputType>)` on your Action type. This is called when the activity is being created, and passed a builder that can safely create the activity based on the defaults Flint has already established for you. The builder has a reference to the `input` of the action for which an activity is being created, which you can use when customising the activity.

Note that the builder has already taken into account if your Action input type conforms to `ActivityCodable` and carries over the appropriate `userInfo` settings, and if not will attempt to use URL mapping to invoke your action. It will warn you if neither of these is possible, to prevent obscure bugs where you expose activities that cannot be invoked later.

```swift
final class DocumentOpenAction: Action {
    typealias InputType = DocumentRef
    typealias PresenterType = DocumentPresenter

    static var description = "Open a document"
    
    static var activityTypes: Set<ActivityEligibility> = [.handoff, .search]

    static func perform(with context: ActionContext<DocumentRef>, using presenter: DocumentPresenter, completion: Completion) -> Completion.Status {
        // …
    }
    
    static func prepareActivity(_ activity: ActivityBuilder<InputType>) {
        guard let document = DocumentStore.shared.documentInfo(for: activity.input) else {
            // Don't publish the activity after all, doc missing
            activity.cancel()
            return
        }

        activity.title = "Open \(document.name)"
        // Required for better search results
        activity.subtitle = document.summary
        activity.thumbnail = UIImage(named: "DocumentSearchResult")

        activity.keywords = "always be coding"
        activity.searchAttributes.creationDate = Date()
    }
}
```

The builder is passed the appropriate input to your Action, having reconstructed it from a URL or `userInfo`, and you can access this within `prepareActivity` to set the activity properties.

Properties you can access on the builder include:

* `title`
* `subtitle` — sets `contentDescription` on the search attributes
* `keywords` 
* `thumbnail`, `thumbnailURL` and `thumbnailData` — three options for specifying a thumbnail for search results and Siri suggestions
* `searchAttributes` — direct access to the full `CSSearchableItemAttributeSet` properties for advanced usage

However we can go one better. Often the input type itself will be able to provide the metadata required for the activity attributes, and this will lead to more code reuse. You can make an Action's input type conform to [`ActivityMetadataRepresentable`](https://github.com/MontanaFlossCo/Flint/blob/e4896c4dc43a6df52ddb893051a6e177f5fe7e97/FlintCore/Activities/ActivityMetadataRepresentable.swift), and Flint will detect this and automatically populate the activity properties from these values.

You can then mix these two approaches so that `prepareActivity` has a short, clean implementation specific to the Action, and the input-based metadata population is done automatically and reusable across actions.

Here's an extension to provide the metadata implementation on the input type:

```swift
extension DocumentRef: ActivityMetadataRepresentable {
    var metadata: ActivityMetadata {
        let searchAttributes = CSSearchableItemAttributeSet()
        searchAttributes.addedDate = Date()
        searchAttributes.kind = "Flint Demo Document"

        return .build {
            $0.title = name
            $0.thumbnail = UIImage(named: "FlintDemoDocIcon")
            $0.keywords = Set("decentralised internet patent".components(separatedBy: " "))
            $0.searchAttributes = searchAttributes
        }
    }
}
```

The metadata is created using a builder provided on the [`ActivityMetadata`](https://github.com/MontanaFlossCo/Flint/blob/e4896c4dc43a6df52ddb893051a6e177f5fe7e97/FlintCore/Activities/ActivityMetadata.swift) type. Custom search attributes can also be configured as shown. This then simplifies the `prepareActivity` implementation.

Any `metadata` extracted from the input is available in an optional `metadata` property of the activity builder. The values from the metadata are used to populate the activity, and then your function can further customise it with the builder.

Note that some properties are not always appropriate to pass straight through to the activity. In particular, the `title` of a document returned as `metadata` needs to have a verb appended to it in the title of the activity, so that the result is a clear call to action.

In this example we'll fix the `title` of the activity to include the verb `Open`, and not just the default `title` from the metadata, and all the rest — search attributes, keywords, thumbnail and so on are automatically populated from the input's metadata:

```swift
    static func prepareActivity(_ activity: ActivityBuilder<InputType>) {
        guard let document = DocumentStore.shared.documentInfo(activity.input.name) else {
            activity.cancel()
            return
        }

        activity.title = "Open \(activity.metadata!.title)"
        activity.subtitle = document.summary
    }
```

This then shows a Siri result with the title "Open MyProject" instead of just "MyProject".

## Adding support for Siri Shortcuts

The simplest way to add basic support for iOS 12 Siri Shortcuts is to use activities. All you need to do is supply a suggested invocation phrase. The Action will then become visible to the user in the Siri Shortcuts section of the Settings app. You can also show the “Add Voice Shortcut” UI from your app to let the user create their voice shortcut there and then.

To turn an `Action` that supports activities into an activity that the system can use for Siri prediction and voice shortcuts you need to:

1. Include `.prediction` in your Action’s `activityTypes`
2. Add a value for `suggestedInvocationPhrase` — or set this property on the activity in your Action’s `prepareActivity()` function
3. Optional: show the Add Voice Shortcut UI by calling `addVoiceShortcut(for:presenter:)`

Here’s an example:

```swift
final class DocumentPresentationModeAction: UIAction {
    typealias InputType = DocumentRef
    typealias PresenterType = DocumentPresenter

    static var description = "Open a document"
    
    /// Include .prediction
    static var activityTypes: Set<ActivityEligibility> = [.handoff, .prediction]
    
    /// Include the explicit String? type here,
    /// not doing so would use the wrong type.
    static let suggestedInvocationPhrase: String? = "Presentation time"
    
    // … the reset of the Action elided
}
```

Once you have done this, you can add code to your application that will let the user add a voice shortcut directly in your app, using the system UI to record their custom phrase:

```swift
func yourAddToSiriButtonTapped(_ sender: Any) {
    // Pass self (a UIViewController) as the parent view controller
    ProFeatures.showInPresentationMode.addVoiceShortcut(for: documentRef, presenter:self)
}
```

This will allow the user to create a shortcut to an `NSUserActivity` that will invoke the action with the `documentRef` input supplied.

## Next steps

* Add [Analytics](analytics.md) tracking
* Use the [Timeline](timeline.md) to see what is going on in your app when things go wrong
* Start using [Focus](focus.md) to pare down your logging
