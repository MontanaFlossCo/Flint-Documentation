---
title: Contextual Logging
subtitle: Use Flint's logging to find out what users were actually doing when things go wrong.
tags:
    - coreconcepts
    - logging
    - debug
    - featured
---

#### In this article:
{:.no_toc}
* TOC
{:toc}

## Overview

Using Flint means that your app is aware of the actions the user is performing, and to which features those belong. This information makes logging much more useful by including the feature and action that culminated in each log entry. Your log message strings typically don't need to include as much context about what is happening because this context is automatically included.

## How Contextual Logging works

Every action that is performed is passed its own contextual logger interface, which encapsulates information about the context of the action. This includes the *feature to which it belongs*, as well as the *input* passed to the action.

There's also information about the original "user activity" that ultimately resulted in the log entry, essentially providing a form of conversational threading to logging. For example if you get an error in the log about sharing a document, it can show you that the start of this user activity was them importing a document in a specific format from another app. 

Note that because Flint has the ability to log when actions are performed, including the inputs passed, this eliminates a lot of logging you would normally write to debug what is happening in your UI. 

Contextual Logging in Flint is also built to allow you to **change logging levels at runtime** — typically while you are debugging — and to filter logging to include only information from specific features. You can even set log levels per feature. 

We strongly believe that statically compiled in logging thresholds make logging useless in many real-world cases — it's either the full firehose or nothing useful without recompiling. Runtime-enabled logging can increase compiled code size but the benefit in being able to capture logging without rebuilding is valuable.

## Yet another logging system?!

That’s a good question and the answer is a reluctant “sort of”. Flint’s logging can be adapted as a layer on top of whatever logging you already use. Flint's logging APIs allow your preferred logging subsystem to include this extra information so you can get far more contextual detail about e.g. why a network request is happening. 

If you carry the context logger from your actions through to your subsystems, you can disambiguate all this internal activity in your logs.

However, Flint has basic `stdout`, OSLog and File based logging supplied out of the box without any other dependencies.

## Development vs. Production logging

Flint has a simple new twist on logging to deal with the problem of logs that contain too much or too little information in production releases versus development.

In development you want as much as possible, but only on the topic you're working on (contextual logging!). In production you want as little as possible but it needs to be relevant to crash reporting and mustn't include confidential data.

Flint addresses this problem by separating the loggers for development and production. In your code you see which of the two is being used, so it is harder to make mistakes and easier to see what the intention of the logging is.

You can also nil the development logger and all development-level logging never even gets evaluated thanks to optional chaining.

## Using logging inside Actions

When actions are performed, the `perform` function receives a `context` parameter. This provides access to the logging via the `logs` property:

```swift
final class CancelPhotoSelectionAction: UIAction {
    typealias InputType = NoInput
    typealias PresenterType = PhotoSelectionPresenter
    
    public static func perform(context: ActionContext<NoInput>, presenter: PresenterType, completion: Completion) -> Completion.Status {
        // Trying to track down a problem in development?
        logs.development?.debug("The presenter we have is actually a \(String(reflecting: presenter))")
				
        // Trying to track down a problem in production?
        logs.production?.debug("CANCELLED!")
				
        presenter.dismissPhotoSelection()
        return completion.completedSync(.successWithFeatureTermination)
    }
}
```

You can take these loggers and pass them into subsystems your actions call into, to gain contextual logging deeper in your code.

## Using logging in subsystems where you don't have an Action

Often you need to perform logging from code that is not called as a result of an action, or where it is too cumbersome to
pass the logs from an `Action` all the way down the call stack. In this situation you can still log contextually relative to a feature.

Each feature type conforming to `Feature` or `ConditionalFeature` in your application provides a static function called `logs(for:)` that you can use to get a reference to contextual loggers:

```swift
import FlintCore

class NetworkClientService {
    let logs: ContextualLoggers    

    init() {
        logs = DataFeedFeature.logs(for: "Network")
    }    

    func fetchNewData() {
        logs.development?.info("Fetching new data from host \(host)")

        logs.production?.info("Fetch started")

        ...
    }
}
```

The same mechanisms apply as when you use the loggers from an Action's context — if a logger is not configured for the current build it will be nil and everything will be a no-op. Of course the loggers will have the context of the feature, and the activity string `"Network"` will be used as the current activity in the log entry.

## What is an "activity" in logging?

The logging system is designed to be decoupled from the rest of Flint. So in the logging subsystem's sense, the "activity" string associated with a `ContextSpecificLogger` is just a text description of "what the user or app was doing that led to this log entry".

For `ContextSpecificLogger` instances automatically created for actions by Flint, this activity is the description of the first action in the current [action stack](action_stacks.md).

If you obtain your own contextual logger from a log factory using `Logging.development?.contextualLogger(with:topicPath:)`, you must supply the activity description and topic path yourself. So for a background networking service you might use "Background fetching" as the activity, and a topic path `["Network"]`.

## What are topic paths?

Because the logging subsystem is decoupled from the rest of Flint, there is a concept of a `TopicPath` which identifies a hierarchical taxonomy for log entries.

Essentially it is an array of strings representing a hierarchical path, which can later be used to filter log entries to only certain portions of this hierarchy. e.g. logging to a topic path `["Networking", "Feeds", "Parsing"]` would mean you could set the log level for `["Networking"]` to `none` but set the parsing logging at path `["Networking", "Feeds", "Parsing"]` to `debug` level.

Flint maps a feature's `identifier` type `FeaturePath` to `TopicPath` under the hood, which is how we get log filtering by feature without the logging subsystem having a dependency on features. In future perhaps we'll spin it off into its own framework.

## Using the built in loggers

Flint ships with a simple `print` logger, a file logger, and an `OSLog` logging output implementation that sends entries to `Console.app`. 

A print logger is configured out of the box when you run `Flint.quickSetup` but you can customise this behaviour.

The Flint-supplied loggers all have options to change the text produced from log entries by specifying your own [`LogEventFormattingStrategy`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Logging/LogEventFormatttingStrategy.swift).

### Using the Print Logger

The `print` logger writes logging to the `stdout` of the process, and is only really useful in development.

A print logger is configured out of the box when you run `Flint.quickSetup` but if you want to change anything like the formatting or the prefix, you can manually set up Flint.

```swift
let printOutputDev = try! PrintLoggerOutput(prefix: "🐞 ", timeOnly: true)
Logging.setLoggerOutputs(development: [printOutputDev], level: .debug, production: nil, level: .off)
FlintAppInfo.associatedDomains = ["mysite.com"]
Flint.setup(AppFeatures.self)
```

The standard `PrintLoggerOutput` can render the full date time or just the time of the event, to reduce log noise. There are other initialisers that can use a custom date pattern, and one that takes a completely custom `LogEventFormatttingStrategy`.

### Using persistent File logging

File logging avoids clogging up your console but also allows you to capture log files for crash reporting or support using Flint's report gathering features.

To specify file-based logging, you'll need to configure your loggers at startup and setup Flint with them:

```swift
let fileOutputDev = try! FileLoggerOutput(appGroupIdentifier: nil, name: "myapp-dev")
let fileOutputProd = try! FileLoggerOutput(appGroupIdentifier: nil, name: "myapp-prod")
Logging.setLoggerOutputs(development: [fileOutputDev], level: .debug, production: [fileOutputProd], level: .off)
FlintAppInfo.associatedDomains = ["mysite.com"]
Flint.setup(AppFeatures.self)
```

☝️Note that if your app has extensions, such as a Siri Intent Extension where you will use Flint and want to capture logging, you'll need to use a shared app group container for your logs and pass the App Group Identifier to the file logger instances in the `appGroupIdentifier` argument. You'll want to use different log file names for each extension you have, so that if the extensions and apps run concurrently they do not have issues writing to the same file.

### Using OSLog for logging

The operating system's `os_log` API allows category-based logging to the system's aggregated logging system, which you can view using `Console.app`. Advantages include high performance, realtime UI and filtering, and basic grouping by topics as well as processes. You can see your app's logging in amongst everything the system is doing which can help troubleshoot some difficult integration issues.

You'll need to manually setup Flint instead of using `Flint.quickSetup`:

```swift
let osOutputDev = try! OSLogOutput()
Logging.setLoggerOutputs(development: [osOutputDev], level: .debug, production: nil, level: .off)
FlintAppInfo.associatedDomains = ["mysite.com"]
Flint.setup(AppFeatures.self)
```

## Exporting a Flint Debug Report ZIP

Flint’s debug-related features [Action Stacks](action_stacks.md), [Timeline](timeline.md) and the `LoggerOutput` implementations all integrate with the Debug Reporting subsystem. This enables you to gather a Zip file at any point containing all the information for export.

There’s support for human-readable and machine-readable formats where it makes sense — so you can build tools that take a JSON Timeline representation to build up a picture of usage patterns that lead to problems, or QA repro scripts.

To export a report:

```swift
let url = DebugReporting.gatherReportZip(options: [])
// You must delete this url when you are done
```

On iOS, to show a share sheet from your UI for this file you can do this:

```swift
@objc public func shareReport() {
    let url = DebugReporting.gatherReportZip(options: [])
    let shareViewController = UIActivityViewController(activityItems: [url], applicationActivities: nil)
    shareViewController.completionWithItemsHandler = { _, _, _, _ in
        try? FileManager.default.removeItem(at: url)
    }
    present(shareViewController, animated: true)
}
```

When extracted, the Zip can look something like this:

```
/flint-debug-report
	/action_stacks.txt
	/flintdemo-dev-2019-01-25.log
	/timeline.json
```

## Changing log levels at runtime

Documentation TBD. See [`Logging`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Logging/Logging.swift).

## Next steps

* See the [Focus](focus.md) feature of Flint to screen out everything from logs except the stuff that matters to you, at  runtime


