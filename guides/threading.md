---
title: Threading
subtitle: How actions are performed in a concurrent environment
tags:
    - help
    - actions
---

#### In this article:
{:.no_toc}
* TOC
{:toc}

## Overview

* Actions indicate which queue they should be called on
* Completion Requirements indicate which queue they should be called on
* Flint makes sure these are try no matter what happens
* Caveat: You cannot use it with queues that have GCD target queues and expect this behaviour to work unless you're on exactly the correct queue i.e. a new queue with `main` as target queue will not be detected as being the `main` queue currently.

## Defining the queue on which an Action will be called

## Defining the queue on which Action completion will be called

## Actions that complete asynchronously

