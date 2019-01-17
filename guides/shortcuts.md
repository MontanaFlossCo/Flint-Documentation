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

Siri Shortcuts are introduced in iOS 12, with some support on watchOS 5. The term Shortcuts and the app Shortcuts are heavily overloaded and confusing, so it helps to be clear what is meant and what is possible.

* When users perform actions in your app, you can expose Siri Shortcuts which iOS will use for predictions and allow users to trigger further actions at a later point, using either `NSUserActivity` in your app or a custom `INIntent` in a separate extension to your app
* Your app can offer a "suggested invocation phrase" so that when the user wants to create a new Siri voice-trigger or your action there is a relevant phrase to prompt the user
* Your app can implement a Siri Intents extension that can execute Intent-based shortcuts in the background, without running your app, and either speak or show the results to the user, or direct them to open your app to continue the action somehow if there is a reason for this (e.g. high memory requirements)
* Users can trigger your app's `NSUserActivity` or custom `INIntent`-based shortcut from within a Shortcuts app "shortcut", also known as a workflow, to create their own custom automations. App shortcuts that are not implemented in an Intent extension will open your app mid-workflow, thus terminating the workflow
* Your app can "donate" specific shortcuts it thinks the user will find useful
* Your app can also donate "relevant shortcuts" that may be shown on the Siri watch face on Apple Watch based on time, location or user activity

Your app does not need to support all of these possibilities. As of Flint ea-1.0.4, you can easily expose `NSUserActivity`-based actions as shortctus. Release ea-1.0.5 supports both activities as shortcuts and custom `INIntent` extensions.

Your app does not need to support all of these possibilities. As of Flint ea-1.0.4, you can easily expose `NSUserActivity`-based actions as shortcuts therefore allowing scenarios (1), (2) and (4). 

For further details on how Shortcuts can be used on iOS, [see Apple's tech note](https://support.apple.com/en-gb/HT209055).

## Adding basic support for Siri Shortcuts using Activities

The simplest way to add basic support for iOS 12 Siri Shortcuts is to use Flint's [Activities](activities.md) to auto-publish an `NSUserActivity`. All you need to do is supply a suggested invocation phrase. The Action will then become visible to the user in the Siri Shortcuts section of the Settings app. You can also show the “Add Voice Shortcut” UI from your app to let the user create their voice shortcut there and then.

These shortcuts will always open your app and Flint will dispatch them.

To turn an `Action` that supports activities into an activity that the system can use for Siri prediction and voice shortcuts you need to:

1. Include `.prediction` in your Action’s `activityTypes`
2. Add a value for `suggestedInvocationPhrase` — or set this property on the activity in your Action’s `prepareActivity()` function
3. Optional: show the Add Voice Shortcut UI by calling `addVoiceShortcut(for:presenter:)`
4. Make sure you have support for Activities in your app delegate; namely your `application(continueActivity:...)` implementation must call `Flint.continueActivity(...)` (see [Activities](activities.md))

Here’s an example of such an action:

```swift
final class DocumentPresentationModeAction: UIAction {
    typealias InputType = DocumentRef
    typealias PresenterType = DocumentPresenter

    static var description = "Open a document"
    
    /// Include .prediction so we are listed on Siri search results
    static var activityTypes: Set<ActivityEligibility> = [.prediction]
    
    /// Include the explicit String? type here,
    /// not doing so would use the wrong type.
    static let suggestedInvocationPhrase: String? = "Presentation time"
    
    // … the reset of the Action elided
}
```

Once the user has performed this action, it will begin to show up in Siri suggestions and also in the Shortcuts app.

For testing, you can go to a device's Settings app and in the "Developer" section you will find a range of settings under the heading "Shortcuts Testing" that can show all the recently registered shortcuts, not just the ones Siri thinks are relevant. This is useful in debugging shortcut-related issues reliably.

## Showing the system UI for adding a voice shortcut

Once you have an action with a suggested voice phrase you can add code to your application that will let the user add a voice shortcut directly in your app. Flint will present the system UI to record their custom phrase, using your phrase as inspiration.

You call the Flint-provided `addVoiceShortcut(for:presenter:)` function on your feature's action and pass in the input the action should use with the shortcut and a `UIViewController` to present the UI.

So if your feature is called `ProFeatures` and it has an action bound as `showInPresentationMode` you would show the Add Siri Voice Shortcut UI like this:

```swift
class YourViewController: UIViewController {
    var currentDocument: DocumentRef?

    func yourAddToSiriButtonTapped(_ sender: Any) {
        // Show the Siri UI for adding a voice shortcut for this specific document
        ProFeatures.showInPresentationMode.addVoiceShortcut(for: currentDocument!, presenter:self)
    }
}
```

This will allow the user to create a shortcut to an `NSUserActivity` or `INIntent` that will invoke the action with the `documentRef` input supplied. You can call this function at any time to register shortcuts without actually performing the action at that point, for example in a Settings UI for your app that allows users to add shortcuts for common actions listed in your app.

Once the user has added a shortcut in this way, your app can be invoked by voice with Siri, or inside the Shortcuts app.

## Implementing a custom Siri Intent extension with Flint

If you want a Siri Shortcut to perform your `Action` without opening your app, you'll need to implement a Siri Intent Extension. It is important to understand the lifecycle of an Intent request, as there are various paths that can be taken.

When the user triggers a custom intent via Siri, Siri Suggestions or the Shortcuts app, your extension will be loaded and the appropriate intent handler for the intent type will be called. This handler can return a response to Siri that it can speak or display, or your intent handler can indicate that the user needs to continue the action inside your app — and Siri will offer to open the app, passing in the Intent information to your app so that your app can then continue the activity directly.

Flint will handle both creating the `INIntent` instance for a given `Action`, in order to "donate" it to Siri explicitly or via `NSUserActivity` if you are using the [Activities](activities.md) feature of Flint

Creating a custom Intent requires:

1. Adding a new Intent Extension target to your app
2. Defining a new `INIntent` type in Xcode, with an Intent Definition that declares the parameters and responses permitted with your intent
3. Defining a new `Action` type that will perform the work of the `Intent`, inside the intent extension. This must conform to `IntentAction`, and will receing the intent instance as its output, and an `IntentResultPresenter`
4. Mapping the intent type to your `Action`
5. Adding code to the generated `IntentHandler` code to call into Flint to dispatch the intent
6. If your Intent may request the app to continue the activity, make sure you have `application(continueActivity:)` in your app delegate set to call `Flint.continueActivity(...)`

For steps 1 & 2 please see the [Apple Intents documentation](). 

### Creating an Action to perform the intent

Coming Soon. TL:DR; Make a type that conforms to `IntentAction`, and a custom presenter type.

### Associating your intent action with Actions in your App

If you want this Intent action to be associated with another `Action` the user can perform in your app, just like [Activities](activities.md) are automatically registered with the system, you need to make your other action implement the `intent(for:)` function to return the appropriately configured `INIntent` instance.

### Make your Intent Extension use Flint to dispatch the Action

Coming Soon. TL:DR; Make your App Delegate call `Flint.performIntentAction(...)`

## Automatically donating shortcuts when other `Action`s are performed

Much like the Activities feature of Flint, the Intent support can auto-donate one or more Intent(s) each time an `Action` is performed in your app.

For example a Podcast player app might want to donate both a "Toggle trim silence for Connected podcast" shortcut and a "Toggle fast speed for Connectted podcast" shortcut, when you play one episode of "Connected" podcast. This obviously shouldn't be abused, but can prove very useful as people increasingly use automation with Shortcuts app.

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
