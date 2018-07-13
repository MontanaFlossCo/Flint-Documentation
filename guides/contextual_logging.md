---
title: Contextual Logging
subtitle: Get more information about what log entries related to, and use smart filtering
tags: guide debug
---

Since the framework is aware of the actions the user is performing, and to which features those belong, it is able to provide a powerful logging concept we call "contextual logging".

Every action that is performed is passed its own contextual logger interface, which encapsulates information about the contact of the action. This includes the *feature to which it belongs*, as well as the *input* passed to the action. There's also information about the original "user activity" that ultimately resulted in the log entry, essentially providing a form of conversational threading to logging.

Flint's logging APIs then allow your logging subsystem to include this information so you can get far more contextual detail about e.g. why a network request is happening. It is also built to allow you to change logging levels at runtime — typically while you are debugging — and to filter logging to include only information from specific features. You can even set log levels per feature.

## What are topic paths?

The logging subsystem is largely decoupled from the rest of Flint, and it is designed to support non-Flint contextual logging, so there is a concept of a `TopicPath` which identifies a hierarchical taxonomy for log entries.

Essentially it is an array of strings, which can then later be used to filter log entries to only certain portions of this hierarchy. e.g. logging to a topic path `["Networking", "Feeds", "Parsing"]` would mean you could set the log level for `["Networking"]` to `none` but set the parsing logging at path `["Networking", "Feeds", "Parsing"]` to `debug` level.

Flint maps `FeaturePath` to `TopicPath` under the hood, which is how we get log filtering by feature.

## Changing log levels at runtime

Documentation TBD. See [`Logging`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Logging/Logging.swift).

## Using the built in loggers

Flint ships with a simple `print` logger and an `OSLog` logging output implementation. These are configured out of the box when you run `Flint.quickSetup` but you can customize their behaviours.

For now, [see the source for details](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Core/Flint.swift#L120).

## Wiring up your own logging output

For now, please see how the [default logging is configured here](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Logging/DefaultLoggerFactory.swift#L68).

## Next steps

* See the [Focus](focus) feature of Flint to screen out everything from logs except the stuff that matters to you


