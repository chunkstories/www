
## Tour of the game engine

The goal of this part is to roughly go over every big conceptual "part" of the engine and have a small write-up about what it's purpose is, how it achieves it and how it fits into the rest. Of course you don't have to read it sequentially, feel free to click away.

* `null`: What's yet to be.
   * [Short-term roadmap](roadmap)
   * [Planned, envisionned and otherwise rejected features](planned_features) (mid/long-term roadmap)
* `common`: The basics of the engine
   * [World data structures](world_data): How the data is stored and kept in memory
   * [Custom file formats & DSLs](content_definitions)
   * Voxels: How custom classes work and fitting them in the engine
   * Entity traits: How we deal with complex behaviors inheritance can't model
   * Items & inventories: One of the simplers systems
   * Networking: Key ideas and current limitations
   * Physics & animation
   * World generation
* `client` : The front-end of the game most players get to see
   * GUI: Quick overview & some gotchas
   * Sound: How to play and manage your sounds
   * Graphics
       * The API: Rendergraphs to Representations
       * The Vulkan backend
           * Ressource management
           * Virtual texturing
       * The OpenGL / GLES backend
       * Interoperating with GLSL
* `server`: It's actually not that complicated! 
* Minecraft world `converter`