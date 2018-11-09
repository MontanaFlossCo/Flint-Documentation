---
title: Analytics
subtitle: Track events when actions are performed using your preferred analytics back end
tags:
    - integration
    - actions
    - featured
---

#### In this article:
{:.no_toc}
* TOC
{:toc}

## Overview

Flint supports pluggable analytics tracking so that when the user performs actions, events relating to them are automatically recorded by your chosen analytics service.

All that you need to do to support this is provide an ID for the actions that should generate events, and optionally provide a way to encode the action's `input` in a way that makes sense for your analytics service.

## Making an Action tracked

The `Action` protocol defines a static property `var analyticsID: String?` that you can define to supply the event ID you want. Once you do this, Flint will automatically start generating analytics information for that action and passing it to your analytics provider.

```swift
final class DocumentCreateAction: UIAction {
    typealias InputType = NoInput
    typealias PresenterType = DocumentCreatePresenter

    static var description = "Create a new document"

    /// Set this to your desired analytics event ID for this action
    static var analyticsID: String? = "document-create"

    static func perform(with context: ActionContext<NoInput>, using presenter: DocumentCreatePresenter, completion: Completion) -> Completion.Status {
        presenter.showCreate(suggestedTitle: "Untitled")
        return completion.completedSync(.success)
    }
}
```

## Customising the information that is tracked for an Action

If you need to capture some information about the input passed to the action, you need to do a little more work. On the action you wish capture, implement the `analyticsAttributes` function to encode values into a dictionary that will be passed to the analytics provider:

```swift
final class DocumentOpenAction: UIAction {
    typealias InputType = DocumentRef
    typealias PresenterType = DocumentEditingPresenter

    static var description = "Open a document"

    static var analyticsID: String? = "document-open"

    /// Customize the data sent to analytics
    static func analyticsAttributes<F>(for request: ActionRequest<F, Self>) -> [String:Any?]? {
        return [
            "doc-type": request.context.input.documentTypeUTI
        ]
    }

    static func perform(with context: ActionContext<DocumentRef>,
            using presenter: DocumentCreatePresenter,
            completion: Completion) -> Completion.Status {
        presenter.openDocument(context.input)
        return completion.completedSync(.success)
    }
}
```

## Plugging in an Analytics service

To actually output anything, you have to pass Flint an instance of `AnalyticsProvider` that will receive all the events and do whatever is needed to log them to your backend. There is a toy provider included for testing, which simple logs analytics events to the system output. 

You can set this up at application start up:

```swift
let myAnalyticsProvider = ConsoleAnalyticsProvider()
Flint.dispatcher.add(observer: AnalyticsReporting(provider: myAnalyticsProvider))
```

The `AnalyticsReporting` type observes the dispatch of actions and calls into your provider with all the required parameters.

See the source of [`AnalyticsProvider`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Analytics/AnalyticsProvider.swift) for the interface you need to implement for your own provider.

## Next steps

* Try the [Flint UI debug tools](flint_ui.md)
* Use the [Timeline](timeline.md) to see what is going on in your app when things go wrong
* Start using [Focus](focus.md) to pare down your logging
