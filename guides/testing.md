---
title: Unit Testing your Flint Apps
tags: guide
---

Yes, we will have lots of unit tests on Flint itself. Not just yet though as we don't want to waste time when the API may still fluctuate significantly. Once there has been time for people to try out and comment on the APIs, things will settle down and the tests will be put together over the coming weeks.

Testing of your Flint-based code should be simple enough - but we likely have to build a few mock classes for you. Work will be ongoing here. 

Actions should be simple enough to unit test. You create an `[ActionContext](blob/master/FlintCore/Actions/ActionContext.swift)` and pass it to the action's `perform` function, along with your test input and presenter.

Examples will follow soon.
