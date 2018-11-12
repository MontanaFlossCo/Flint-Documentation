---
title: Siri Shortcuts
subtitle: Learn how to expose your Actions as Siri Shortcuts
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

Siri Shortcuts are introduced in iOS 12, with some support on watchOS 5. The term Shortcuts and the app Shortcuts are heavily overloaded and confusing, so it helps to be clear what is meant and what is possible.

1. Your app can expose Siri Shortcuts which iOS will use for prediction of what might be useful and allow users to trigger these same actions again at a later point, using either `NSUserActivity` or a custom `INIntent`. (This is called “donating” shortcuts)
2. Your app can offer a "suggested invocation phrase" so that when the user wants to create a new Siri voice-trigger or your action, there is a relevant phrase to prompt the user
3. Your app can implement a Siri Intents extension that will execute Intent-based shortcuts in the background without running your app, and either speak or show the results to the user, or direct them to open your app to continue the action somehow if there is a reason for this (e.g. high memory requirements)
4. Users can trigger your app's `NSUserActivity` or custom `INIntent`-based shortcut from within a Shortcuts app "shortcut", also known as a workflow, to create their own custom automations. App shortcuts that are not implemented in an Intent extension will open your app mid-workflow, thus terminating the workflow
5. Your app can "donate" specific "relevant shortcuts" that may be shown on the Siri watch face on Apple Watch based on time of day, location or user activity such as working out at the gym

Your app does not need to support all of these possibilities. As of Flint ea-1.0.4, you can easily expose `NSUserActivity`-based actions as shortcuts therefore allowing scenarios (1), (2) and (4). 

Support for Intent-based shortcuts is currently under development.

## Adding support for Siri Shortcuts

The simplest way to add basic support for iOS 12 Siri Shortcuts is to use Flint's [Activities](activities.md) to auto-publish an `NSUserActivity`. All you need to do is supply a suggested invocation phrase. The Action will then become visible to the user in the Siri Shortcuts section of the Settings app. You can also show the “Add Voice Shortcut” UI from your app to let the user create their voice shortcut without leaving your app.

To turn an `Action` that supports activities into an activity that the system can use for Siri prediction and voice shortcuts you need to:

1. Include `.prediction` in your Action’s `activityTypes`
2. Add a value for `suggestedInvocationPhrase` — or set this property on the activity in your Action’s `prepareActivity()` function
3. Optional: show the Add Voice Shortcut UI by calling `addVoiceShortcut(for:presenter:)`

Here’s an example of such an action:

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

Next, we’ll look at how to show the UI for adding a voice command for this action.

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

This will allow the user to create a shortcut to an `NSUserActivity` that will invoke the action with the `currentDocument` input supplied.

## Next steps

* Add [Analytics](analytics.md) tracking
* Use the [Timeline](timeline.md) to see what is going on in your app when things go wrong
* Start using [Focus](focus.md) to pare down your logging
