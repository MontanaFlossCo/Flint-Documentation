---
title: Timeline
subtitle: Find out what your user was doing in the run up to a problem
tags:
    - coreconcepts
    - logging
    - debug
    - featured
---

#### In this article
{:.no_toc}
* TOC
{:toc}

## Overview

The Timeline feature of Flint is a rolling limited-length history of all actions performed in the app across all sessions and features. This is a sequential way to see exactly what has happened in the app at a high level.

It is incredibly valuable if captured and included in logs or support requests.

## Using the Timeline Browser from FlintUI on iOS

```swift
import UIKit
import FlintUI

... in some action or gesture handler of a UIViewController ...

guard let request = TimelineBrowserFeature.request(TimelineBrowserFeature.show) else {
    preconditionFailure("Timeline is not enabled")
}
request.perform(using: self)
```

This will show the modal UI listing a live-updating timeline of actions performed. You can tap items to get more details.

## Disabling Timeline

The Timeline feature is enabled by default. If you do not use it or would rather not have the overhead in certain builds, you can easily disable it:

```swift
TimelineFeature.isEnabled = false
```

## Using Timeline in the debugger

TBD

## Generating Timeline reports

TBD — timelines are not currently persisted but this is coming in 1.0

## Changing the maximum number of entries

TBD

## Next steps

* Start using [Focus](focus.md) to pare down your logging
