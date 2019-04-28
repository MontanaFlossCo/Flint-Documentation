---
title: Getting Started
subtitle: Learn how to integrate Flint into your project
tags:
    - coreconcepts
    - help
    - featured
---

#### In this article:
{:.no_toc}
* TOC
{:toc}

## Overview

In order to use Flint you need to build the framework and add it and its dependency to your project.

## Carthage — the recommended way

The easiest way to use Flint in your own project is via [Carthage](https://github.com/Carthage/Carthage). You add the dependency to your `Cartfile` like this:

```
github "MontanaFlossCo/Flint" "{{site.flint.release_tag}}"
```

**Note**: We are specifying tags rather than semantic versions to use for now, as currently we are not at the point where we can guarantee no API breakage between these early access 1.0.x releases. Once we hit the final 1.0 API we'll be semver friendly!

Then run `carthage bootstrap` to build it for all platforms. For faster builds you can limit to one platform and use caching to avoid rebuilding if there are no changes to the version you already have, e.g.:

```sh
carthage bootstrap --platform iOS --cache-builds
```
### Adding the frameworks to your project

Once you have built the frameworks, you'll need to add them to your project.

1. Drag the FlintCore.framework from `./Carthage/Build/iOS` (or whatever platform folder your project requires) onto your "Linked Frameworks & Libraries" section of your app target's General tab in Xcode
2. If you need `FlintUI` for iOS then also do the same for that framework
3. Verify on your Build Phases tab that those frameworks are listed in the "Link Binary With Libraries" section
4. Ensure that your main app target is set up properly for Carthage's `copy-frameworks` build phase, so that it correctly copies the frameworks `FlintCore`, and if you're using it, `FlintUI`. Add a `Run Script` build phase that runs the command `carthage copy-frameworks`, and add each Framework as an input file e.g. `$(SRCROOT)/Carthage/Build/iOS/FlintCore.framework` and as an output file such as `$(SRCROOT)/Carthage/Build/iOS/FlintCore.framework` so it only processes modified frameworks for quicker build times.

## Cocoapods — the not so recommended way

You can also use Cocoapods if your project already requires this. Add the dependency to your `Podfile`:

```ruby
pod 'FlintCore', '~> {{site.flint.release_tag}}'
```

Then run `pod install` to install the dependency into your project.

## Next steps

* You need to [bootstrap Flint in your code and create some features](features_and_actions.md).
