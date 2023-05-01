---
title: C2P0 - Introduction
date: 2023-04-30 21:05:58
tags: tools
---

The world needed a new C2 framework (it didn't). Perhaps a better way to phrase that is... I wanted to build a C2 framework. With plenty of paid and open source options for C2 frameworks, it'd be much quicker to work with an existing solution if you needed to get something up quickly for an engagement or wanted to play around. Luckily, I have plenty of time on my hands and wanted to learn the ins and outs of how to not only structure a framework but how to build upon the modularity that any good C2 framework should have (including a custom one).

Thus I present to you, C2P0. C2P0 is a project built upon C# that intends to give red teamers easy tools to build their own implants in whatever language they desire. The currently implemented communication is done over HTTP, but the interface can be extended to support a variety of communication mediums. The agent included in the repository is an example in C# but the listener <-> agent contract can be implemented in whichever language the operator feels comfortable writing a beacon in supporting the communication mechanism of the listener. 

The general idea I had when creating everything, was to keep the listener interface as generic as possible as well as the code generator (for making agents) so that a new operator could come in and create their own listener and generator to allow for communication over new and exciting mediums. Even the HTTP listener can be extended to perform all sorts of wacky obfuscation and encryption to keep blue teamers from being able to easily detect that something bad is going on in their network. For more information, I'll post a wiki on the github repository providing support on use of the framework and examples of implementations as I extend the project. The command and control is in its infant stage as it only allows for command execution between the team server and agent, but this is the first direction I intend to beef up this project shortly. 

The following DEFCON talk and AtlasC2 inspired me to create this:
- https://github.com/Gr1mmie/AtlasC2
- https://www.youtube.com/watch?v=0Z3VadqyFiM

C2P0 doesn't yet provide anything new to the world. However I hope to soon explore fun niches in persistence that a greenfield C2 is optimal for.

Repository:
- https://github.com/victorsch/c2p0