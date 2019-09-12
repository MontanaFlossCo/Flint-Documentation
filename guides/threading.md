---
title: Threading
subtitle: How actions are performed in a concurrent environment.
tags:
    - help
    - actions
---

#### In this article:
{:.no_toc}
* TOC
{:toc}

## Overview

Flint is designed to remove many of the threading concerns when performing actions.

A common source of threading bugs, performance problems or delayed UI updates can be found in uncertainty over the thread that your code is being called on. This applies to code that implements "actions" the user can perform or trigger as well as completion handlers supplied by the caller. 

It is not unusual to see completion handler code in apps that simply "trampoline" their code onto `DispatchQueue.main` asynchronously, even though the caller may already have done this. You also find call sites making decisions about whether they should perform something synchronously or not, when in many cases the action being called may complete synchronously *or* asychronously depending on other factors (e.g. cache fast paths where data is immediately available).

The Action dispatch mechanism in Flint makes these guarantees:

1. You can perform any Action from any queue/thread.
2. You can specify which queue your Completion handler will be called on. By default this is the queue the Action requires (to minimise thread hops/queue usage when everything is on the main queue)
3. The Action decides whether it will completed asynchronously or not. Callers do not know the difference unless they need to for some reason

Flint **does not** make any guarantees about your action's interaction with its presenter. You are responsible for defining the threading contract there. In the vast majority of cases your presenter is intended to be called on the same queue as your action (e.g. the main thread).

Remember that the Xcode Thread Sanitizer is your friend. Turn it on and build more robust apps.

_(Note: the "magic" under the hood is that Flint knows what queue it is currently called on. There is just one caveat of interest to advanced users: you cannot use this with queues that have GCD target queues and expect this behaviour to work unless you're on exactly the correct queue i.e. a new queue with `main` as target queue will not be detected as being the `main` queue currently.)_

## Defining the queue on which an Action will be called

There is a static property convention on `Action` called `queue` that your Action can define to set its queue:

```swift
final class MyBackgroundAction: Action {
    static let queue: DispatchQueue = DispatchQueue.global()

    ...
}
```

You simply have it return the queue you want the `Action` to be called on whenever it is called.

By default Flint provides two special `Action` protocols that your actions can conform to, which have default queues (and ActionSession) already set; `UIAction` for main-thread UI actions, and `IntentAction` for performing intents in Siri Intent extensions.

You make your actions conform to these protocols instead of `Action` to get these defaults for free:

```swift
final class DocumentOpenAction: UIAction {
    ...
}

final class GetNoteAction: IntentAction {
    ...
}
```

## Defining the queue on which Action completion will be called

By default in Flint 1.0 all the action completion handlers assume the simple case where completion will be called on the same queue on which the Action is performed (the one it specifies in its `queue` convention property). Most actions are UI actions (on the main thread) and hence calling completion on the main thread is usually what you want.

If you need to alter this behaviour, you must call one of the `ActionSession.perform(...)` functions and pass in a `completionQueue` argument or an entire `Action.Completion` (a [`CompletionRequirement`]()) that you have created. You can do this via your `ActionSession` or via your Action's `defaultSession` convention:

```swift
ActionSession.main.perform(MyFeature.myAction, completion: { outcome in  }, 
                           completionQueue: anotherQueue)

// - or -

MyFeature.myAction.defaultSession.perform(MyFeature.myAction, completion: { outcome in  }, 
                           completionQueue: anotherQueue)
```

## Actions that complete asynchronously

Actions generally complete synchronously. This is indicated by returning `completion.completedSync(...)` with an outcome value. If however the `Action` implementation cannot or should not complete synchronously, it needs to tell Flint that this is the case, for housekeeping and to avoid common problems where completion handlers are not called. i.e. having to tell Flint that your action is completing asynchronously and not calling completion directly in your implementation makes it clear in your code that this is happening, and you cannot forget to call completion.

To indicate that your action will complete asynchronously, instead of returning the result of `completion.completedSync(...)` you must return the result of `willCompleteAsync()`, and keep hold of the return value to indicate completion later:

```swift
final class AsyncCompletingAction: UIAction {
    typealias PresenterType = MyPresenter

    static func perform(context: ActionContext<NoInput>, 
                        presenter: MyPresenter, 
                        completion: Completion) -> Completion.Status {
        let asyncResult = completion.willCompleteAsync()

        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            presenter.doSomething()
            asyncResult.completed(.success)
        }
        
        return asyncResult
    }
}
```


