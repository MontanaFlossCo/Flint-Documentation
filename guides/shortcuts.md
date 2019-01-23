---
title: Siri Shortcuts
subtitle: Learn how to expose your Actions as Siri Shortcuts
tags:
    - integration
    - useractivity
    - intents
    - shortcuts
    - intents
    - featured
    - siri
---

#### In this article:
{:.no_toc}
* TOC
{:toc}

## Overview

Siri Shortcuts are introduced in iOS 12, with some support on watchOS 5. The `Action` pattern used throughout Flint provides all the benefits of contextual logging, analytics, permissions and feature flagging for Shortcuts and Intents.

The term Shortcuts and the app Shortcuts are heavily overloaded and confusing, so it helps to be clear what is meant and what is possible.

### Siri Predictions expose common actions in Search

When users perform actions in your app, you can expose Siri Shortcuts that iOS will use for predictions, allowing users to trigger the same actions at a later point, using either `NSUserActivity` to open your app, or a custom `INIntent` in a separate extension to your app.

### Voice Shortcuts allow users to trigger your actions with a custom phrase

Your app can offer a "suggested invocation phrase" so that if the user wants to create a new Siri voice-trigger for your action there is a relevant phrase to prompt the user. You can show the standard UI to add the shortcut and record their phrase.

### Intent Extensions perform your actions in the background

Your app can implement a Siri Intents extension that can execute Intent-based shortcuts in the background, without running your app, and either speak or show the results to the user, or direct them to open your app to continue the action somehow if there is a reason for this (e.g. high memory requirements).

### The Shortcuts app can invoke your actions inside workflows

Users can trigger your app's `NSUserActivity` or custom `INIntent`-based shortcut from within a Shortcuts app "shortcut", also known as a workflow, to create their own custom automations. App shortcuts that are not implemented in an Intent extension will open your app mid-workflow, thus terminating the workflow.

### Donating shortcuts to the system

Your app can "donate" specific shortcuts it thinks the user will find useful. This feeds the prediction system and the list of shortcuts available to the user in Shortcuts app and Settings.

### Show relevant shortcuts on Apple Watch

Your app can also donate "relevant shortcuts" that may be shown on the Siri watch face on an Apple Watch based on time, location or user activity.

### You choose what to support

Your app does not need to support all of these possibilities. As of Flint ea-1.0.4, you can easily expose `NSUserActivity`-based actions as shortcuts. Release ea-1.0.5 supports both activities as shortcuts and custom `INIntent` extensions.

For further details on how Shortcuts can be used on iOS, [see Apple's tech note](https://support.apple.com/en-gb/HT209055).

## Adding basic support for Siri Shortcuts using Activities

The simplest way to add basic support for iOS 12 Siri Shortcuts is to use Flint's [Activities](activities.md) to auto-publish an `NSUserActivity`. Once your `Action` is set up for this already, you have minimal work to do for the user to be able to trigger that action from Siri via a voice shortcut, Siri prediction or in the Shortcuts app.

All you need to do is supply a suggested invocation phrase and add `prediction` support if you want that. The Action will then become visible to the user in the Siri Shortcuts section of the Settings app. You can also show the “Add Voice Shortcut” UI from your app to let the user create a voice shortcut there and then.

These shortcuts will **always open your app** and Flint will dispatch them as it does [Activities](activities.md) — meaning you must either have a URL mapping for your action or rely on `NSUserActivity.userInfo` data to recreate your input via `ActivityCodable` support on your input type.

To turn an `Action` that supports activities into an activity that the system can use for Siri prediction and voice shortcuts you need to:

1. Include `.prediction` in your Action’s `activityEligibility`
2. Add a value for `suggestedInvocationPhrase` — or set this property on the activity in your Action’s `prepareActivity()` function
3. *Optional*: show the Add Voice Shortcut UI by calling `addVoiceShortcut(for:presenter:)` on the action binding
4. Make sure you have support for Activities in your app delegate; namely your `application(continueActivity:...)` implementation must call `Flint.continueActivity(...)` (see [Activities](activities.md))

We’ll also assume you have the [Activities](activities.md) feature of Flint enabled.

Here’s an example of such an action:

```swift
final class DocumentPresentationModeAction: UIAction {
    typealias InputType = DocumentRef
    typealias PresenterType = DocumentPresenter

    static var description = "Open a document"
    
    /// Include .prediction so we are listed on Siri search results
    static var activityEligibility: Set<ActivityEligibility> = [.prediction]
    
    /// Include the explicit String? type here,
    /// not doing so would use the wrong type.
    static let suggestedInvocationPhrase: String? = "It's show time"
    
    // … the reset of the Action elided
}
```

Once the user has performed this action, it will begin to show up in Siri suggestions and also in the Shortcuts app.

For testing, you can go to a device's Settings app and in the "Developer" section you will find a range of settings under the heading "Shortcuts Testing" that can show all the recently registered shortcuts, not just the ones Siri thinks are relevant. This is useful in debugging shortcut-related issues reliably.

## Showing the system UI for adding a voice shortcut

Once you have an action with a suggested voice phrase you can add code to your application that will let the user add a voice shortcut directly in your app. Flint will present the system UI to record their custom phrase, using your phrase as inspiration.

You call the Flint-provided `addVoiceShortcut(for:presenter:)` function on your feature's action binding and pass in the input the action should use with the shortcut and a `UIViewController` to present the UI.

Note that creating a shortcut to an action does so for a specific input to that `Action`. In iOS 12 you cannot create a shortcut that takes a different input each time. You can think of it as a frozen snapshot of an action you performed on a given input, that you can repeat later.

By way of example, if your feature is called `ProFeatures` and it has an action bound as `showInPresentationMode` you would show the Add Siri Voice Shortcut UI like this:

```swift
class YourViewController: UIViewController {
    var currentDocument: DocumentRef?

    func yourAddToSiriButtonTapped(_ sender: Any) {
        // Show the Siri UI for adding a voice shortcut for this specific document
        ProFeatures.showInPresentationMode.addVoiceShortcut(for: currentDocument!, presenter:self)
    }
}
```

You can call this function at any time to register shortcuts without actually performing the action at that point, for example in a Settings UI for your app that allows users to add shortcuts for common actions listed in your app.

Once the user has added a shortcut in this way, your app can be invoked by voice with Siri, or inside the Shortcuts app.

Note that this will allow the user to create a shortcut to an Action via `NSUserActivity` or `INIntent` that will invoke the action with the `documentRef` input supplied. For intent-based actions, it will always create a shortcut to the Intent if it can.

## Implementing a custom Siri Intent extension with Flint

If you want a Siri Shortcut to perform your `Action` without opening your app, you'll need to implement a Siri Intent Extension. It is important to understand the lifecycle of an Intent request, as there are various paths that can be taken.

When the user triggers a custom intent via Siri, Siri Suggestions or the Shortcuts app, your extension will be loaded and the appropriate intent handler for the intent type will be called. This handler can return a textual response to Siri that it can speak or display, use a custom Intents UI extension for display, or your intent handler can indicate that the user needs to continue the action inside your app.

In this latter case, common if permissions or login credentials are missing, Siri will offer to open the app, passing in the Intent information to your app so that your app can then continue the activity directly.
 
Flint provides conventions for creating the `INIntent` instance for a given `Action` input, and creating the input to the action from a received `INIntent` instance containing parameters. Your action needs to implemnent thes.

You can "donate" intent-based Actions to Siri explicitly with `donateToSiri(for:)` on the action binding, or automatically via the `associatedIntents` function on other `Action`s.

There’s a lot to cover there, so let’s break it down. Creating a custom Intent requires the following steps:

1. Add a new Intent Extension target to your app
2. Define a new `INIntent` type in Xcode, with an Intent Definition that declares the parameters and responses permitted with your intent
3. Add FlintCore as a dependency to your Intent Extension, and add any types or frameworks you need from your main application to the target. 
4. Define a new `Action` type that will perform the work of the `Intent`, inside the intent extension. This must conform to `IntentAction`, and will receive the intent instance as its output, and an `IntentResultPresenter`
5. Add code to the generated `IntentHandler` code to call into Flint to dispatch the intent
6. If your Intent may request the app to continue the activity, make sure you have `application(continueActivity:)` in your app delegate set to call `Flint.continueActivity(...)`

For steps 1 & 2 please see the [Apple Intents documentation](). 

### Creating an Action to perform the intent

Actions that can be performed in the background as an Intent need to indicate how they would like the system to present their results. As such you cannot reuse an existing app `Action` for an Intent directly — the presenter type is different to what you would normally use.

An intent action type conforms to `IntentAction`, which guarantees the correct presenter type and threading behaviour for intent extensions which are executed on a background thread so they do not block the Siri UI while fetching data for example.

In the following example, the intent Action takes a document reference (ID) as input and specifies the type of intent and response it is associated with. This type is used to convert to and from the input type of the action and the parameters of the Intent type.

```swift
import FlintCore

/// Implement the GetNoteIntent, taking a document name as input and showing the document contents via Siri
final class GetNoteAction: IntentAction {
    typealias InputType = DocumentRef
    typealias IntentType = GetNoteIntent
    typealias IntentResponseType = GetNoteIntentResponse
    
    enum Failure: Error {
        case documentNotFound
    }
    
    /// Return an intent instance to execute the action with
    /// the supplied input. Used when donating intents that invoke this action.
    static func intent(for input: DocumentRef) -> GetNoteIntent? {
        let result = GetNoteIntent()
        result.documentName = input.name
        result.setImage(INImage(named: "GetNoteIcon"), forParameterNamed: \.documentName)
        return result
    }
    
    /// Return an input instance to pass to the action with
    /// when the intent is received. Used when executing the action when the user performs the Intent
    /// via an Intent extension.
    static func input(for intent: GetNoteIntent) -> DocumentRef? {
        guard let name = intent.documentName else {
            return nil
        }
        let ref = DocumentRef(name: name, summary: nil)
        return ref
    }

    /// Perform the action and use the Siri Intent presenter to indicate the response type.
    /// The available responses are configured in your Intent's definition file in Xcode.
    static func perform(context: ActionContext<InputType>, presenter: GetNoteAction.PresenterType, completion: Completion) -> Completion.Status {
        let response: GetNoteIntentResponse
        let outcome: ActionPerformOutcome
        if let document = DocumentStore.shared.load(context.input.name) {
            response = .success(content: document.body)
            outcome = .successWithFeatureTermination
        } else {
            // If they are to continue in-app we can pass an activity that will be used to continue in the app, but here we're treat it as an error.
            response = GetNoteIntentResponse(code: .failure, userActivity: nil) 
            outcome = .failureWithFeatureTermination(error: Failure.documentNotFound)
        }
        
        // Tell Siri what the response is
        presenter.showResponse(response)
        
        return completion.completedSync(outcome)
    }
}
```

Note that the `PresenterType` for `IntentAction`-conforming actions is automatically set to a `IntentResponsePresenter<IntentResponseType>`. This means that your action receives a presenter with a single `showResponse()` function that you call with one of the `INIntentResponse` instances that are valid for this intent. The types automatically generated by Xcode have convenience functions for each response you define in your Intent definition file.

Separate from the Intent response, your action also needs to participate in Flint’s normal completion status handling as shown above, so that the caller can adapt its behaviour.

**It is important to understand** that IntentAction perform calls take place on a non-main queue, and they may complete asynchronously if desired.

### Make your Intent Extension use Flint to dispatch the Action

Now that you have an `Action` that can perform the work of your intent, you have an Intent Extension target in your project and your Intent definition file all together, all you need to do is make the Intent extension perform the action.

Your Intent Extension has a single entry point, the Intent Handler that extends `INExtension` from the `Intents` framework  on iOS. You need to edit this to set up Flint and return an intent handler instance appropriate for each Intent you support:

```swift
import Intents
import FlintCore

var hasRunFlintSetup = false

class IntentHandler: INExtension {
    override init() {
        super.init()
        if !hasRunFlintSetup {
            // Pass in the FeatureGroup you want to have
            // access to in your extension. This is usually
            // a small subset of your Features to minimise 
            // dependencies.
            Flint.quickSetup(IntentFeatures.self)
            hasRunFlintSetup = true
        }
    }

    override func handler(for intent: INIntent) -> Any {
        // Return an instance of the handler type Xcode
        // generated for your given intent.
        switch intent {
            case is GetNoteIntent: return GetNoteIntentHandler()
            default: fatalError("Unknown intent type: \(intent)")
        }
    }
}
```

With this code in your extension, all that is left to do is add code to each intent’s handler. For the `GetNoteIntentHandler` in the example above, this would be simply:

```swift
import Foundation
import Intents
import FlintCore

@objc
class GetNoteIntentHandler: NSObject, GetNoteIntentHandling {
    @objc(handleGetNote:completion:)
    func handle(intent: GetNoteIntent, completion: @escaping (GetNoteIntentResponse) -> Void) {
        let outcome = SiriFeature.getNote.perform(intent: intent, completion: completion)
        assert(outcome == .success, "Intent failed: \(outcome)")
    }
}
```

Flint’s `perform(intent:completion:)` function exists on all your `IntentAction`-conforming actions’ bindings. It will create an `IntentResultPresenter` that is passed to your action’s `perform(coontext:,presenter:,completion:)` function so that your action can set the appropriate intent response.

## Automatically donating shortcuts when other `Action`s are performed

Much like the Activities feature of Flint, the Intent support can auto-donate one or more Intent(s) each time an `Action` is performed in your app.

For example a Podcast player app might want to donate both a "Toggle trim silence for Connected podcast" shortcut and a "Toggle fast speed for Connected podcast" shortcut, when you play one episode of "Connected" podcast. This obviously shouldn't be abused, but can prove very useful as people increasingly use automation with Shortcuts app.

All you need to do to achieve this is to add a function `associatedIntents(for:)` to your `Action` implementation:

```swift
final class PlayPodcastAction: UIAction {
    typealias InputType = PodcastID
    typealias PresenterType = PodcastPlayer

    // The donateToSiri() functionality will call this. 
    @available(iOS 12, *)
    static func associatedIntents(for input: InputType) -> [FlintIntent]? {
        let trimSilenceIntent = ToggleTrimSilenceIntent()
        trimSilenceIntent.podcast = input
        trimSilenceIntent.setImage(INImage(named: "ToggleSilenceIcon"), \.podcast)

        let fastSpeedIntent = ToggleFastSpeedIntent()
        fastSpeedIntent.podcast = input
        fastSpeedIntent.setImage(INImage(named: "ToggleFastIcon"), \.podcast)

        return [trimSilenceIntent, fastSpeedIntent]
    }
    ...
}
```

This will be called whenever the action is performed, and each of the resulting intents is donated.

## Donating shortcuts explicitly to Siri

Some apps provide shortcuts that are always meaningful but don't necessarily need a voice phrase set up with them. You might register common actions that you want to be visible in the Shortcuts app for example.

To do this you "donate" the shortcuts explicitly. In Flint this is easy — you just call `donateToSiri` on the relevant `Action` binding:

```swift
/// Automatically register the "play and sleep for 50 minutes" intent with Siri
SiriFeatures.playPodcastAndSleep.donateToSiri(for: 50)
```

This will allow the user to create a shortcut to an `NSUserActivity` that will invoke the action with the `currentDocument` input supplied.

## Declaring "relevant shortcuts" for Siri support on Apple Watch

Coming soon!

## Next steps

* Add [Analytics](analytics.md) tracking
* Use the [Timeline](timeline.md) to see what is going on in your app when things go wrong
* Start using [Focus](focus.md) to pare down your logging
