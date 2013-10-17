---
layout: post
title: "Fully Concurrent Game Engine   pt.2"
description: "The idea behind the Cloister Core engine"
category: "Game Development"
tags: ["Cloister", "Gamedev", "Engine"]
---
{% include JB/setup %}

[<-- Part 1](http://morgawr.github.io/game%20development/2013/10/15/fully-concurrent-game-engine---pt1/)

### Design Theory

Before delving into actual implementation and technical details, let's have a moment to reflect on the possible design of a fully parallel and concurrent game engine and what could be a simple development model for it.

We want to achieve the following goals:
- Easy to understand
- Easy to develop for
- Entities are loosely coupled with each other
- No need to think about threads

There are various paths that a developer can take in order to reach the state of a fully concurrent game engine, however I feel it is of the utmost importance to keep everything as simple as possible. It is no use if we can have a massive engine at the cost of nobody being able to understand it. It would eventually become frustrating and too hard to maintain, it would just lose in popularity and become [yet another stigma](http://www.blachford.info/computer/articles/CellProgramming1.html) in the history of multithreaded game development (I actualy love the Cell processor, don't get me wrong).

Many people don't fully realize it, but we live in a concurrent world. If we just look around, we can see a lot of tiny processors everywhere, they are called **humans**. How do they do it? How do they move around such a massively complex world without synchronizing with each other, without having to understand everything to take the next step in the ginormous computation of this universe?

If you think about it, the solution is actually very simple: they don't. There is no need to schedule their life in time slices to share with everybody else, dictated by a globally synchronized clock. Imagine you are driving on a highway with dozens of cars in each lane. There are cars in front of you, cars behind you, cars at the sides. Traffic is a very complex system that is case of study for multiple researchers all over the world. Imagine that, we are a part of it! We, ourselves, are actually creating a vastly complex system and we don't even realize it. And yet, what do we do while we drive? We just focus on the road. We have a target in mind, we don't care about what happens around us, we only care about the cars directly in front and behind us (also the close surroundings, drive safely!). We don't need to know that a few miles from us Fred is watching TV shows on his brand new 4K screen, why would that matter to us at all?

There is also another really important point to make: we cannot stop the world around us whenever we interact with it. We do not control the car behind us. You want to read that billboard? Welp, too fast, you could just glance at it once and now it's gone. You don't lock the world in place just for your own comfort. Imagine everybody locking everybody else, the whole system would grind to a halt, it would be unmanageable. Yet, this is exactly what a lot of multithreaded systems resort to: **locking the world every time you interact with something is not the solution.**

Let's summarize this, there are two fundamental key points to remember:
- **Entities care only about themselves**
- **The world must never stop running**

The first point is not actually that hard to achieve, many game systems have been developed already with this in mind. Any classical Actor/Entity based system deals with entities being their own little world. There can be various grades of technical in-breeding for these engines, some may go with a fully compartmentized and heavily OOP driven approach while others prefer to go with data driven development, thinner and more spread out. Modern engines seem to be preferring the latter approach but the former certainly has its benefits too.

### Towards a (not so) event-driven system

What we really want, though, is what people call an event-driven system. Traditionally in such systems, entities are blocks of code designed to respond to events and interact with the world via callbacks. This starts out as a very elegant model but, from my personal experience, it quickly becomes hard to maintain and operate (opinions may vary!). Did this event come from that entity or that other one? Which event was fired first? Should I reply to this event before or after the end of this update loop?

In my opinion, the problem behind an event-driven system is not being able to take it to the next level of abstraction. Developers are still tied to a classic update loop firing events at X frames per second. We need this because we need to provide a temporal ordering of actions in our game engine. We certainly don't want our ball to pass through a brick simply because the "move" event was fired before the "collision" event in [breakout](https://en.wikipedia.org/wiki/Breakout_%28video_game%29). It would be disastrous. I have yet to see an accident being prevented because a car was able to pass unscathed through another one.

What we ultimately need is a way to provide asynchronous computation while maintaining the order of operations, we need to abstract away from the traditional timed update loop. Just provide these entities with the right behaviors and let them do their job, they surely know how to handle it better than we do, as long as the system is consistent.

For this purpose, we will now start referring to entities as *agents*, the reasoning behind this will be more clear in the next post.

### Stop the world, I need to pee

The second point we need to consider is stopping the world. In a classic concurrent environment we need to make sure the shared data we are accessing from one thread is in a consistent state throughout the whole computation else problems could arise. This means that I need to prevent other threads from messing up with my data for the quantum of time I need to operate on it myself. This is called [mutual exclusion](https://en.wikipedia.org/wiki/Mutual_exclusion). Let me get this straight to you: **mutual exclusion is bad**. It implies some privileged flow of execution that overtakes other threads and expects them to stop for it to continue. It is unreasonable, as I mentioned earlier. You should not expect traffic to slow down just because you want to take another look at the cutie waiting at the crossing light.

How do we implement this? How can I go mingle data in a fully concurrent environment without blocking others or fearing my data to suddenly become corrupt and unusable? There is a solution, and it is what I will be proposing in my next post. 

Morg.