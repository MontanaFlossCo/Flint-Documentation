## Getting started

[Back to the Documentation index](../index.md)

To use Flint in your own project, use [Carthage](https://github.com/Carthage/Carthage) to add the dependency to your `Cartfile`:

```
github "MontanaFlossCo/Flint" "master"
```

Then run `carthage bootstrap`. For faster builds you can limit to one platform and use caching, e.g.:

```
carthage bootstrap --platform iOS --cache-builds
```

Next, you need to [bootstrap Flint in your code and create some features](features_and_actions.md).
