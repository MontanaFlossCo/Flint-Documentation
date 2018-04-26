# Routes

Most apps need to handle some URLs, whether for e-mail sign-in confirmations, deep linking or custom workflow URL schemes.

Flint’s Routes feature makes it easy to implement these. You have only two steps to carry out in code.

Routes can work automatically behind the scenes with the [Activities](activities.md) feature.

## Declaring the URL routes for your Feature’s actions

Using Flint’s [Routes](guides/routes.md) feature we can add URL routes for any actions. This gives us custom app URL scheme and universal link support with minimal effort. 

URL mappings are declared on the Feature, so that actions can be reused across features without fixing the mappings in the action. Editing the previous feature declaration, we can just add conformance to `URLMapped` and add an implementation of the `urlMappings` function to the existing code:

```swift
/// Add the `URLMapped` conformance to get support for Routes
class DocumentManagementFeature: Feature, URLMapped {

    /// Add the URL mappings for the actions.
    static func urlMappings(routes: URLMappingsBuilder) {
        // Note that there is full support for mapping to specific app URL schemes and specific universal link
        // domains. By default it maps to the primary app scheme and primary associated domain.
        // Primary means first in the list — See `FlintAppInfo`
        routes.send("create", to: createNew)
        routes.send("open", to: openDocument)
    }
}
```

That's all you need to do to define some URLs that will invoke actions.

## Adding the code your app needs to handle URLs and present the UI

To actually make this work, there are a few more one-off things to do:
1. If you haven't done so already you have to declare your App's custom URL schemes in your `Info.plist`
2. For universal link / associated domains you need to set up the entitlements and a file on your server. See the Apple docs for this.
3. You need your application delegate to handle requests to open URLs and pass them to Flint.
4. Implement an object to get your UI ready and return a presenter. See the docs for [Routes](guides/routes.md))*

This application delegate part is very simple – add this to your `UIApplicationDelegate`, for example:

```swift
func application(_ app: UIApplication, open url: URL, options: [UIApplicationOpenURLOptionsKey : Any] = [:]) -> Bool {
     let result: URLRoutingResult = Flint.open(url: url, with: presentationRouter)
     return result == .success
}
```

The presentation router part needs to look at your current UI's state and do any work required to shuffle around view controllers to achieve the behaviour you want when your app receives a request for an action when it is already in a different UI state. This can be tricky, but it's the nature of the beast. It can often be quite simple – the key is to make clear decisions about how you want the app to behave for each kind of action that it can receive from an external stimulus like this. 

Here's the example from the [FlintDemo-iOS](https://github.com/MontanaFlossCo/FlintDemo-iOS) sample project:

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

Now the app will respond to and be able to generate URLs referring to those actions, including arguments. So for example the [Flint Demo](https://github.com/MontanaFlossCo/FlintDemo-iOS) app has this code and could respond to `flint-demo://open?name=hello.md` as well as `https://demo-app.flint.tools/open?name=hello.md`. Pretty neat, isn’t it?

## Deep linking using web URLs with Associated Domains

Docs coming soon

## Customising the URL schemes and Associated Domains per-Action Route

Docs coming soon

## Updating your `Info.plist` for URL schemes and Associated Domains

Docs coming soon
