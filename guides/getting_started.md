---
title: Getting Started
subtitle: Learn how to integrate Flint into your project
tags: guide featured
---

In order to use Flint you need to build the framework and add it and its dependency to your project.

## Building Flint

The easiest way to use Flint in your own project is via [Carthage](https://github.com/Carthage/Carthage). You add the dependency to your `Cartfile` like this:

```
github "MontanaFlossCo/Flint" "master"
```

**Note**: We are specifying "master" as the version to use for now, as currently there are no tagged releases on Flint at the moment. Tags are coming soon.

Then run `carthage bootstrap` to build it for all platforms. For faster builds you can limit to one platform and use caching to avoid rebuilding if there are no changes to the version you already have, e.g.:

```
carthage bootstrap --platform iOS --cache-builds
```

## Adding the frameworks to your project

Once you have built the frameworks, you'll need to add them to your project.

1. Drag the FlintCore.framework from `./Carthage/Build/iOS` (or whatever platform folder your project requires) onto your "Linked Frameworks & Libraries" section of your app target's General tab in Xcode
2. If you need `FlintUI` for iOS then also do the same for that framework
3. Do the same for `ZIPFoundation.framework`, a dependency of Flint
4. Verify on your Build Phases tab that those frameworks are listed in the "Link Binary With Libraries" section
5. Ensure that your project is set up properly for Carthage's `copy-frameworks` build phase. See the [Carthage Quick Setup](https://github.com/Carthage/Carthage#quick-start)

## Next steps

* You need to [bootstrap Flint in your code and create some features](features_and_actions).
