---
layout: post
title: "The making of a tool : emulate Unity Gaming Services back-end locally to reduce iteration time"
excerpt_separator: <!--more-->
slug: "making-a-tool-emulate-ugs-locally"
---
How I made a tool that emulates the Unity Gaming Services REST API to reduce iteration time when working on a multiplayer game.

I created a local emulator for Unity Gaming Services to speed up development on the multiplayer rowing game, **VirtuRow**. By mimicking the main UGS APIs and behaviors, I was able to test features instantly without waiting for cloud deployments, making iteration much faster and smoother.

<!--more-->
__________
* Toc
{:toc}
__________

> In this article, I talk about work I've done for my company. I'm not able to provide any source code or build.

At my current company, we are working on a gamified training sports app for rowing, called **VirtuRow**. The app allows users to row solo or online, in 3D environments.

It is aimed at amateur rowers who might need extra motivation—a gamified app can help—or seasoned rowers who want to train with their mates online, track their progress, and have a bit of competition.

[![Screen 01 - Paris](/assets/articles/01_UnityABL/Screen01.jpg){: width="350"}](/assets/articles/01_UnityABL/Screen01.jpg)
[![Screen 02 - Lucerne](/assets/articles/01_UnityABL/Screen02.png){: width="350"}](/assets/articles/01_UnityABL/Screen02.png)
[![Screen 03 - In game](/assets/articles/01_UnityABL/Screen03.png){: width="350"}](/assets/articles/01_UnityABL/Screen03.png)
[![Screen 04 - Menu](/assets/articles/01_UnityABL/Screen04.png){: width="350"}](/assets/articles/01_UnityABL/Screen04.png)

## Context

To support its online features, we opted to use solutions from Unity.

For the netcode of the game, I decided to use the package [Netcode for GameObject](https://docs-multiplayer.unity3d.com/netcode/current/about/). And for the online services, which is what I want to talk about in this article, I decided to use the [Unity Gaming Services](https://docs.unity.com/ugs/en-us/manual/overview/manual/getting-started) solutions.

Right now, VirtuRow makes use of:
- [Lobby](https://docs.unity.com/lobby/): a player can create or join a game session, with other players they know
- [Matchmaker](https://docs.unity.com/matchmaker/): match players together, from different lobbies, in the same game server
- [Multiplay](https://docs.unity.com/ugs/manual/game-server-hosting/manual/welcome/): manage game server allocation, on which a build of the game runs
- [Cloud Code](https://docs.unity.com/cloud-code/): game services code running on Unity's back-end

The game also uses the **Unity SDK** to interact with those services.

## Iterating on online features

Unity Gaming Services provide great tools. It is really easy to get started and have something working.

However!

The developer experience suffers a lot of friction and iteration time, which overall makes working with these tools very irritating.

For example, with the services we use on VirtuRow, the process is something like so:

- Testing new Cloud Code changes
    - Build the Cloud Code module
    - Upload it on the UGS back-end with a command-line tool
    - Wait for it to be deployed and synchronized
    - On the game client, make a call to the Cloud Code function that must be tested
    - Wait for the result
    - If something is not right (a bug or whatever), you must check the logs on the UGS dashboard *(and the UX is not great here)*
    - Then make the appropriate fixes, and do that again …
- Working on the server side of the game, implementing a new feature or testing the proper flow of the client/server communication is very long
    - This is especially true when using Matchmaker, or when we need to test the code path that must run when the build is hosted on Multiplay
    - The Unity server app must be built (and it can take several minutes to complete, if everything goes right)
    - Then, the build must be uploaded on the Multiplay service, using a CLI tool or from the UGS dashboard (which has a wonky UX too …)
    - Then, the client game must schedule a new server allocation or create a new Matchmaking ticket
    - Then, wait for the server instance to be available
- And for Matchmaker
    - It requires having a Multiplay server configuration, so the same difficulties as before apply
    - Which in turn makes iterating on matchmaking rules configuration very long
    - And the Matchmaker service can take a while to fulfill a match request, adding a bit more friction to the process

When combining all of that, there are situations where testing a single line change can take like 20 minutes or more …

![Tired](/assets/articles/02_UGSEmulator/Meme_RDJ.jpg){: width="150px" .centered}

## Time to tool

When I started to work on the online features of VirtuRow, I quickly realized it was not livable.

After doing that for a while, I knew there was only one solution: make a tool !

### The plan

The overall plan was to be able to completely cut down the iteration time to 0 _(or almost)_. That made me think that everything that is part of the process should run locally on my machine.

So my idea was to make a **Unity Gaming Services Emulator** that provides the same features and the same behaviours, but runs on my computer and doesn't require any build or upload steps.

But how …

### How UGS works

First, I had to know precisely how the Unity Gaming Services SDK works. The SDK is what allows the game app to interact with the back-end services, so this is the starting point.

Fortunately, UGS is very well documented, and to simplify a lot, the only thing that the SDK does is call a [REST API](https://services.docs.unity.com/).

So … what if … I make my own REST API … and somehow make the SDK use it. :)

[![API Swap](/assets/articles/02_UGSEmulator/Schema01.png)](/assets/articles/02_UGSEmulator/Schema01.png)

### First step: create a REST API

To create a REST API in C#, the easiest way is to use [ASP.NET](https://learn.microsoft.com/en-us/aspnet/core/).

I've no prior experience with it, but it is well documented.

#### Starting simple with Player Authentication and Lobby

The first thing I implemented is the [Player Authentication API](https://services.docs.unity.com/docs/client-auth/). I had to research a bit to correctly understand how JSON Web Tokens work, but it is also very well documented, and for my use case I could take a few shortcuts.

Then, the [Lobby API](https://services.docs.unity.com/lobby/v1/), as lobbies are the main starting point of the game.

There were no real difficulties here—mostly just recreating the same endpoints and data, with simple internal code to store the information.

#### Let's complicate things, with Cloud Code support

Implementing the [Cloud Code API](https://services.docs.unity.com/cloud-code/v1/) endpoints was easy.

The challenge here was to load the Cloud Code Module dynamically, without having to implement special code in it or having a specific build process to work in the emulator.

A Cloud Code Module is bundled in a ZIP archive along with its DLL dependencies.

In the emulator, this is how it is loaded and invoked.

[![Invoke CCM](/assets/articles/02_UGSEmulator/Schema02.png)](/assets/articles/02_UGSEmulator/Schema02.png)

The Cloud Code Module doesn't need specific code to support the emulator. For that, the emulator provides a few custom implementations of certain interfaces:
- `IExecutionContext`: contains information about the current execution context
- `ICloudCodeConfig`: responsible for registering dependencies in a module
- `IGameApiClient`: allows calling other services from within a function
- `ILogger`: a logger that allows sending logs to the Unity editor, and displaying logs in the Console

[More information here](https://docs.unity.com/ugs/en-us/manual/cloud-code/manual/modules/how-to-guides/module-structure#available-interfaces)

#### Another complicated element, Matchmaker

As usual, implementing the [Matchmaker API](https://services.docs.unity.com/matchmaker/v2/) endpoints was easy …

The real challenge was to somehow reproduce the actual logic of the Matchmaker service: matching request tickets based on rules, and with optional relaxations.

I didn't want to fake things here, as the goal was to be able, on the game, to implement, test, and iterate on matchmaking rules.

There is no information about the internal behaviour of the Matchmaker service, only how to use the public API, the available rules, etc.

But with that, after quite some time of trial and error, I was able to implement a functional\* matchmaking logic, that supports all the rules of the original service.

> \* obviously, nothing here is scalable or optimized.  
> This is not the goal—the goal is to emulate the behavior only for a dev environment.
{: .note}

#### Multiplay

The server-side of Multiplay was easy to implement: it is mostly implementing the same API and storing data, with some internal endpoints to actually fulfill a server allocation manually.

The rest of the Multiplay emulation is done in Unity. Essentially, an editor instance can act as a Multiplay server, with the game running in Play Mode.

### Second steps: making a control panel in Unity

On the Unity Editor side, I've made a control panel interface that allows configuring the emulator and getting data from the different services.

**A video showing the different options available in the tool.**

<iframe class="yt-frame" src="https://www.youtube.com/embed/ovHnakBz5Ks" title="Nomisenb Blog - UGSEmulator control panel" frameborder="0" allow="accelerometer; encrypted-media; gyroscope; picture-in-picture" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

**A video of the tool used in the VirtuRow project.**
{: .space}

<iframe class="yt-frame" src="https://www.youtube.com/embed/Oc1-p5yM6Qs" title="Nomisenb Blog - Using UGSEmulator on a game project" frameborder="0" allow="accelerometer; encrypted-media; gyroscope; picture-in-picture" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

The game uses the Unity SDK, and the flow shown here makes use of Lobby, Matchmaker, and Multiplay. We can see the Matchmaker tickets and how the Unity editor instance is used like a Multiplay server instance.

The game code is unaware of the emulator and only interacts with the SDK.

### How it all works together

[![Emulator flow](/assets/articles/02_UGSEmulator/Schema03.png)](/assets/articles/02_UGSEmulator/Schema03.png)

The [Centrifugo server](https://centrifugal.dev/) allows sending messages and notifications to the emulator control panel. For example, this is what allows forwarding logs from the Cloud Code Module to the Unity console.

It is also used by the Unity SDK to communicate with some services, like Lobby, that send events when something happens _(a player joins or leaves the lobby, etc.)_.

## Final thoughts on the project

This tool project was really cool! I had to play with technologies I'd never used, and do some hacky stuff (it's always fun :) ).

This was also the first time I used Unity UIElement to create a tool, and it made it possible to have a nice-looking and easy-to-use interface without too much effort.

That said, what went right:
- The tool made a real difference in the dev process of the online features, iteration time got really shorter
- The tool is easy to use and reliable
    - My implementation of the UGS API is, so far, close to the original
    - Any behaviour inconsistencies can be easily fixed
    - Testing in real conditions (server build on Multiplay, Cloud Code deployed, etc.) confirmed that
- I can spend time implementing the actual code that is used on build, and not manage two code implementations, one for the actual build and one for editor

And what went wrong:
- It took 4 months to get the tool to a production-ready stage, which is a lot considering I'm the only programmer on VirtuRow
- … and nothing else really, this project was nice to work on, with some technical challenges but no real hurdles

Thank you for reading this article!