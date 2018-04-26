# Flint 💥

Flint is a pure Swift framework for building iOS, tvOS, watchOS or macOS apps that utilise
[Feature Driven Development](https://www.montanafloss.co/blog/feature-driven-development), using the power of Features,
Actions and conventions to make app development and debugging more productive. 

If you don't know what Feature Driven Development can give you, please [read this linked blog post](https://www.montanafloss.co/blog/feature-driven-development) for a detailed explanation. The TL;DR description of FDD is:

> Expressing information about the Features and Actions of your application in the code itself, and using this information to make your apps better   

Have you ever tried to implement deep-linking URLs? Have you had to sprinkle analytics reporting code here and there throughout your app? Dealt with the request to feature-flag or A/B test your new features? Implemented Handoff or Siri Suggestion integration?

The answer may be “yes’ to all of these and you’re somewhat exhausted by all the little details. On the other hand the answer may be “no” to several, in which case your users or your business didn’t get the experience they deserve, because these things are either fiddly, unattractive to implement, or both.

By defining the Features and Actions of your app using Flint, you get a bunch of functionality for free on Apple platforms. 

* **Timeline** — an automatic history of the actions your users have performed  
* **Routes** — App URL schemes and Universal links that invoke those actions for deep linking and link generation
* **Activities** — Automatic registration of NSUserActivity for actions performed, for Handoff, Siri suggestions, Spotlight search
* **Focus** — Runtime control of what is logged in your app based on your app Features. Find problems quickly without the noise of all your subsystems' logging
* **Analytics** — Automatic recording of app analytics when actions are performed, using any Analytics service you use
* **Action Stacks** — Know what your users were doing in your app for debugging and crash reporting
* **Toggling** — manual feature toggling, A/B testing, IAP or subscription based toggling of features is made easy and typesafe
* Debug UIs — Flint also provides a `FlintUI` framework (iOS-only right now) with several useful debug UIs for browsing the Features and Actions declared in your application, viewing the user Timeline,  viewing Focus logs in realtime and browsing the current Action Stack.  

Much of this functionality is implemented within Flint as Flint’s own Features and Actions — it’s features all the way down.

Flint is in an “early access” phase at the moment to shake down the public API and concepts. Expect some API changes before the full 1.0 release. *Please try it and tell us what you think*.

**Note that Flint is deliberately Swift-only**, using capabilities of the language to provide a type-safe implementation of FDD. It will not work with Objective-C apps unless you create stub functions that call the Swift code to perform each action. It is probably not worth your time. Of course your Swift Flint `Action` code can call into Objective-C as normal.

## Documentation and sample code

If you want to see a sample project that uses Flint, there is the  [FlintDemo-iOS][] project here on Github. You can browse that to get an
idea of how a real app might use Flint.

We're currently writing guides to the various aspects of Flint:

* [Features and Actions](guides/features_and_actions.md)
* [Activities](guides/activities.md)
* [Routes](guides/routes.md)
* [Analytics](guides/analytics.md)
* [FlintUI](guides/flint_ui.md)
* [Focus](guides/focus.md)
* [Timeline](guides/timeline.md)
* [Action Stacks](guides/action_stacks.md)



## Unit Testing

Yes, we will have lots of unit tests on Flint. Not just yet though as we don't want to waste time when the API may still fluctuate significantly. Once there has been time for people to try out and comment on the APIs, things will settle down and the tests will be put together over the coming weeks.

Testing of your Flint-based code should be simple enough - but we likely have to build a few mock classes for you. Work will be ongoing here. 

Actions should be simple enough to unit test. You create an `[ActionContext](blob/master/FlintCore/Actions/ActionContext.swift)` and pass it to the action's `perform` function, along with your test input and presenter.

Examples will follow soon.

## The roadmap to 1.0 final release

There is of course much left to do! Here is a high level roadmap  of planned work prior to the full 1.0 release.

* ✅ Feature and Action declaration, Action dispatch
* ✅ Timeline feature
* ✅ Deep Linking feature
* ✅ Activities feature
* ✅ Focus feature
* ✅ Action Stacks feature
* ✅ Exportable debug reports
* 👨‍💻 Early-access public API review 
* 👨‍💻 Implement IAP / Subscription validation
* 👨‍💻 Implement core unit tests, set up CI
* 👨‍💻 Implement Built-in persistent file logger
* 👨‍💻 Implement Persistence of Action Stacks, Focus Logs and Timeline at runtime
* 👨‍💻 Examples of Mixpanel, Hockey and Fabric integrations
* 👨‍💻 1.0 Release

## Philosophy

We are all-in on Swift but we don’t want to be smartypants who can’t read our own code weeks later. We take a few advanced Swift features that make great things possible: Protocol Oriented Programming, some generics and a very small amount of associated types.

We deliberately avoid the more oblique patterns because we want this framework to be very accessible and easy for everybody to reason about, irrespective of the paradigm they have chosen for their codebase.

## Community and Contributing

We have a community Slack you can join to get help and discuss ideas. Join at [flintcore.slack.com](https://join.slack.com/t/flintcore/shared_invite/enQtMzUwOTU4NTU0OTYwLWMxYTNiOTNjNmVkOTM3ZDgwNzZiNzJiNmE2NWUyMzUzMjg3ZTg4YjNmMjdhYmZkYTlmYmI2ZDQ5NjU0ZmQ3ZjU).

We would love your contributions. Please raise Issues here in Github and discuss your problems and suggestions. We look forward to your ideas and pull requests.

Flint is copyright Montana Floss Co. with an [MIT open source licence](LICENSE).

[FlintDemo-iOS]: https://github.com/MontanaFlossCo/FlintDemo-iOS
