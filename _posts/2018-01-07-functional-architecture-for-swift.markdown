---
title: "Functional architecture for Swift"
layout: post
date: 2018-01-07 23:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- functional programming
- swift
- ArchitectureKit
- FunctionalKit
- Swiftz
- Monad
category: blog
author: Ricardo Pallas
projects: true
description: Simplest architecture for FunctionalKit
# jemoji: '<img class="emoji" title=":ramen:" alt=":ramen:" src="https://assets.github.com/images/icons/emoji/unicode/1f35c.png" height="20" width="20" align="absmiddle">'
---


![](https://cdn-images-1.medium.com/max/1600/1*6rHrtpuEEnQj19mOUpS-pQ.png)

In this article I am going to introduce a library for architecting iOS apps,
called ArchitectureKit: 

> “Simplest architecture for FunctionalKit”

### Table of contents

1.  Introduction
1.  Motivation
1.  ArchitectureKit
1.  Dependency Injection
1.  Full example
1.  Conclusion 

### 1. Introduction

[ArchitectureKit](https://github.com/RPallas92/ArchitectureKit) is a library
that tries to enforce correctness and simplify the state management of
applications and systems. It helps you write applications that behave
consistently, and are easy to test. It’s strongly inspired by
[Redux](https://redux.js.org) and
[RxFeedback](https://github.com/NoTests/RxFeedback.swift).

### 2. Motivation

I have been trying to find a proper way and architecture  to simplify the
complexity of managing and handling the state of mobile applications, and also,
easy to test.

I started with
[Model-View-Controller](https://developer.apple.com/library/content/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html)
(MVC), then
[Model-View-ViewModel](https://msdn.microsoft.com/en-us/library/hh848246.aspx)
(MVVM) and also
[Model-View-Presenter](https://en.wikipedia.org/wiki/Modelâviewâpresenter)
(MVP) along with [Clean
architecture](https://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/).
MVC is not as easy to test as in MVVM and MVP. **MVVM and MVP are easy to
test**, but the issue is the **UI state can be a mess**, since there is not a
centralized way to update it, and you can have lots of methods among the code
that changes the state. 

Then it appeared [Elm](https://guide.elm-lang.org/architecture/) and
[Redux](https://redux.js.org) and other Redux-like architectures as
[Redux-Observable](https://github.com/redux-observable/redux-observable),
[RxFeedback](https://github.com/NoTests/RxFeedback.swift),
[Cycle.js](https://cycle.js.org), [ReSwift](https://github.com/ReSwift/ReSwift),
etc. The main difference between these architectures (including ArchitectureKit)
and MVP is that they **introduce constrains of how the UI state can be updated,
in order to enforce correctness and make apps easier to reason about.**

Which make ArchitectureKit different from these Redux-like architectures is it
uses feedback loops to run effects and encodes them into part of state (we will
see this in point 3) and uses monads from FunctionalKit to wrap the effects.

**ArchitectureKit runs **[side
effects](https://en.wikipedia.org/wiki/Side_effect_(computer_science))** for
you. Your code stays 100% pure.**

### 3. ArchitectureKit

ArchitectureKit itself is very simple.

#### The core concept

Each screen of your app (and the whole app) has a state itself. in
ArchitectureKit, this state is represented as and object (i.e. Struct). For
example, the state of a TO-DO app might look like this:

This State object, representes the state of the “List of TO-DOs screen” in a
TO-DO app. The “todos” var contains all the to-dos that might be drawn in the
screen and “visibilityFilter” tells what todos should appear in the list.

With this approach of having an object that actually represents the state of a
screen, **views are a direct mapping of state**:

> `view = f(state)`

That “f” function will be the UI binding function that we will see later on.

To change something in the state, you need to dispatch an **Event**. An event is
an enum that describes what happened. Here are a few example events:

Enforcing that every change is described as an event lets us have a clear
understanding of what’s going on in the app. If something changed, we know why
it changed. Events are like breadcrumbs of what has happened. Finally, to tie
state and actions together, we write a function called a reducer. A reducer it’s
just a function that takes state and action as arguments, and returns the next
state of the app:

> **(State, Event) -> State**

We write a reducer for every state of every screen. For the list of todos’
screen:

Notice that the reducer is a pure function, in terms of [referencial
transparency](https://en.wikipedia.org/wiki/Referential_transparency), and for
state S and event E, it always return the same state, and has no side effects.

This is basically the whole idea of ArchitectureKit. Note that we haven’t used
any ArchitectureKit APIs. It comes with a few utilities to facilitate this
pattern, but the main idea is that you describe how your state is updated over
time in response to events, and 90% of the code you write is just plain Swift,
so the UI logic can be tested with ease.

But what about asynchronous code and side effects as API calls, DB calls,
logging, reading and writing files?

#### AsyncResult and FunctionalKit

The  `AsyncResult`** **data structure is used for handling asynchronous
operations. AsyncResult is just a  `typealias` to a `Reader<Future<Result>>`
monad stack. These monads (and its monad transformers) are available in
[FunctionalKit](https://github.com/facile-it/FunctionalKit), which is the only
dependency in ArchitectureKit.

FunctionalKit provides basic functions and combinators for functional
programming in Swift, and it can be considered a extension to `Foundation`. We
mainly use the Reader monad along with the Future and Result.

* [Reader
monad:](https://github.com/facile-it/FunctionalKit/blob/master/Sources/FunctionalKit/ReaderType.swift)**
**it is used in the top of the monad stack to provide a way to inject
dependencies. We will see it in depth later.
* [Future
monad:](https://github.com/facile-it/FunctionalKit/blob/master/Sources/FunctionalKit/FutureType.swift)
it is used to represent async values.
* [Result
monad:](https://github.com/facile-it/FunctionalKit/blob/master/Sources/FunctionalKit/ResultType.swift)
it represents if a computation was successful or there was an error.

We use monad transformers to create AsyncResult as a stack of these three
monads. **An AsyncResult is a monad that represents an asynchornous operation
that returns either a successful value or an error, and also provides a
mechanism for dependency injection.**

We can see in the following snippet an example of Facebook login using
AsyncResult:

To create an AsyncResult we use its static method  `unfoldTT` (the TT stands for
transformer, since it is a monad transoformer). It expects a function as a
parameter which has two inputs: an environment or context and a continuation or
callback. The environment parameter comes from the Reader monad and it is an
object that contains the injected dependencies. The continuation parameter is a
callback function that must be called with the   `Result` value returned from de
async operation. In the example the Result returns an   `string` when it
succeeds. When the login succeeds, we call the continuation method with a
sucessful `Result` with the token from Facebook. It the login fails, we call the
continuation method with a failure `Result` that contains the error.

The AsyncResult must me parameterized with 3 values. First one is the
Environment type (which contains the dependencies), second one is the actual
value expected from the async operation (in the example we use    `string`
because we expect the facebook login to return the login token), and the last
one is the error type the  `Result` will return if something goes wrong.

Every asynchronous operation and side effect must be performed using the
AsyncResult monad and we will use Feedbacks from ArchitectureKit to execute the
their side effects. Also, we will see how to work with AsyncResults in the full
example.

#### Design feedback loops

Let’s add a new feature to our previous TO-DO app! We want let users to save
their TO-DOs to the cloud. That would require an network call, which is a side
effect and it is asynchronous, so to achieve it, we will use feedback loops. The
way of dealing with effects in ArchitecrueKit is encode them into part of state
and then design the feedback loop.

A feedback loop is just a computation that is triggered in some cases, depending
on the current state of the system, that launches a new event, and produces a
new state.

A whole ArchitectureKit loop begins from a
[UserAction](https://github.com/RPallas92/ArchitectureKit/blob/master/ArchitectureKit/UserAction.swift)
that triggers an event. Then the reducer function computes a new state from the
event and previous state. ArchitectureKit checks if any feedback loop must be
triggered from the new state. If so, the feedback produces a new event
asyncrhonously (by executing side effects) and a new state if computed from the
feedback’s event.

So, we can see a whole ArchitectureKit loop as the following sequence:

1.  UserAction produces an event.
1.  reducer(Current state, event) -> new state.
1.  Query new state to check if feedback loop must be triggered.
1.  if so, new event triggered (side effects executed).
1.  reducer(new state, new event) -> newer state.
1.  Repeat from step 3 until no more feedback (or maximum of 5 feedback loops)

![](https://cdn-images-1.medium.com/max/1600/1*HHbqRbOi9HHBrbwFahoCAQ.png)
<span class="figcaption_hack">ArchitectureKit whole loop</span>

In the following code snippet, we can see a Feedback example of how to store the
user’s TO-DOs in the cloud:

For implementing this feature, two new events have been added  `storeTodos()`
and `todosStored(Bool)` and there is a new Bool variable in the state: 
`mustStoreTodos`. The  `storeUserTodos(todos:[Todo])` function is the function
executed in the feedback loop, which returns an AsyncResult monad that returns
the  `todosStored(Bool)` event when side effects are executed. This function is
in charge of storing the user’s TO-DOs.

A Feedback object is composed by two functions that receive the current state as
parameter. The first function is the actual AsyncResult to be executed, and the
second one checks when the feedback loop must be executed, depending on the
state. In the example, the user’s TO-DOs feedback will be executed when  
`mustStoreTodos` variable is true.

In the new reducer, the `storeTodos()` event is setting    `mustStoreTodos` to
true, and  `todosStored(Bool)` is setting it back to false. The`storeTodos()`
event will be triggered by an UserAction, like tapping a button.

The following diagram illustrates the steps for storing the user’s TO-DOs:

![](https://cdn-images-1.medium.com/max/1600/1*gs6P6pEHxK6_iSNNaV4WbQ.png)
<span class="figcaption_hack">How Feedback loop is executed after an UserAction</span>

#### Who dispatches events? UserActions

[UserAction](https://github.com/RPallas92/ArchitectureKit/blob/master/ArchitectureKit/UserAction.swift)
is the object from ArchitectureKit that represents any action from the User or
the iOS framework that triggers an event that changes the state (and from that
state change it could trigger a feedback loop). 

It has two methods: 

* `init`: creates the UserAction and specifies what event will be triggered when
user actions is executed
* `execute`: executes the user action.

#### Simple example

We can see here a simple example of how ArchitectureKit’s code would look like:

It’s a simple counter with an increment and decrement buttons. The State is just
an integer that contains the current count.

![](https://cdn-images-1.medium.com/max/1600/1*Mk6XqMNScFYorwUbdrHEuQ.gif)

### Dependency injection

see Jorge Castillo article in Kotlin using Kategory

### Full example

also add a diagram

key: view = f(state) direct mapping between state and view

ArchitectureKit runs side effects for you. So your code stays 100% pure

### Conclusion

(when I would use ArchitectureKit)

difference ArchitectureKIt uses Feedback loops, i remconed use it with
Functioanl Clean Architecture, it runs the side effects

but Functional clean architecture (functions and no objects, except objetcs for
dependency inbjection and protocols)

next steps: create user actions for every UIKit control
