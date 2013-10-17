---
layout: post
title: "Fully Concurrent Game Engine - pt.1"
description: "The idea behind the Cloister Core engine"
category: "Game Development"
tags: ["Cloister", "Gamedev", "Engine"]
---
{% include JB/setup %}

[Part 2 -->](http://morgawr.github.io/game%20development/2013/10/17/fully-concurrent-game-engine---pt2/)

### Introduction

Game engine development is a very interesting and complex subject for game developers. There is something incredibly appealing in trying to cram a highly modular, flexible and yet fast design into the quasi-realtime requirements of a game engine. 

There is really so much you can do inside a tiny quantum of time, as tiny as 16.7ms for most requirements. Even when halving the framerate to ~33ms you're still going to reach a limit to what you can do. There are a lot of ways to deal with timestepping and separation of rendering and updating and [this is not the place to talk about it](http://gamesfromwithin.com/casey-and-the-clearly-deterministic-contraptions). What I want to talk about, however, is concurrency problems.

In 2013 (been a while, actually) processors have pretty much hit their physical limit as far as computational speed goes. Engineers have been coming up with the most convoluted solutions in order to cram even more juices out of a small die, but the common trend is to go with more cores. If you cannot get a single core to run faster, why not just double it? Double the cores means double the performance! ([It's not actually true](https://en.wikipedia.org/wiki/Amdahl%27s_law)) The line of reasoning is sound, though: with more cores your machine can handle more threads, which are parallel flows of execution, gaining more performance out of parallelism.

How game development relates to multiple cores, though, is a totally different story. A lot of cores mean you can run a lot of applications at the same time in a better way than you'd do with just a single core and as long as each application lives in its own world, and doesn't require interaction with the others, you should expect good speedups in performance. As any game developer can attest, this is not the case for game engines. A videogame is a very complex system composed with a high number of entities all operating together, synchronized and aware of the world, requiring a single timestep and multiple computations to be applied to more than one entity at the same time. This is **nightmare** for multithreading. 

I won't go into details about issues with multithreading, but it is important to realize that the fundamental problem with highly synchronized multithreaded systems is the state. The moment two threads try to access the same shared resource at the same time, the consistency of the state of said resource is at risk and this should always be kept in mind when designing a multithreaded game engine. Another substantial problem has to do with the update rate of entities. One entity should not be allowed to be faster than the others, it should not "get ahead" of the group, and with concurrency in the mix you really need to do a lot of over engineering to solve this issue, up to the point of losing performance because of it.

On the one hand, concurrent game engines seem impossible to achieve, however on the other hand there are various portions that can be safely parallelized without incurring in the wrath of the Nondeterministic Gods. Modern AAA engines (like the [Unreal Engine](http://udn.epicgames.com/Three/ActorTicking.html) or the [Doom3 Engine](http://fabiensanglard.net/doom3_bfg/threading.php)) already separate some portions of their code in different threads. The most common approaches involve having the rendering, physics, input/networking and AI all on their own threads, and then successively synchronized update cycles to the whole game world. 

How to further parallelize the job of a game engine is still [a very hot topic of research in the game industry](http://dice.se/publications/parallel-futures-of-a-game-engine/) with, unfortunately, little to no actual solution readily available to the public. Wouldn't it be great to just be able to develop a game with little to no concern about the underlying multithreaded architecture or CPU structure while still gaining performance boosts from multiple cores? 

With this series of blog posts I am going to provide a rationale and design concretization of my game engine idea, the Cloister Core engine. It is a fully concurrent game engine I am currently developing alongside these blog posts. I don't know where this journey will take me and if this is a sensible design, however I think it's going to be fun nonetheless.

Morg.
