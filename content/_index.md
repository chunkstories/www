---
title: "Home"
---

![](/images/header.jpg)

Chunk Stories is a [free and open source](https://github.com/Hugobros3/chunkstories/blob/master/LICENSE.MD) block game and engine written on the JVM using Kotlin and Java. It features a high amount of attention to the extensibility and moddability of the engine, and an unreasonnable amount of bikeshedding since the start of the project in 2015.

# Goals

As outlined in more detail in the [README](https://github.com/Hugobros3/chunkstories/blob/master/README.md) of the project, we aim to make a decent block engine that is also a competent substitute for Minecraft Java edition, given the sometimes questionnable decision-making of the new IP holders. But that's not it of course.

CS is also a long-standing experimentation & learning platform, and has seen constant large-scale refactorings. The project started out in the ancient times of *2015*, and continuously got improved as a better understanding was gained of every aspect. The current state of the project is, unsurprisingly, unfinished.

# Status

Some major re-thinking is in order currently, as we have now reached a point where we think the best course of action is to stop and think about programming languages. A game engine can be viewed as a tiny operating system onto it's own, or rather a platform for other applications to be written in.

Whether you set out to or not, you inevitably end up creating domain-specific languages when writing such a thing. We think there is a large opportunity for QoL & simplicity gains by consolidating those DSLs into a single, or at least a cohesive family of languages. The JVM has served us right, and may continue to do so, but we have started looking into [going further](https://Hugobros3/worm-lang/).

![](images/csos.png)
*Chunk Stories OS, coming september 2038*
