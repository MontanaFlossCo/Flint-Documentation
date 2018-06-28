---
title: Testing
subtitle: Unit testing your Flint Apps and Flint itself
tags: guide
---

Flint has been designed to make your code more testable, and to be well tested itself.

We will have lots of unit tests on Flint itself, but not just yet as we don't want to waste time when the API may still fluctuate significantly in the run up to 1.0 final release. Once there has been time for people to try out and comment on the APIs, things will settle down and the tests will be put together before the 1.0 final.

Testing of your Flint-based code should be simple enough - but we likely have to build a few mock classes for you. Work will be ongoing here. 

Actions should be simple enough to unit test. You create an `[ActionContext](blob/master/FlintCore/Actions/ActionContext.swift)` and pass it to the action's `perform` function, along with your test input and presenter.

Examples will follow soon.
