
# JavaScript Asynchronous Context

## Overview
Javascript Asynchronous Context is the ability to understand the sequence of asynchronous function invocations that result in a specific point in program execution.  That is, given an arbitrary point-in-time of a JavaScript program's execution, we want to know the sequence of "top-level" function invocations that caused the current invocation.

This document explains the high-level concepts (or the model) by which we can track async context, and starts to explore where these pieces fit into the TC-39 spec.

## Principles
We strive to adhere to some basic principles in this document.  We believe that adhereing to these principles will ensure simplicity and help guide the model.

1.  Definitions should flow from well-understood, well-defined constructs in synchronous programming.
2.  Definitions should be independent of the host environment (e.g., Node.js, the browser).
3.  Constructs should enhance JavaScript programmers' understanding of async code flow & async boundaries.
4.  Model should allow for simplified reasoning about async code execution.
5.  Resulting Data Structures and APIs and should support solving common "async understanding" use cases (e.g., long stack traces, continuation local storage, visualizations, async error propogation, user-space queueing...).
6.  It should be straightforward to implement this model at any API layer (i.e., VM, host or "monkey-patched").

## A Simple Example
We'll start with a simple example that is not controversial: 

```javascript
function x(s) { ... }
function myLoggingAPI(s) { x(s); }

function f1() {
    let i = 0;
    let interval = setInterval(function f2() {
        if ( i < 2) {
            myLoggingAPI(`i is ${i}`);
            ++i;
        } else {
            clearInterval(interval);
        }
    }, 1000);
}

f1();
```

Most JS programmers should have an intuitive understanding of what happens when the code above is executed.  We'll build off of this example going forward.  Implicit in this example is an undefined concept of "async context".  We'll build stronger definitions later on, but for now, we'll use "async context" to refer to JS programmers' colloquial understanding of relations between asynchonrous function calls.

There are three interesting constructs in the above example:
  - First, the function `f2` is a function created in one "async context" and passed through an API.  When `f2` is invoked later, it is a "logical continuation" of the "context" in which it was created.
  - Second, the function `setInterval` takes a function as a parameter, and invokes that function later during program execution.  Note that when this parameter is invoked, the call to `setInterval` has completed - i.e., `setInterval` is not on the stack when `f2` is invoked. 
  - Third, `f2` is invoked precisely twice.  These two invocations are distinct.

Let's give these three concepts some names and some initial definitions:

   - An **Async API** - is a function that accepts another function `cb` as a parameter, and `cb` is enqueued and invoked at some later point in time.
   - A **Continuation** is a special variant of a JavaScript function that is passed into an **Async API**.  It retains a link to the "async context" in which it was created.  Upon invocation, it will establish a new "async context", and will provide a "logical continuation" of the "async context" in which it was created.  In the example above, `f2` is a **Continuation**.
  -  A **Context** is a specific structure that is created when a **Continuation** is invoked. A **Context** is *active* for the *period of time* where a `continuation`'s frame is on the stack.  As soon as the continuation frame pops off the stack, then that **Context** is completed. Note that a `continuation` instance can have more than one **Context** associated with it.  In our code sample above, there are precisely two **Contexts** associated with invocations of the `continuation` f2. 

## A lower-level view

At runtime we can view the example above as two distinct call stacks at specific points in program execution.  These look something like this:

```
---------------------------
|   setInterval()         |
---------------------------
|   function f1()         |
---------------------------
|    ...host code...      |
---------------------------
|    ...host code...      |
---------------------------
|    host function A()    |
---------------------------
```

and

```
---------------------------
| function x()            |
---------------------------
| function myLoggingAPI() |
---------------------------
|    function f2()        |
---------------------------
|    ...host code...      |
---------------------------
|    host function B()     |
---------------------------
```

Since we previously made a distinction between a regular JavaScript function and a `continuation`, let's make the same distinction in our pictures of the callstacks:

```
--------------------------- 
|   setInterval()         | 
--------------------------- 
|   function f1()         | 
---------------------------      
|    ...host code...      |      
--------------------------- 
|    ...host code...      | 
===================================
|    Host Continuation A()        |
===================================
```

and

```
---------------------------
| function x()            |
---------------------------
| function myLoggingAPI() |
===================================
|    continuation f2()            |
===================================
|    ...host code...      |
---------------------------
|    ...host code...      |
===================================
|   Host Continuation B()         |
===================================
```

Some notes about the pictures above:
  - We've introduced a "Root Continuation" as the bottom frame.  This  illustrates a basic assumption that all code is executing in the context of a `Context`.
  - Multiple `continuations` can be on the stack at the same time, which result in mulitple `Contexts`.

Here, we've updated our pictures of runtime callstacks, giving  labels to the `Context` instances:

```
---------------------------                    -----------                            
|   setInterval()         |                              |
---------------------------                              |
|   function f1()         |                              |
---------------------------                              \/
|    ...host code...      |                          Context ci1
---------------------------                              /\  
|    ...host code...      |                              |
===================================                      |
|    Host Continuation A()        |                      |
===================================            -----------
```

and

```
---------------------------                    -----------
| function x()            |                              |
---------------------------                              \/                   
| function myLoggingAPI() |                          Context ci3                        
===================================                      /\
|    Continuation f2()            |                      |
===================================            -----------
|    ...host code...      |                              |
---------------------------                              \/
|    ...host code...      |                          Context ci2                   
===================================                      /\
|   Host Continuation B()         |                      |
===================================            -----------
```

## Context States
A `Context` can be in a number of states:
  - `pending` - A  `Context` instance has been created, but is not yet executing, nor is it ready to execute.
  - `executing` - The `Continuation` frame associated with a `Context` is currently on the call stack
  - `completed` - A `Continuation` frame associated with a `Context` is unwound off of the stack.

## Context and Current Context
For any given stack frame, we define it's  **Context** as that defined by the first `Continuation` below the given frame on the stack.  For example, in the previous diagram, the stack frame of `function x()` has a `Current Context` of `ci3`,
and the stack frame of  `... host code...` has a "`Current Context` of `ci2`.  We define the **Current Context** as the `Context` of the top frame on the stack. 

## Link Context
We define the **Link Context** of a `Continuation` to be the `Current Context` when a `Continuation` is constructed.   In the example from above, the `continuation` `f2` has a `link context` pointing to `Context` `ci1`.  Generally, this is the `Current Context` when a `Continuation` is passed into a `Continuation Point`.

## Ready Context
For promises and async/await structures, we define the **Ready Context** to be the `Current Context` when a "promise reaction job is queued". This is useful when want to understand the series of `Contexts` involved in resolution on a promise chain.

For example:

```javascript
const p = new Promise((resolve, reject) => {
    setTimeout(function f1() {
       p.then(function f2() {
             myLoggingAPI('hello');
         });
     }, 1000);

     setTimeout(function f3() {
         resolve(true);
     }, 2000)
});
```

In this code example, we have:
  - `setTimeout` and `Promise.then` as our `continuation points`. 
  - four `continuations` - `f1`, `f2` and `f3`, plus the "root continuation" that 
    the initial code is executing in. 
  - four `Contexts`.

For non-promise-based APIs, the `Ready Context` is the same as the `Link Context`.

In our example above, the `Context` associated with `continuation f2()` has as its `Ready Context`, a reference to the `Context` associated with `continuation f3()`.  

// TODO:  add stacks w/ `Context` to illustrate `ready context`.

## Crisper Definitions

  - **Continuation** -  a function that, when invoked, creates a new `Context` instance.  Also, it contiains a reference to the Continuation where it was created. Using TypeScript for descriptive purposes, a `Continuation` has the following shape:

    ```TypeScript
    /**
     * maker interface for a generic function type
     */
     interface IFunction {
        (...args: any[]): any
    }

    interface Continuation extends IFunction{
        linkData: ContextLinkData;
        originalFunction: IFunction;
    }
    ```

  - **ContextLinkData** - a specific structure used to associate a `Continuation` with the  `Context` where it was created.  This level of indirection allows us to free Continuations and their functions after they can no longer be invoked, while maintaining relevant data inside the context tree.

    ```TypeScript
    /**
      * structure that let GC collect "Continuations" when no longer pinned/queued, but lets us keep their "link data" around 
      * for purposes of the context graph. 
     */
    interface ContinuationLinkData {
        linkContext: Context;
        Object: props; // arbitrary store for key-value pairs
    }
    ```

  - **Context** - 

    ```TypeScript
    interface Context {
        id:  number;
        continuationLinkData: ContinuationLinkData;
        readyContext: Context;
        Object: props; // arbitrary store for key-value pairs
    }
    ```


  - **Async Call Graph** - A directed acyclic graph comprised of `ContinuationLinkData` and `Contexts` instances as nodes, and the `linkingContext` and `readyContext` references as edges. // TODO draw some pictures of the DAG

## Where are the Async APIs?
The `Async APIs` are defined by convention in the host environment, including the VM (e.g., native promise APIs, async/await), host libraries (e.g., setTimeout()), and user-space libraries (e.g., a DB client).  The host needs logic to determine if a given argument is a function or a continuation, and if not yet a continuation, needs to "continuify" the parameter.  The host is also responsible for "pinning" any `Continuation` instances to prevent premature garbage collection.

    ```TypeScript
    // given function f, transform f into a Continuation
        continuify(f) {
            if (isContinuation(f)) {
                return newContinuation(f);
            }
            return f;
        }
    ```


##  Possible Implementation Approaches
There are two possible approaches to an implementation.  One approach is to find the minimal set of JS runtime changes to support implementations that realize the above model.  A second approach is make the above model fully realized at the VM level.  There are pros & cons associated with each approach.

The "minimalist" approach has the benefit of less impact at the VM level, fewer points of controversy, and allows implementation & experimentation in both application space, as well as native inside the runtime.

The "deep" approach risks bogging down in details, but has the benefit of making the concepts first-class citizens inside the language spec & runtime.

### "Minimalist" Approach

A high-level view of changes of the minimalist approach:
  - Support an API for "continuify"
  - Support a first-class "Continuation" function
  - continuify callback params passed to `Promise.then()`
  - 
  - JS Runtime will raise events:
    - `Link Context` - when `Continuation` is created
    - `Execute Begin` - when `Continuation` invocation is starting
    - `Execte End`  - when `Continuation` invocation is completing
    - `Ready Context` - when a `Continuation` is "enqueued"
      - promise reaction jobs enqueued
      - async-await
      - async generators?
  - 
  - Expose API to attach/remove listeners to events
    - Native API?  Or JS Only?

#### Observable Changes at VM level with "Minimalist" Approach
  - new API `continuify()`
    - Open Questions:
      - how do we reflect properties of f through a continuation?
      - is "continuify()" a global?
  - new API to add/remvoe listeners for above events     

#### 262 Spec Changes
  - Promises
    - need to `continuify` promise reactions
    - need to raise `Ready Context` events when spec says to "EnqueueJob("PromiseJobs",...)
      - https://tc39.github.io/ecma262/#sec-enqueuejob 
  - Async/Await
    - ???
  - Async Generators
    - ???
  -  Generators    
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
    - need to reflect all properties of the function it wraps through
  - APIs & Types
    - `continuify()` API
    - `getCurrentContext()` API
    - `Context` - Type
    - `ContinuationLinkData` - Type

#### 262 Spec Changes
  - Promises
    - need to `continuify` promise reactions
    -  when spec says to "EnqueueJob("PromiseJobs",...), we need to 
      - Create the new `Context` `c`
      - Set `c`'s `readyContext` to the value of `getCurrentContext()`
      - Associate c w/ the queued promise reaction, and use that context when reaction is invoked
  - Async/Await
    - ???
  - Async Generators
    - ???
  -  Generators    
    - ???


## Unsolved Problems

### "Async Recursion"

    ```javascript
    f = () => {
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
      //TODO:  draw a picture that shows graph acg with edge directionality pointing "up to the root"
    ```
    - Traverse `acg`, and for each node, keep a sum of number of inbound edges.
    - For all "root nodes" of `acg`, walk the "path to the" root
      - if a path in `acg` has a length greater some threshold `n`, and all nodes in the path have an inbound edge count of 1, then "compress" the nodes  
