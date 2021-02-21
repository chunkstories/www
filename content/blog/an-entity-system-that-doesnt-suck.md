---
title: "An entity system that doesn't suck"
date: 2018-05-26
---

One of the more complicated things that I'm slowly getting better at with Chunk Stories, is how to arrange code for complicated systems into a pattern that is both concise, clear to understand and easy to maintain. By far the most complicated type of objects in Chunk Stories are the <i>Entities</i>. Entities are any complex object that exists in the world and isn't part of the voxel grid/terrain. Entities have to be serialized, both on disk and over the network, they have to be rendered using very specific rules for each of them, but also rendered fast, they can represent anything from an item on the ground, a mob roaming the world, a vehicle you can drive or a player model with many abilities (mining, using items etc) and properties (rotation, food level etc).

<!--more-->
The Entity system in Chunk Stories is undergoing it's second big refactor. The first one actually took place in 2016 in the earlier days of the project, where I switched from a completely inheritance-based approach (with huge monolithic classes that handled their own serialization) to an hybrid ECS-style system where entities no longer saved anything and their important state would put into Components that could handle their own serialization, with added flexibility.

There was already a lot of gains to that approach, namely entities wouldn't be corrupted entirely in the save file whenever entity code was touched. Only the changed component would be reset (ie going from a double to a int for their health component would only reset *that* component, and removing the health component from an entity just discards it the next time it loads). This also enabled components to decide how they should inform remote players about the state they hold (other players don't need to know about what is in *your* inventory, this is a cheat-preventing concern as much as a network performance concern)

## The troubles

But today I'm rewriting that system once more. There are two issues that I'm looking into fixing, and both are actually holdouts that the first refactor couldn't quite iron out. The first one is that I have a huge mess of interfaces that basically say "Entity X has component Y", and offer nothing more. The components weren't stored in a unified query-able data structure in the entities, but rather in Java fields and exposed through the implementing of an Interface. This is a very verbose way to do it, it makes stuff a pain to maintain and negates much of the benefits of shifting complexity to external components. Furthermore, it meant I had an Entity interface **and** an EntityBase class that implemented it and that's the one modders were meant to use, in contrast with the other systems of the engine where they just extend a base class (`Voxel.java`, `Item.java` etc) that is named after the system.

![](/images/blog/interfaces-mess2.png)
*This is peak Java*

The second issue is that I still have a lot of code in the Entity* classes themselves. That's not a problem in of itself because I do believe that inheritance isn't all that bad, and that some entities should really extend others ( EntityHumanoid -> EntityPlayer, for instance ), but some things like what renderer to use or collision logic are too generic to be forced into a hierarchical model. Below is an example of what I mean: the class hierarchy makes semantic sense but we have a problem: where do we put the collision code ? We can't put it in EntityLiving or EntityBase because they might have child classes that don't need it, and we don't want to have the code copy/pasted everywhere we need it. We need a clean solution.

![](http://chunkstories.xyz/blog/wp-content/uploads/2018/04/traits-problem.png)
*Both cars and Bipeds need the collision code, but bees do not. In what class do I put the collision code?*

But isn't this problem already solved ? The collision stuff should live in a component! The thing is, components are really about storing data and propagating that data across time and space. An entity's collision box and logic isn't something that changes at all, it has no persistent mutable state to keep track of. It doesn't need all the boilerplate pull/push code, and it would be waste of space because the game would still bother with loading/saving blocks of empty data with the name of that component for each entity. What we need is something like a component but for logic only. I've decided to call that a `Trait`.

Traits in this refactor are essentially lightweight components, they are nothing but an object associated with an entity. They contain arbitrary code about any trait of an entity that's complex and generic enough to warrant being extracted from the base EntitySomething class, but don't contain any state that should go into a component. They pretty much solve the collision problem: I just have a CollisionTrait that exposes the collision box for an entity and contains the logic for handling world and entity-entity collision. Any entity that needs a collision model just has to expose that trait and configure it for itself.

## Fitting it together
As mentioned before, the current way of having interfaces to declare the presence of a component is a pain, it's a lot of boilerplate code, a lot of useless interfaces with confusing names ( is EntityFlyMode the component that remembers whether or not we're in fly mode or the interface that merely says that we have that component ? ), and the user code reflect that, with a lot of `instanceof `calls, null-checking and general ugliness. With this refactor every single one of those interfaces got ditched, and Entity.java and EntityBase.java got rewrote in a single Entity class that looks a little like that:

```java
//pseudocode for the new entity class
class Entity {
  +uuid // a long unique id
  +definition // contains static properties defined in a text file
 
  +location //a quick reference to the location component
 
  tick() //called every tick, simple logic and calling methods in traits can go here
 
  traits {
    +all()
    +has(Trait)
    +get(Trait)
    +with(Trait)
 
    -registerTrait()
  }
 
  components {
    +all()
    +has(Component)
    +get(Component)
    +with(Component)
 
    -registerComponent()
  }
 
  Set<Subscribers> // who gets to receive updates about that entity
}
```
As you can see it's pretty bare now! The actual class contains a bit more stuff, mostly getters and helpers, but it's really generic now. Speaking of generics, that's a cool feature I'm leveraging with traits & components storage: when calling get(ComponentX.class), you get either null or an actual ComponentX object. No need for instanceof/casting, you get all or nothing. The with method goes further, allowing you to supply a lambda expression to run on the component/trait if it exists in the entity. So something like

```java
if(entity instanceof EntityWithInventory)
    ((EntityWithInventory)entity).getInventory().doSomethingWithIt();
```
becomes
```java
entity.components.with(EntityInventory.class, i -> i.doSomethingWithIt());
```

which is far more readable and actually makes you want to do stuff with it, rather than yawn at the verboseness of it all.
Some of the friendly folks on our <a href="https://discord.gg/wudd4pe">Discord</a> server have not been able to shut up about Kotlin, and having tried it, it's kind of porn for syntactic sugar enthusiasts. Stuff like the Elvis operator is indeed pure awesome! But unless I want to switch to Kotlin for good ( something I'm not sure about and would definitely warrant a blog post here ), I'll just try to make the best out of what Java gives me.

## Other considerations

While doing this refactor (actually not finished while I'm writing this), some other design things popped up that I figured I should answer to myself at least: [http://ourmachinery.com/post/should-entities-support-multiple-instances-of-the-same-component/](Should entities support multiple instances of the same component?) is a great article I suggest you run out and read. It's written by people who have immensely more knowledge and experience than me on literally all areas of this field.

The question is an interesting one, and I too chose the single instance of a component approach too. With a caveat:
Components ( and traits ) can be overridden by subclasses of themselves. As I've not gone full-data-oriented and kept a loosely hierarchical model for my entity classes, sometimes you might want to override a component the parent class declares: let's say you have bipeds that can walk and you want some of them to be able to jump.

In this code style, you'd have a `TraitMovementWalk` that handles the walking movement that is registered at constructor time by the `EntityBiped`. Your `EntityKangaroo` calls the super() constructor and ends up with a `TraitMovementWalk` trait. You have a `TraitMovementJump` trait that extends `TraitMovementWalk` and adds a method for jumping, and you want that trait to take the place of the `TraitMovementWalk` trait in `EntityKangaroo`.

With this system you can do that, just by registering the fancier version of the trait after calling the superconstructor, thus overwriting the other trait. The system is smart enough to also give you that `TraitMovementJump` even when you make a request for a `TraitMovementWalk` , because it `knows` it fills in the same role thanks to inheritance.

This comes with a handful of corner-cases, though. Namely, I have abstract EntityComponentGeneric[Boolean/Int/Double/...] classes to have less duplicated code in components that store trivial information. Without careful control, the system would believe any two components that comes extends the same generic one ( ie a component about health and one about food level ) are referring to the same thing, when all they share is the computer representation of that information.

Solving this can be done in two ways: either I don't make those silly generic helper classes and live with boilerplate code duplication :( or I tag them in some way so the register() method knows they don't represent anything and consider their children to be unique unrelated things (like it does with the base EntityComponent/Trait class already !)

Another thing that went through my mind is that Components are in some way just an extension of Traits that can be serialized. I could make it so they are placed in the same data structure in the entity, with no hard dichotomy between them, no `entity.traits` vs `entity.components`. But there is a rather large conceptual difference here ( state and data, versus pure logic, even though components often have logic too ), and it'd make the entity serialization/propagation logic a good deal uglier, if not bug-prone.

## In closing

Entities are HARD. Actually, good code design is hard (*Especially when using Java !*, I hear in the back), and this probably isn't the last time I revisit this aspect of the engine. But that's okay to me. Unlike other projects it "competes" with, Chunk Stories is not a commercial one, it's not even about making a game as much as it is about learning how real-world, non-trivial, "general-purpose"¹ game engines work, in all their glorious complexity, design challenges and technological marvel.

I should probably finish this refactor and make it compile again before the summer exams, though.

¹ Yeah a voxel game engine isn't really general purpose. A better way to phrase it would be *not tailor-made for one specific game with hardcoded stuff all other the place*