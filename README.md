# Javascript Asynchronous Context

## Overview

Javascript Asynchronous Context is the ability to understand the sequence of asynchronous function invocations that resulted in a specific point in program execution.  This document explains the high-level concepts by which we can track async context, and starts to explore where these pieces fit into the TC-39 spec.

## Why Solve This?

The concept is foundational to JS programming, yet there is no shared concepts and terminology and no formal specification.

Being able to reason about async context  is useful from the runtime through user code and in tooling scenarios.

Some examples include:

  - Continuation Local Storage
  - Async call stacks
  - Debugger actions across async boundaries (e.g., step-into async)
  - APM reporting
  - Memory allocation/reference analysis (e.g., being able to detect when memory references escape an http request)

## Existing Efforts

Multiple approaches have been attempted at solving this problem, but they all had various problems:

  - Monkey Patching
    - Various ad-hoc approaches – module load interception and special cases for async-await
    - Breaks when monkey-patched APIs change
  - Domains
    - Conflated async call flow w/ exception handling
    - Explicit enter()/leave() required
  - Async-Hooks
    - Life-cycle events on Node.js “resources”
    - Exposes Node implementation details
    - Node.js only  - No corollary to browser, other hosts.
    - Native->JS transitions and lifecycle events in model are expensive
  - Academic Investigations
    - “Finding Broken Promises in Asynchronous JavaScript Programs” – Alimadadi et al., 2018
    - “Semantics of Asynchronous JavaScript” – Loring et al., 2017

## Concepts


1.  **Async API** - is an API that takes a function as a parameter, and invokes that function asynchronously.   Examples are `setTimeout()`, `Promise.then()`.
2.  **Continuation** - a special type of JavaScript function that is passed as a param to an *Async API*.
3.  **Context** - a structure created when a *Continuation* is invoked.  May act as a property bag for storing arbitrary key/value pairs
4.  **Link Context** - pointer from a **Continuation instance** to the **context** where the continuation was created.
5.  **Ready Context** -  pointer from a **Context** to the **Context** where a promise reaction is "triggered"
6.  **Async Call Graph** - DAG resulting from *contexts* & *continuations* (nodes) and *Link Contexts* and *Ready Contexts* (edges).
      - Directionality of edges is important, they point "up to the root"
      - Gives us a "path to the root" for any continuation/context
      - Does not let us traverse from root down (unless you transpose the graph)
      - Has nice property that GC will clean up any contexts that have no pending children.

##  Possible Implementation Approaches

There are two possible approaches to an implementation.  One approach is to find the minimal set of JS runtime changes to support implementations that realize the above model.  A second approach is make the above model fully realized at the VM level.  There are pros & cons associated with each approach.

The "minimalist" approach has the benefit of less impact at the VM level, fewer points of controversy, and allows implementation & experimentation at both JS/userland, as well as native inside the runtime.

The "deep" approach risks bogging down in details, but has the benefit of making the concepts first-class citizens inside the language spec & runtime.


### "Minimalist" Approach

A high-level view of changes the minimalist approach:

  - Support an API "continuify"
  - Support a "Continuation" function
  - JS Runtime will raise events:
    - "Link Context" - when `Continuation` is created
    - "Execute Begin" - when `Continuation` invocation is starting
    - "Execte End"  - when `Continuation` invocation is completing
    - "Ready Context" - when a `Continuation` is "enqueued" for pomises and async/await
      - promise reaction jobs enqueued
      - async-await
      - async generators?
  - Expose API to attach/remove listeners to events
    - Native API?  Or JS Only?


#### Observable Changes with "Minimalist" Approach

  - new API "Continuify"

```js
// Given function f, transform f into a Continuation
function continuify(f) {
  if (isContinuation(f)) {
    return newContinuation(f);
  }
  return f;
}
```

Open Questions:
  - how do we reflect properties of f through a continuation?
  - is "continuify()" a global?

#### 262 Spec Changes

  - Promises
  - Async/Await
  - ???

### "Deep" Approach

A high-level view of changes to support the "deep" approach

  - Support an API "continuify"
  - support an API "getCurrentContext"
  - Invocation of a "Continuation" is special
    - Runtime will keep track of notion of "current context"
    - e.g., a special "continuation frame" on stack
    - it will establish a "context instance"
  - No events come out of runtime.

#### Observable Changes with "Deep" Approach

- Continuation type
  - "invocable" like a function
  - "has-a" link-context property
  - need to reflect all properties of the function it

#### TC-39 "Execution Context"

- https://www.ecma-international.org/ecma-262/9.0/index.html#sec-execution-contexts
- need an updated definition to for this that differentiates between "regular stack frames", and "continuation frames" (i.e., the frame introduced by a "continuation")

## Unsolved Problems

### "Async Recursion"



```js

const f = () => {
  // do something...
  setTimeout(f, 1000)
}

f();
```

- This results in an infinitely long chain of contexts.
- Can we do some form of "path compression" on this?
- Algorithm:
  - Given an Async Call Graph `acg`


```
// TODO:  draw a picture that shows graph acg with edge directionality pointing "up to the root"
```

- Traverse `acg`, and for each node, keep a sum of number of inbound edges.
- For all "root nodes" of `acg`, walk the "path to the" root
  - if a path in `acg` has a length greater some threshold `n`, and all nodes in the path have an inbound edge count of 1, then "compress" the nodes
