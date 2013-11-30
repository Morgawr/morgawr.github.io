---
layout: post
title: "Fully Concurrent Game Engine   pt.3"
description: "The idea behind the Cloister Core engine"
category: "Game Development"
tags: ["Cloister", "Gamedev", "Engine"]
---
{% include JB/setup %}

[<-- Part 2](http://morgawr.github.io/game%20development/2013/10/17/fully-concurrent-game-engine---pt2/)

### Time model in a concurrent game engine

>"Everything changes and nothing remains still and you cannot step twice into the same stream" - [Heraclitus](https://en.wikipedia.org/wiki/Heraclitus#Panta_rhei.2C_.22everything_flows.22)

Every game has a world filled with entities. All those entities contain data that exists and mutates over time. The only way to read such data is to take a snapshot of the world, a photo of the current state of an entity, and then operate on it without caring about anything else. Unfortunately, most modern concurrent designs operate on the false assumption of mutual exclusion and the idea that critical sections need to be locked to prevent multiple accesses at the same time. Other paradigms use an actor-based model where messages are sent to a thread-safe interface that replies with the state in a thread-safe manner. 

Both of these approaches, unfortunately, do not scale to large applications with hundreds (or thousands?) of concurrent accesses. What's even worse is that the whole world slows down the more threads you add to the equation: when an actor spends time replying to dozens of requests, its thread's timeslice becomes smaller and, on a soft-realtime constraint like a videogame, it can lead to funny and frustrating bugs. The same idea applies to locking and mutual exclusion: high contention of resources, especially with undefined hierarchies of locking mechanisms, can also cause [deadlocks](https://en.wikipedia.org/wiki/Deadlock) and stalemates.

### Clojure

Enter Clojure, a possible solution to all these problems. [Clojure](http://clojure.org/) is a [Lisp](https://en.wikipedia.org/wiki/Lisp) dialect that runs on the [JVM](https://en.wikipedia.org/wiki/Jvm). There are also ports for CLR, Javascript (ClojureScript) and various other in-development implementations, but that is irrelevant to this post.

Clojure is built and designed from the ground up with a focus on concurrency and data immutability. Its [time model](http://www.infoq.com/presentations/An-Introduction-to-Clojure-Time-Model) is very peculiar compared to traditional paradigms. Thanks to its data persistence and immutability, it is ([almost](http://clojure.org/transients)) always possible to read any variable in the program without incurring into blocking or inconsistent state. It also sports an impressive range of [concurrent semantics](http://clojure.org/concurrent_programming) (Refs, Vars, Agents and Atoms) which make parallel development trivial and straightforward.

This post is not for Clojure [insight and design](http://clojure-doc.org/articles/language/concurrency_and_parallelism.html), however a superficial explanation of the [Clojure concurrency magic](http://www.youtube.com/watch?v=dGVqrGmwOAw) is expected: the trick consists of **separating identities and values**. It really is that simple, especially when the whole environment is immutable and persistent. The only mutable things in Clojure (leaving out the Java-related libraries) are transients and the reference types already mentioned in the previous paragraph. Whenever a new variable is defined, it is paired with a value in the environment. A variable named *foo* whose assigned value is, let's say, *"bar"* is not the same as the value *"bar"*. The identifier assigned with that variable is *foo* and it contains a value that is *"bar"*. If that variable is flagged as immutable, then, *foo* will always refer to *"bar"* in that context and nothing will change. We can now read *foo* from any thread without incurring in consistency problems because that defined value will never change for that identifier.

To allow stateful and mutable data while sticking to this principle, we just have to add a new layer of indirection. Clojure's reference types are defined as an *(identity, container)* pair where the container itself is just a box that can mutate given a specific thread-safe function. Said function changes depending on which reference type is being used. Inside this mutable container there is immutable (and [persistent](http://blogs.msdn.com/b/ericlippert/archive/2011/07/19/strings-immutability-and-persistence.aspx)) data which is the actual state/value of the variable. Changing what's inside the container is always thread-safe through its concurrent semantic interface. Reading the value inside the container is called dereferencing and is always a constant-time operation that does not block or stop any other pending concurrent operation, relieving the system from the stress of having to send and receive messages to/from blackboxed actors. It is very similar to [monads](https://en.wikipedia.org/wiki/Monad_%28functional_programming%29), but arguably easier to work with.

![Test](http://www.morgawr.eu/p/1385817189.png)
_Here's a shamelessly stolen explicative picture about the difference between mutable state and immutable identity, taken from [this page](http://narkisr.github.io/clojure-concurrency) which, admittedly, does a much more thorough and better job at explaining Clojure concurrency than I do._
 
### Agents

Going back to game engine design, there is one reference type in Clojure that meets almost all the requirements for our entities: **agents**. An [agent](http://clojure.org/agents) is a container that provides independent and asynchronous change of its state. 

* **Independent** - It does not care about other agents or reference types in the environment and operates only on its own internal data.
* **Asynchronous** - It does not depend on operations carried to other agents or reference types in the environment, its succession of mutations in time is not related to other reference types. It is a fire-and-forget operation.

Agents operate on a similar approach to the actor model, but with a fundamental difference: the agent itself is data and it receives a function that transform that data. Unlike an actor model, where there's a receive-message loop dispatching for various commands (even to read!), an agent simply reads from a queue of functions in its own separate thread and applies the function's result to its own data. 

To read a snapshot of the state of any agent in our game world, all we need to do is just dereference it. A constant-time operation that does not impact resource contention or interfere with other threads. To change an agent we just send a function that is applied to the agent's state (like, for example, increasing the X,Y coordinates of a moving entity). Under the hood, the Clojure implementation makes sure this is all thread-safe. Without delving into [too](http://stackoverflow.com/questions/1646351/what-is-the-difference-between-clojures-send-and-send-off-functions-with-re) [many](http://stackoverflow.com/questions/11316737/managing-agent-thread-pools-in-clojure) [implementation](http://clojuredocs.org/clojure_core/clojure.core/send) [details](http://clojuredocs.org/clojure_core/clojure.core/send-off): each agent gets a cut of a managed thread pool of N+2 threads (where N is the number of cores available in your CPU). Internally the thread pool is managed by the Java Executor Service given a specific scheduling algorithm (which can be modified for tweaking purposes). 

What is specifically useful of agents is that the order of asynchronous functions is always coherent. If you send two functions in succession to an agent, you're always guaranteed that the first function will be applied before the second one, in a FIFO manner. 

Unfortunately, with this approach there is still one visible flaw that needs to be taken care of: what about coordinated updates between agents? More often than not in a game engine there are hierarchies of entities and sequences of updates that need to be fast and responsive. An example: if the player attacks an enemy, we want to tell the enemy to instantly play the *hurt* animation, without having to wait for that agent's thread to be scheduled and updated god-knows-when, especially if the player needs to act differently depending on the response of the action (enemy is dead? enemy parried? enemy counter-attacked?). Luckily for us there is a solution for this issue and it will be outlined in the next post.

Morg.
