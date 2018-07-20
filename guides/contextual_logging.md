---
title: Contextual Logging
subtitle: Get more information about what log entries related to, and use smart filtering
tags:
    - coreconcepts
    - logging
    - debug
    - featured
---

## What is meant by Contextual Logging?

Using Flint means that your app is aware of the actions the user is performing, and to which features those belong. We make use of this information to make logging much more useful, by including the information about the feature and action that culminated in each log entry. Your log strings typically don't need to give as much content about what is happening because this is implied.

Every action that is performed is passed its own contextual logger interface, which encapsulates information about the context of the action. This includes the *feature to which it belongs*, as well as the *input* passed to the action. There's also information about the original "user activity" that ultimately resulted in the log entry, essentially providing a form of conversational threading to logging. For example if you get an error in the log about sharing a document, it can show you that the original start of this user activity was them importing a document in a format from another app.

Note that because Flint has the ability to log when actions are performed, including the inputs passed, this eliminates a lot of logging you would normally write to debug what is happening in your UI.

Context Logging in Flint is also built to allow you to change logging levels at runtime — typically while you are debugging — and to filter logging to include only information from specific features. You can even set log levels per feature. We strongly believe that statically compiled in logging thresholds make logging useless in most cases - it's either the full firehose or nothing useful without recompiling.

## Yet another logging system?!

No, not really. Just a thin layer on top of whatever logging you want to use. Flint's logging APIs allow your preferred logging subsystem to include this information so you can get far more contextual detail about e.g. why a network request is happening. 

If you carry the context logger from your actions through to your subsystems, you can disambiguate all this internal activity in your logs.

## Development vs. Production logging

Flint has a simple new twist on logging to deal with the problem of logs that contain too much or too little details in production releases versus development.

In development you want as much as possible, but only on the topic you're working on (contextual logging!). In production you want as little as possible but it needs to be relevant to crash reporting and mustn't include confidential data.

Flint addresses this problem by separating the loggers for development and production. In your code you see which of the two is being used to it is harder to make mistakes and easier to see what the intention of the logging is.

## Using logging inside Actions

When actions are performed, the `perform` function receives a `context` parameter. This provides access to the logging via the `logs` property:

```swift
final class CancelPhotoSelectionAction: Action {
    typealias InputType = NoInput
    
    typealias PresenterType = PhotoSelectionPresenter
    
    public static func perform(context: ActionContext<NoInput>, presenter: PresenterType, completion: @escaping (ActionPerformOutcome) -> Void) {

				// Trying to track down a problem in development?
				logs.development?.debug("The presenter we have is actually a \(String(reflecting: presenter))")
				
				// Trying to track down a problem in production?
				logs.production?.debug("CANCELLED!")
				
        presenter.dismissPhotoSelection()
        completion(.success(closeActionStack: true))
    }
}
```

You can take these loggers are pass them into subsystems your actions call into, to gain contextual logging deeper in your code.

## Using logging in subsystems where you don't have an Action

New stuff coming! TBD

## What is an "activity" in logging?

The logging system is designed to be decoupled from the rest of Flint. So in the logging subsystem's sense, the "activity" string associated with a `ContextSpecificLogger` is just a text description of "what the user or app was doing that led to this log entry".

For `ContextSpecificLogger` instances automatically created for actions by Flint, this activity is the description of the first action in the current [action stack](action_stacks.md).

If you obtain your own contextual logger from a log factory using `Logging.development?.contextualLogger(with:topicPath:)`, you must supply the activity description and topic path yourself. So for a background networking service you might use "Background fetching" as the activity, and a topic path `["Network"]`.

## What are topic paths?

Because the logging subsystem is decoupled from the rest of Flint, there is a concept of a `TopicPath` which identifies a hierarchical taxonomy for log entries.

Essentially it is an array of strings representing a hierarchical path, which can later be used to filter log entries to only certain portions of this hierarchy. e.g. logging to a topic path `["Networking", "Feeds", "Parsing"]` would mean you could set the log level for `["Networking"]` to `none` but set the parsing logging at path `["Networking", "Feeds", "Parsing"]` to `debug` level.

Flint maps a feature's `identifier` type `FeaturePath` to `TopicPath` under the hood, which is how we get log filtering by feature without the logging subsystem having a dependency on features. In future perhaps we'll spin it of into its own framework.

## Changing log levels at runtime

Documentation TBD. See [`Logging`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Logging/Logging.swift).

## Using the built in loggers

Flint ships with a simple `print` logger and an `OSLog` logging output implementation that sends entries to `Console.app`. These are configured out of the box when you run `Flint.quickSetup` but you can customise their behaviours.

For now, [see the source for details](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Core/Flint.swift#L120).

## Wiring up your own logging output

For now, please see how the [default logging is configured here](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Logging/DefaultLoggerFactory.swift#L68).

## Next steps

* See the [Focus](focus.md) feature of Flint to screen out everything from logs except the stuff that matters to you


