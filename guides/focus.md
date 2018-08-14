---
title: Focus
subtitle: Debug specific features with Focus
tags:
    - debug
    - logging
    - featured
---

#### In this article
{:.no_toc}
* TOC
{:toc}

## Overview

The Focus feature of Flint allows you to debug more effectively by stripping away all the noise from your logs and the [Timeline](guides/timeline.md).

When you tell Flint to focus on one or more specific features, all timeline and logging output will be dynamically filtered to include only entries derived from actions performed on those features. It's like drinking from a bottle instead of a firehose.

## How to change focus

Focus is implemented as a feature inside of flint. 

To change focus you perform the `focus` or `defocus` actions on `FocusFeature`, passing in a `FocusArea` instance that describes you area of interest:

```swift
if let request = FocusFeature.focus.request() {
    request.perform(with: FocusArea(feature: MyBuggyFeature.self)))
}

...


// Revert back to previous focus - probably the firehose
if let request = FocusFeature.defocus.request() {
    request.perform(with: FocusArea(feature: MyBuggyFeature.self)))
}
```

## Disabling the Focus Feature

You can leave focus-changing code like the above in your app, perhaps tailored for different build flavours during development to help reduce noise during a sprint.

Because `FocusFeature` is a conditional feature the uses the `runtimeEnabled` constraint, you can entirely disable it at runtime with one line of code:

```swift
FocusFeature.isEnabled = false
```