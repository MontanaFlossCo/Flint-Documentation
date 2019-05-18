---
title: Threading
subtitle: How actions are performed in a concurrent environment
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

It is not unusual to see completion handler code in non-Flint apps that simply "trampoline" their code onto `DispatchQueue.main` asynchronously, even though the caller may already have done this. Also you find call sites making decisions about whether they should perform something synchronously or not, when in most cases the action being called may complete synchronously *or* asychronously depending on other factors (e.g. cache fast paths where data is immediately available).

The Action dispatch mechanism in Flint makes these guarantees:

1. You can perform any Action from any queue/thread.
2. You can specify which queue your Completion handler will be called on. By default this is the queue the Action requires (to minimise thread hops/queue usage when everything is on the main queue)
3. The Action decides whether it will completed asynchronously or not. Callers do not know the difference unless they need to

The "magic" under the hood is that Flint knows what queue it is currently called on. There is just one caveat: you cannot use it with queues that have GCD target queues and expect this behaviour to work unless you're on exactly the correct queue i.e. a new queue with `main` as target queue will not be detected as being the `main` queue currently.

## Defining the queue on which an Action will be called

## Defining the queue on which Action completion will be called

## Actions that complete asynchronously

