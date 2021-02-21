---
title: "Chunk Stories's new Graphics API (Part 1)"
date: 2018-10-31
---

Chunk Stories's original graphics API was essentially slowly carved out of a set of helper functions and OpenGL OOP wrappers. While it did make OpenGL code more bearable to read, it operated in a very explicit way, expecting more or less direct translation to actual GL calls, making things like implementing instancing tricky and not transparent at all to the end user, not to mention it was still very complex for what it did.

This will all changes in Chunk Stories Alpha 1.1.

The `xyz.chunkstories.rendering` package has been simply deleted and redesigned from scratch into the brand new `xyz.chunkstories.graphics` package. This new API has been designed with the goal of allowing users to express what they want to get on the screen in simple terms, and to include nothing irrelevant as the previous one did. So, how does it work ?

<!--more-->

## The general ideas

Chunk Stories renderer is a frame-graph based renderer. You declare a bunch of buffers, and operations (render passes) to apply on them : Add objects, apply lighting, resolve shadows etc. It's called a graph because there are dependencies, for example you'd want the objects to be rendered before applying lighting to them, and you should draw the transparent objects after the opaque ones. In Chunk Stories Render Passes apply a shader over a set of input/output render buffers, for all the supplied vertex input data.

Next you want to describe your objects to be rendered. Chunk Stories offers a generic way to feed the renderer : The Representations API. In short, some objects ( ie a chair, a player, etc) can be `represented` by the game visuals, the role of a Representation is to state what the attached object should be rendered with. Representations are trees, with leafs made of either lights (point lights, spot lights etc) or model instances. Model instances are the interesting part, they are well `instances` of a model and have per-instance data attached (transformation matrix, custom materials, bone data, ...). That per-instance data can actually be customized to use any class you want, seamlessly. More on that later :)

Lastly a bunch of rendering in this game is `fixed-function`, meaning you shouldn't really be changing how it generates it's geometry or does drawcalls : Decals, cube blocks rendering, far terrain mesh generation and a bunch of stuff like that can be implemented on a per-backend basis without having to deal with a intermediary API in the way. For instance, block rendering will now use either a representation or will be just made of fullsize cubes, with actual meshing techniques left to the implementation code.

## Backends and Systems

Since we are no longer beholden to OpenGL, the plan is to support multiple graphics APIs, namely Vulkan as a new first-class citizen. The idea is to have a fast, lean and efficient Vulkan backend for showing off, and a legacy OpenGL 3.3 backend for all the lower-end hardware that wants a chance at running the game. This means a bunch of systems will be written two times, once for each API. We hope the saner API and reduced cruft will offset this additional work. Speaking of systems, here's how these are defined in the API :

There are two sorts of systems: Drawing Systems and Dispatching Systems. Both are opaque `things` the API knows about and can talk to, but their difference resides in how they communicate with the graphics engine and interface into the render graph : Drawing Systems are `summoned` in a Render Pass, ie somewhere in the render pass declaration someone wrote "I want to draw the decals now", and so the system will feed the renderpass with a bunch of drawcalls. Dispatching Systems on the other hand have the control over the renderpasses, `they` get to submit drawcalls to the Render Passes ("Hey you I have some particles for you to draw")

## Shading language Quality of Life

The following changes are somewhat driven by the adoption of Vulkan but in hindsight they make a lot of change on their own: GLSL for Vulkan bans the usage of loose, non-opaque uniforms (ie `uniform float myFloat`), it requires you to use UBOs/SSBOs for everything. This means (almost) everything becomes an interface block, and swapping uniforms arround means swapping buffers. Ugh that's a lot of boilerplate to write, isn't it ?

Chunk Stories offers a simple but very useful feature to make this as painless as possible: **Automatic generation of GLSL code from Java Classes**. So when you feed the renderer instance data, you don't feed him a ByteBuffer with carefully padded and placed elements after having triple-checked the struct layout in GLSL is equivalent. No, you do this :

```glsl
#version 330

#include struct <xyz.chunkstories.api.Camera>
uniform Camera camera;

#include struct <your.amazing.mod.CustomPerInstanceData>
in CustomPerInstanceData instanceData;

in vec4 vertexPositionIn;

out vec4 vertexColor;

void main() {
    gl_Position = camera.viewProjectionMatrix * instanceData.transformation * vertexPositionIn;
    vertexColor = instanceData.color;
}
```

Want to look at CustomPerInstanceData ? Here it is :

```java
public class CustomPerInstanceData extends InstanceData, InterfaceBlock /** redudant */ {
    Matrix4f transformation;
    Vector4f color;
    float haveYourCake;
    int andEatItToo;
}
```

Could hardly be simpler :) The implementation takes care of the translation seamlessly, you just have to obey a few rules in the classes you use for this, such as having a default constructor and not having recursive datatypes (a class can't have a field of it's own class type, directly or indirectly)

## GUIs should be easy

Lastly I want to mention how UI programming is going to evolve alongside this. CS has it's own tiny GUI framework, with layers, buttons, input boxes etc. I have suffered for weeks trying to make UI scaling somewhat consistent and the resulting code was beyond ugly, with lots of complex manual work to scale every object size and offset and keep track of everything.

In hindsight the solution I found is obvious: Make the GUI code unaware of scaling !

The new solution just lies to the user code, by scaling the GUI screen viewport itself: Let's say an user is running at 1080p and has chosen 2x scaling: The GUI subsystem will report the resolution is 960 x 540 and the drawing code can just ignore the concept of scaling altogether. Then when it comes times to render or read mouse position, we just multiply/divide the data so things end up filling the whole screen. Things like line or font rendering magically benefit from this, and given high-resolution assets every GUI element will look nice and sharp.

That's all for this blog post, for more updates about the game progress please make sure to hang out on our Discord and check out the code on GitHub.
