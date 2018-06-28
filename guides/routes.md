---
title: URLs Routes
subtitle: Learn to use Flint's powerful routes to direct incoming URLs to your actions with minimal code
tags: guide featured
---

Most apps need to handle some URLs, whether for e-mail sign-in confirmations, deep linking or custom workflow URL schemes. 

Flint’s Routes feature makes it easy to implement these. You have only three steps to carry out in code. What this will do is take a URL like:

`my-app://send?tweet=Hello%20World`

…and route that to the appropriate Action in your app, pull out the information from the URL to create an instance of the `InputType` required for the Action, and then perform the action for you. The same Action that you use to perform the code internally in your app, that has its availability controlled by the Feature it is bound to.

So for example if you have an in-app purchase to unlock advanced workflow features, there's nothing more to do other than make sure the workflow URLs are defined on the conditional feature. The URLs will not work unless they have paid for the feature.

Flint's Routes support:

* Multiple custom app URL schemes
* Multiple associated domains
* Multiple URL paths to the same action
* Multiple schemes and associated domains for the same URL path
* Custom marshalling of URL arguments
* Reuse of existing app Actions — URLs are just another way to perform them

Routes can also work automatically behind the scenes with the [Activities](activities.md) feature so that you get `NSUserActivity` support for free.

## Declaring the URL routes for your Feature’s actions

URL routes for actions are declared on the Feature, so that actions can be reused across features without fixing the mappings in the action. This also means the availability of the URLs is controlled by the Feature's availability.

All you need to do is add conformance to the `URLMapped` protocol to your `Feature` and add an implementation of the `urlMappings` function:

```swift
/// Add the `URLMapped` conformance to get support for Routes
class DocumentManagementFeature: Feature, URLMapped {

    ... 
    static let createNew = action(DocumentCreateAction.self)
    static let openDocument = action(DocumentOpenAction.self)

    /// Add the URL mappings for the actions.
    static func urlMappings(routes: URLMappingsBuilder) {
        routes.send("create", to: createNew)
        routes.send("open", to: openDocument)
    }
}
```

That's all — you have now defined some URLs that will invoke actions. If your app has configured a custom URL scheme `x-your-app` and an associated domain `your-app-domain.com`, your URLs might look like this:

* `x-your-app://create?name=MyFile1`
* `https://your-app-domain.com/open?docRef=456643564563634643643`

## Adding the code your app needs to handle URLs and present the UI

To actually make this work, there are a few more one-off things to do:

1. If you haven't done so already you have to declare your App's custom URL schemes in your `Info.plist`
2. For universal link / associated domains you need to set up the entitlements and a file on your server. [See the Apple docs for this](https://developer.apple.com/library/content/documentation/General/Conceptual/AppSearch/UniversalLinks.html#//apple_ref/doc/uid/TP40016308-CH12-SW1).
3. You need your application delegate to handle requests to open URLs and pass them to Flint.
4. Implement an object to get your UI ready and return a presenter.

This application delegate part is very simple – add this to your `UIApplicationDelegate`, for example:

```swift
func application(_ app: UIApplication, open url: URL, options: [UIApplicationOpenURLOptionsKey : Any] = [:]) -> Bool {
     let result: URLRoutingResult = Flint.open(url: url, with: presentationRouter)
     return result == .success
}
```

The [`PresentationRouter`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Routes/PresentationRouter.swift) component needs to look at your current UI's state and do any work required to shuffle around view controllers to achieve the behaviour you want when your app receives a request for an action when it is already in a different UI state. This can be tricky, but it's the nature of the beast. You can make it simple in many cases – the key is to make clear decisions about how you want the app to behave for each kind of action that it can receive from an external stimulus like this. 

An example strategy is that maybe for all actions you want to present the view controller modally, unless there is already a modally presented view controller in which case you would fail the routing request rather than interrupt the user again.

Here's a simple example [`PresentationRouter`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Routes/PresentationRouter.swift) implementation from the [FlintDemo-iOS](https://github.com/MontanaFlossCo/FlintDemo-iOS) sample project, which only supports presenting actions if the app's main `UINavigationController` has a specific `MasterViewController` at the top of its view controller list:

```swift
/// A presentation router for setting up the UI and returning the appropriate presenter instance for
/// an action request that has come from outside the app, e.g. an `openURL` request.
class SimplePresentationRouter: PresentationRouter {
    let mainNavigationController: UINavigationController
    
    /// We instantiate this in our `AppDelegate` and pass it our primary navigation controller
    init(navigationController: UINavigationController) {
        mainNavigationController = navigationController
    }
    
    /// Set up the UI for unconditional feature actions
    func presentation<FeatureType, ActionType>(for actionBinding: StaticActionBinding<FeatureType, ActionType>, with state: ActionType.InputType) -> PresentationResult<A.PresenterType> {
        // Switch (not literally, we can't do that sadly), on the expected presenter type
        // and return `.appReady` with the main navigation controller as presenter if
        // it is one of the supported types, but only if the user is currently on the master view
        // This will silently ignore requests for actions that come in if the user is on the detail view controller
        if ActionType.PresenterType.self == DocumentCreatePresenter.self ||
                A.PresenterType.self == DocumentPresenter.self {
            if let masterVC = mainNavigationController.topViewController as? MasterViewController {
                return .appReady(presenter: masterVC as! ActionType.PresenterType)
            } else {
                return .appCancelled
            }
        } else {
            // The presentation router doesn't know how to set up the UI for this action.
            // This is probably a programmer error.
            return .unsupported
        }
    }
 
    /// Set up the UI for conditional feature actions. We don't support any of these in the demo app.
    func presentation<FeatureType, ActionType>(for conditionalActionBinding: ConditionalActionBinding<FeatureType, ActionType>, with state: A.InputType) -> PresentationResult<A.PresenterType> {
        return .unsupported
    }
}
```

Now the app will respond to and be able to generate URLs referring to those actions, including arguments. So for example the [Flint Demo](https://github.com/MontanaFlossCo/FlintDemo-iOS) app has this code and could respond to `flint-demo://open?name=hello.md` as well as `https://demo-app.flint.tools/open?name=hello.md`.

## Creating the Input for your Action from the URLs

In order to respond to URLs and perform the Action defined by your routes, Flint needs to be able to create an instance of your `InputType` type from the URL.

Flint provides three protocols to support this:

* [`QueryParametersDecodable`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Routes/QueryParametersCodable.swift)
* [`QueryParametersEncodable`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Routes/QueryParametersCodable.swift)
* [`QueryParametersCodable`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Routes/QueryParametersCodable.swift) (combines both of the above)
 
You need to make your action's `InputType` conform to one or two of these protocols as appropriate for your needs. If you never create URL links in your app but only have to handle them, you can conform to just `QueryParametersDecodable`. If you need to create links you can conform to `QueryParametersDecodable` or `QueryParametersCodable`. 

It is nice to do this in an extension separate from your `InputType` definition. So for an `InputType` of `DocumentRef` we might have something like this:

```swift
extension DocumentRef: QueryParametersCodable {
    init?(from queryParameters: QueryParameters?) {
        guard let name = queryParameters?["name"] else {
            return nil
        }
        self.name = name
    }
    
    func encodeAsQueryParameters() -> QueryParameters? {
        return ["name": name]
    }
}
```

Note that the `QueryParameters` give you convenient access to the query parameters directly.

This will allow the `DocumentRef` to be used to create links and to parse them to create an instance of `DocumentRef` to pass to the `Action` (that is the part that the `RoutesFeature` of Flint does for you).

## Customising the URL schemes and Associated Domains per-Action

The `urlMappings(routes:)` function is passed a builder that you use to set up the routes. The build supports a single `send` function that takes a URL path and a `to:` argument of the action binding to use. It has an optional `in:` argument for the scopes:

```swift
routes.send(_ path: String,
            to actionBinding: …binding type… ,
            in scopes: Set<RouteScope> = [.appAny, .universalAny])
```

There is full support for mapping to specific app URL schemes and specific universal link domains using the `scopes`. By default it maps to the all app schemes and associated domains handled by your app. To route differently, say to some legacy URLs you would write something like this:

```swift
class DocumentManagementFeature: Feature, URLMapped {

	... 

    /// Add the URL mappings for the actions.
    static func urlMappings(routes: URLMappingsBuilder) {
        // Legacy URL action
        routes.send("create", to: createNew, in: [
	        .app(scheme: "x-test"),
	        .app(scheme:"internal-test"),
	        .universal(domain: "yourdomain.com")])
	
        // Current URL action
        routes.send("create-document", to: createNew)

        // Current URL action, with just legacy domain
        routes.send("open", to: openDocument, in: [
	        .appAny,
	        .universal(domain: "legacydomain.com"),
	        .universal(domain: FlintAppInfo.urlSchemes.first)])
    }
}
```

See [`FlintAppInfo`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Core/FlintAppInfo) for access to the known domains and schemes for the app. 

As you can see there is support for declaring multiple scopes, and you can declare the same route multiple times to different destinations if need be, so long as the scopes differ. The URL mapping will choose the most specific match, so an explicit scheme or domain match wins even if you have `.appAny` or `.universalAny` defined.

Your app won't invoke URLs you have defined yet, as you need to tell Flint when URLs are received and allow it to perform the action for you, and you need to make sure your app is configured to receive the custom URL scheme and associated domains.

## How to test URL routes

The easiest way to test URL routes manually is to create a suitable URL and paste it into Safari on the device where the app is installed. You should be prompted to open your app. If you press OK it should then open the app and invoke your action. If you did all the `PresentationRouter` stuff right you should see the correct UI!

Typing URLs is particularly painful on iOS devices, so other alternatives include:

* sending yourself the URL via iMessage or by Email
* storing the URLs you want to test in a note in the Notes app, using iCloud sync, and just tap them to test them again
* build an internal HTML page somewhere describing all the URLs your app should support, so they can easily be tested by tapping one after another

Testing associated domains is harder. You must have the server set up correctly for the associated domain and be running the app on a real device. You will need to take care to avoid clashing with production releases of your app, so you must either remove the production apps or use subdomains for each flavour of your app build (e.g. development vs. beta vs. production).

**💡 TIP**: You should test the case where your app is not already running on the device. In Xcode you can do this by terminating the app first, and then in the "Run" options for your app's scheme, set the "Launch" option to "Wait for executable to be launcher". Then when you open a URL from Safari Xcode will be able to stop on breakpoints you set in your `PresentationRouter` so you can debug any issues there.

## Deep linking using Universal Link URLs with Associated Domains

Depending on where the user interacts with an associated website URL, you may receive a call to `application:openURL:…` or `application:continueActivity:…`.

For deep linking that works with the `continueActivity` variant you will need to also enable the [Activities](activities.md) feature of Flint and add the required call to your app delegate.

## Updating your `Info.plist` and Entitlements for URL schemes and Associated Domains

For custom URL schemes to work, you must list each scheme in the "URL Types" section of your `Info.plist` (or the "Info" tab of your app target in Xcode). The schemes must go in the "Schemes" box.

Associated domains on the other hand require entitlements. You must go to the "Capabilities" tab and open the "Associated Domains" section. There you can add the domains you require, with the `applinks:` prefix e.g. `applinks:yourdomain.com`. You must also create and upload the `apple-site-association` file to your server on that domain.

**💡 TIP**: Do your users a favour while you are here — if you have website logins that the users require in the native app, add `webcredentials:` associations at the same time as this!

For more details see [Supporting Universal Links](https://developer.apple.com/library/content/documentation/General/Conceptual/AppSearch/UniversalLinks.html#//apple_ref/doc/uid/TP40016308-CH12-SW1) on the Apple documentation site.

## Creating links to your Actions

Sometimes you want to create a URL that will invoke your app and perform an action, or generate the equivalent Universal Link so that you can share content with other users from the app.

To do this you use the [`LinkCreator`](https://github.com/MontanaFlossCo/Flint/blob/master/FlintCore/Routes/LinkCreator.swift) protocol. An object that implements this is provided in `Flint.linkCreator` if you called `Flint.quickSetup` at startup. `Flint.quickSetup` chooses the first custom URL scheme and the first associated domain as the scopes to use when creating links. You can create your own `LinkCreator` instances using other schemes and domains if you require it.

The `LinkCreator` creates `app` or `universal` links like so:

```swift
let appUrl = Flint.linkCreator.appLink(to: MyFeature.someAction, with: someInput)
let webUrl = Flint.linkCreator.universalLink(to: MyFeature.someAction, with: someInput)
```

**💡 TIP**: You can use links for other things like Home Screen Quick Actions. Anywhere you might need to open your app in a specific state, creating links is a great solution. This what the [Activities](activities.md) feature takes advantage of.

## Next

* Set up [Activities](activities) handling for Handoff, deep linking and more
* Add [Analytics](analytics) tracking
* Use the [Timeline](timeline) to see what is going on in your app when things go wrong
* Start using [Focus](focus) to pare down your logging
