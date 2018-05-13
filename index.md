# Flint Documentation

Building great apps for Apple platforms involves a lot of work; **custom URL schemes**, in-app **purchases**, authorising **system permissions**, universal **links**, **Handoff** and **Siri** support, tracking **analytics** events, **feature flagging** and more. These things can be fiddly and time consuming, but you shouldn't be hand-cranking all that!

Flint is a framework that helps you deal with all this easily, leaving you and your team to focus on what makes your product special. Using an approach called [feature driven development](https://www.montanafloss.co/blog/feature-driven-development) you split your code into actions that make up the Features of your app and Flint takes care of the rest. These high level interactions with your UI are simple to test and decouple your UI. The icing on the cake is that because Flint knows what your users are actually doing in your app, you also get revolutionary debug capabilities for free! 🎂🎉 

### Version 1.0 — "early access" 

[Guides and How-to's](1.0/index.md)

API Reference (coming soon)

## Other resources

Sample application built with Flint: [FlintDemo-iOS][]

## Philosophy

We are all-in on Swift but we don’t want to be smartypants who can’t read our own code weeks later. We take a few advanced Swift features that make great things possible: Protocol Oriented Programming, some generics and a very small amount of associated types.

We deliberately avoid the more oblique patterns because we want this framework to be very accessible and easy for everybody to reason about, irrespective of the paradigm they have chosen for their codebase.

## Community and Contributing

We have a community Slack you can join to get help and discuss ideas. Join at [flintcore.slack.com](https://join.slack.com/t/flintcore/shared_invite/enQtMzUwOTU4NTU0OTYwLWMxYTNiOTNjNmVkOTM3ZDgwNzZiNzJiNmE2NWUyMzUzMjg3ZTg4YjNmMjdhYmZkYTlmYmI2ZDQ5NjU0ZmQ3ZjU).

We would love your contributions. Please raise Issues here in Github and discuss your problems and suggestions. We look forward to your ideas and pull requests.

Flint is copyright Montana Floss Co. with an [MIT open source licence](LICENSE).

[FlintDemo-iOS]: https://github.com/MontanaFlossCo/FlintDemo-iOS
