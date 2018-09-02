---
title: Action Stacks
subtitle: See what the user is doing right now
tags:
    - debug
    - logging
---

#### In this article
{:.no_toc}
* TOC
{:toc}

## Overview

The Action dispatch mechanism of Flint maintains a "stack" of actions performed within a given feature. 

Each time the user performs an Action from a Feature that is not currently active in the current set of Action Stacks, a new stack will be created for that Session and Feature. These stacks are scoped per Action Session and Feature.

What this means is that at any point in time you can browse the current active stacks, or save these out to a file if you experience a crash or the user makes a support request in your app.

You will get information about every action they performed in time order per feature, across all the sessions so you can see possible interactions between background and foreground tasks — for the stacks that have not been "closed"

## How does this work?

When you `perform` an action it takes place within an `ActionSession` behind the scenes.

Flint will see if the feature the action belongs to is already part of an active action stack in the current session, and if not creates a new root-level stack automatically. Otherwise, it appends the new action to the existing stack for that feature.

A critical part of this mechanism is that your actions *must always* call the `completion` handler passed to them, with a result that correctly identifies whether or not the current action stack for the feature can be closed.

## What does closing an action stack mean?

The idea around an action stack is that it encapsulates a set of interactions the user has with a single feature. Some features do not have a "start" or an "end", but most typically do. We need to know when a feature is "no longer in use" to track what users are currently doing vs. tracking everything they have done over a time period (this is something else, the [Timeline](timeline.md)). 

The user will usually perform one or more actions of a feature, and some of these actions are a natural "end" to the use of that feature. For example a document editing feature may have many actions including "insert photo", and this latter action would not close the action stack as the user has not finished editing.

However a "close document" action on that feature would naturally delineate use of the feature, and that is where your action should pass `true` for `closeActionStack`:

```swift
final class CancelPhotoSelectionAction: Action {
    typealias InputType = NoInput
    
    typealias PresenterType = PhotoSelectionPresenter
    
    public static func perform(with context: ActionContext<NoInput>, using presenter: PresenterType, completion: Completion) -> Completion.Status {
        presenter.dismissPhotoSelection()
        return completion.completedSync(.success(closeActionStack: true))
    }
}
```

## Browsing active stacks with Flint UI

If you are running on iOS you can use the `FlintUI` framework to show a basic UI to browse the currently active action stacks in the app.

To do this, you need to import the framework and use the supplied feature to display it:

```swift
import UIKit
import FlintUI

// ... in some action or gesture handler of a UIViewController ...

ActionStackBrowserFeature.show.perform(using: self)
```

## Generating a debug report containing the stacks

At the time of writing persistence of action stacks is not implemented.

You can however access the active stacks yourself at runtime for now, using [`ActionStackTracker`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Core/ActionStackTracker.swift)
