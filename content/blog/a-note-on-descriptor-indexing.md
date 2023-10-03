---
title: "A note on Descriptor Indexing"
date: 2019-01-26
---

*Was writing part two of the series on Chunk Stories new rendering tech, but this section was getting out of hand so I just spun it off as it's own post. Here's a very technical post intended at Vulkan developers wondering about VK_EXT_descriptor_indexing*

Talking with some folks on rendering-related discords, I had a few conversations about the way I solve the texture binding problem in my engine, and there has been some confusion, notably due to a really crappy naming convention used since OpenGL and unclear terminology in the spec ( those are almost the same problem come to think of it ... ). I sort of needs to lay this out flat before I can proceed.

<!--more-->

 * In OpenGL and Vulkan, there is a texture object type called "Array Textures". These are textures with many "slices" or "layers" you can index in your call to texture() in GLSL, as a sort of third dimensions. The difference with actual 3D textures is you can't interpolate between two slices. Layers of array textures share the same size and the number of layers is predefined. These textures use the `sampler2Darray` type in GLSL.

 * In GLSL you can also declare an *array of textures*, like so: `uniform sampler2D myTextures[8];`. These are *separate* texture objects put into an array, they don't share anything but the name of the array in GLSL.

 The similarity between these two wildly different things makes it very hard to have an understandable discussion, especially on Discord where the conversation can get very noisy. This is why I propose an alternative naming convention for Array Textures: since those have layers, like onions (and ogres), they should really be called **Onion Textures**, and is what I'll be referencing those as from now on. Here's a summary table:

| What people call it | What it does | How GLSL calls it | What it should be called |
 | --- | --- | -- | -- |
 | Array Textures / *Textures Array* | It's a big texture with many slices/layers of the same size | `sampler2DArray` / `texture2DArray` | Onion textures. |
 | Array *of* textures / Textures arrays | It's a bunch of textures referenced in an array | `sampler2D t[8]` / `texture2D t[8]` | Array of textures. |

![](/images/blog/onion_textures_know_the_diff.svg)
*If you sign my change.org petition I think we can make Khronos update the spec to call these onion textures by summer.*

## The texture binding problem

Texture binding is one of these nasty things you need to stay on top of in realtime graphics today. The classical approach is to have a bunch of textures declared as opaque uniform variables and swap them in-between draw calls. Unfortunately, even if Vulkan lowers the cost of draw calls dramatically, rendering some scenes with a lot of objects with different textures gets impractical quite fast. Batching object instances that have arbitrary textures becomes a requirement in a lot of cases.

The difference between actual arrays of textures and onion textures is especially important in that context: your shader needs to select a different texture at runtime, depending on some arbitrary input data that's not constant. Onion textures on the face of it make it easy, since doing that works out of the box, you just use the `z` component of the coordinates you're using to sample them to choose which layer. On the other hand, Vulkan doesn't even guarantee you can dynamically index into an array of textures **at all**.

## The tiers of texture array indexing support

There are two features you can enable that relate to support for using arrays of textures in your shaders. Unfortunately the explanations in the spec are somewhat fuzzy to the average reader, and the meaning of dynamic indexing, dynamically uniform and dynamically non-uniform can be confusing, so here's another table that tries to explains the question as a series of tiers:

| Tier | Requires | What it allows |
| --- | --- | --- |
| 0 | Nothing (core Vulkan 1.0) | Your indexes into texture arrays need to be compile-time constants, making them essentially just syntactic sugar with no real utility. |
| 1 | Enabling the core feature `sampledImageArrayDynamicIndexing` | You can index into your texture arrays with whatever you want ( your index can be *dynamic*, that is, known only at runtime) but the index has to be *uniform* accross the draw call: it has to be the same for all shader invocations otherwise it is undefined behavior and you get nasty visual corruptions. |
| 2 | Support for `VK_EXT_descriptor_indexing` and enabling the extension feature `sampledImageArrayNonUniformIndexing` | Indexes can be dynamically *non-uniform*: they can freely diverge between shader invocations. This means you can use anything you want as an index: a `flat int` vertex input attribute, the draw/instanceID, an atomic counter, a random number ...

Tier 0 is essentially irrelevant because almost all Vulkan-enabled devices have dynamic uniform indexing of texture arrays. It's mostly representing older hardware that had no concept of arrays of samplers natively. Stuff like MoltenVK and Swiftshader don't support them, but that's more of a work-in-progress issue than a limitation of the underlying hardware.

Tier 1 is where interesting stuff begins, but it ultimately isn't a great solution: it can help you avoid switching texture descriptors arround by using an index stored in a push constant or an UBO indexed by the baseInstanceIndex, but you are still required to make sure whatever you use as indexes are uniform, and thus the same textures are used across one draw (including multiple instances of that draw if it's instanced), and so you can't batch much with instancing, you can mostly save on swapping descriptors arround. You can, however, cut pretty well into the number of vkCmdDraw, since each entry in those is considered a separate draw when it comes to uniformity in shader wavefronts.

Tier 2 is the real deal, the EXT_descriptor_indexing extension gives devices a "non-uniform dynamic descriptor indexing" feature you can use in combination with unsized descriptors (= giant descriptors whose size is runtime dependent). This enables the use case where you make one giant descriptor set with all your texture resources in an unsized array, and then you can use regular integers to index that array, effectively getting non-opaque texture handles like you would have gotten with `GL_ARB_bindless_texture`, except this time supporting non-uniform access, **something you cannot do at all on AMD and Intel in OpenGL**, since they don't expose any extension for that functionality. This is one of the rare cases where Vulkan has an actual exclusive feature! 

I point you to [this presentation](https://www.youtube.com/watch?v=tXipcoeuNh4) on Youtube for more details on the VK_ext_descriptor_indexing extension and how it works. I will be relying on that tier of support with the Vulkan backend for Chunk Stories, and will now explain the rationale for why.

![](/images/blog/descriptor_indexing_corruption.png)
*What you get if you try to use non-uniform indexes with Tier 1 support. Disgusting.*

## Caveats with using descriptor indexing

First, let's talk about the problems using `VK_EXT_descriptor_indexing` causes:

 * It's not supported everywhere:
	 * won't work on mobile (that I don't care)
	 * won't run on Intel HD graphics before Skylake (meh)
 * *Might* be marginally slower at runtime because of the indirection, in practice I haven't been able to detect any significant performance delta yet.
 * It's cutting edge ! Found an AMD driver crash and a validation layer crash when working with it, both fixed since.
 * ~~**No RenderDoc support**~~ RenderDoc now supports it !

## Why it is still the better solution

I honestly don't mind most of these caveats ( Renderdoc not supporting hurts my feelings more than anything else because I didn't want to do a fallback mode ), because I see the alternatives as being way worse overall. In particular people have asked me a lot why not just use onion textures. Here's why not:

### Onion textures are fundamentally fixed-size.

And all these often-proposed hacks are horrible:

* An array of onion textures, make one per size (128x, 256x etc). Doesn't work for me because it introduces the descriptor indexing problem back to handle multiple sizes (or you must batch per-texture size, which defeats the point), is conceptually ugly and doesn't work for npot/rectangular textures because of how many of these you'll need.
* An array of atlases (or anything involving texture atlases). Been there with the old renderer. Texture atlases suck, they break filtering, they take ages to build and they make you mess with your UV coordinates in turns that makes you write hacks on hacks (the worst is when combined with non-atlased resources, ugh)
* Sparse onion textures are **actually the most viable idea of the bunch**, but still falls short for me because of the npot/rectangular edge cases, those still require specific hacks/complexity that I can just avoid with descriptor indexing.
	* If I restricted myself to only square power-of-two textures, would have used.

### Onion textures have a fixed layer count.

Therefore:

* You must guess in advance not only how many textures you'll be using, but their approximate total area. Get it wrong ? See below.
* You may have to stall the entire engine when you rebuild your giant onion texture, and it's a lot of data to move arround.
* Plenty of potential for nasty bugs where texture coordinates in vertex data have been built assuming a specific onion texture layout and now must get rebuilt/updated. (Been there did that)

### Only (sort of) works for your "normal" 2D textures

* Descriptor indexing works with any texture format
* **Descriptor indexing works with any texture dimensionality** ( 1D, 2D, 3D, Cube and indeed onion textures of each dimension !)
	* You can even alias their bindings in the shader, and share the same descriptor array for all of them
	* You still have to know what dimensionality you are sampling from, ie `vtexture2D(myHandle, coords)`

Put simply, using onion textures or any form of global texture atlas feels like a hack compared to proper descriptor indexing. Descriptor indexing is a truly general solution, it has no limitations or complicated setup process, it's truly a profound departure from the classical binding model, and not just a clever hack. To hammer this point home, here is some other applications of descriptor indexing that wouldn't be possible with onion textures:

 * Animated textures: Just overwrite descriptor N in the giant descriptor set to point to another frame of the animated texture ! The rest of the engine doesn't have to know a thing, the descriptor set works just an array of pointers into the actual textures.
 * Adaptative lod/loading thumbnails: Like when animating textures, you can also progressively load a high-quality texture and update the descriptor set to point to the full-res asset when available. You can do that from a worker thread even thanks to the softening of descriptor set updating rules made possible by other features in the extension.
 * Having integer texture handles makes writing materials sytems dead easy, just put all your materials in a big UBO/SSBO and look up the textures you need with a materialID
 * Using a lot of cubemaps for per-object reflection probes or other smart techniques I can now start to think about ... 3D voxel textures anyone ?

## Conclusion

This note might make it sound like I hate onion textures. That's not really my point, it's an older and technically inferior solution to the old problem of batching multiple objects with arbitrary different textures. There was a time where texture atlases was all we could do, and at that time I would have been a huge advocate to using those for everything. The point is that it's the best, most flexible, one-size-fits-all solution to the texture binding problem. 

My intent with Chunk Stories tech is to give it the best possible (flexibility / simplicity) ratio, even if I have to pay a small price at runtime. You can see that in the choice of using the JVM instead of a native language, the choice of data structures or the few amount of dependencies and frameworks used. Chunk Stories has different design goals than IdTech 6, and this is something that's sadly sometimes hard to get across.

![](/images/blog/tileset.png)
*This was never fun to deal with. At least it's easy to visualize !*

In the context of this project, it makes sense not to support mediocre GPUs that barely got Vulkan support at all and to deal with the freshness of the descriptor indexing extension because of how damn simple it makes handling texture binds in the engine. Bindless is one of these features that sounded like the future in OpenGL, and I'm sort of disappointed Vulkan couldn't make that a core feature (and shocked it took them so long to properly re-introduce the same functionality, `VK_EXT_descriptor_indexing` is from 2018 !). All in all, I'm pretty happy about the situation though, I knew from the start I was going to have to cut off a lot of old but still quite good hardware with Vulkan anyway (pre-gcn AMD cards still have some life into them, and so does Fermi cards from Nvidia), and the additional limitations don't really change that picture. Addressing that hardware left behind is something that would and will be more appropriate in a seperate legacy OpenGL backend anyway !
