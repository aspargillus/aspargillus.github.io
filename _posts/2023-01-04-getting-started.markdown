---
layout: post
title:  "Getting Started..."
date:   2023-01-04 21:47:33 +0100
tags: [kotlin, asteroids, design]
---

This is all Rons fault.

I do not expect anyone to actually read this, but, just in case someone does, some explanations may be in order.

A Bit Of History
----------------

This all started with a series of articles that Ron Jeffries started back in August on his blog, <https://ronjeffries.com/>.
(I am a not-quite-regular reader.)
In this series, Ron writes about learning the [Kotlin](https://kotlinlang.org/) programming language.
At the time of this writing, he is almost 200 articles in.
And while writing about learning Kotlin, Ron demonstrates his way of programming.
I recommend his writing to anyone interested in coding, software development and software design.
I continue to find it fascinating how Ron starts out with such a vague idea of what his code will look like and to let himself be guided by, using his words, what the code wants.

At some point, Ron started to write a clone of the game [Asteroids](https://en.wikipedia.org/wiki/Asteroids_(video_game)) (see the [kotlin-asteroids category](https://ronjeffries.com/categories/kotlin-asteroids/) on his blog).
As mentioned, Ron tends to start out with a rather fuzzy idea for the design of the program.
In this case, it took him down a very interesting path, where the coordinating class, which he named `Game`, actually knows next to nothing about how the game works.

All the game rules emerge from the interactions between a bunch of objects, some of which represent actual things in the game, such as the players spaceship and the eponymous asteroids.
But others have special tasks, like re-spawning a spaceship if there isn't any to be found and the player still has some left.
All the `Game` really does is execute a simple cycle on all the game objects that are, as Ron calls it, "in the mix":

1. Update each object separately given the amount of time passed since the last cycle. 
2. Notify each object now in the mix that object-object interactions are about to start. 
3. For each unique pair of objects in the mix, execute the interaction for each of them. 
4. Notify each object now in the mix that object-object interactions have now ended. 
5. Draw each object that is in the mix now.

All steps except 2 and 5 may result in new objects added or existing ones removed from the mix.
For details, see [Rons repository on GitHub](https://github.com/RonJeffries/openrndr-ship-1).

At some point, all the actual _things_ of the game (basically everything that you can see on the screen) where even instances of the same class, just with different settings.
Notably, these objects came configured with a view to make them look like, well, asteroids and spaceships on screen.
So, one class with a pluggable behaviour to achieve the different look.

All that got me thinking...

I Need Space In My Head
-----------------------

So I was, and still am, thinking a lot more about this program design than I probably should.

These thinkinigs include

* Since the game rules emerge entirely from interactions between the game objects, visible things as well as special objects, should these interactions be first-class citizens in the program design and not only methods?
  <br>(They actually are not really methods but anonymous functions stored in fields. Maybe we'll get to that later...)
* Could one roll the differentiated classes all back to one class (or maybe two or three classes) and create the different characteristics entirely by pluggable behaviour?
* If enough (whatever that means) such pluggable behaviours can be discovered, could one make a nice DSL using those as building blocks?
  <br> Maybe that would be expressive enough to make different Games such as [Spacewar!](https://en.wikipedia.org/wiki/Spacewar!https://en.wikipedia.org/wiki/Spacewar!) or [Pong](https://en.wikipedia.org/wiki/Pong).

In an attempt to get back that precious brain space, I figured:
considering the amount of time I spend thinking about this, I might as well try to _do_ some of that.
I will be horribly mistaken, of course, and this exercise will eat much more of my time than it should.
But I intend to have some fun on the way.

With this, it's time for a

Caveat
------

This is a fun project I do for myself.
The software and designs that may emerge from it will likely not be "good" (whatever that means).
And they are not supposed to be.
I want to explore and maybe learn something on the way.
The chances of me discovering the next big thing(TM) in software design are ... slim.


Can We Finally Get On With It, Please?
--------------------------------------

OK, OK. I made my own [fork on GitHub](https://github.com/aspargillus/kotlin-asteroids) to toy around with this. Now, where should I start? Let's explore interactions first.

Maybe like this:
Let's say there is a class called `Interactions` that will at some point know all relevant interactions between the different kinds of game objects.
Then I want to start a new interaction cycle by calling a method `beginInteractions`.
That should result in the `beforeInteractions` hook[^1] of all the known game objects (i.e. `ISpaceObject`s) to be called.
So I wrote this test to drive that out:[^2]

```kotlin
@Test
fun `beginInteractions should notify all objects`() {
    val object1 = object : ISpaceObject {
        var beforeInteractionsCalled: Boolean = false

        override fun update(deltaTime: Double, trans: Transaction) {
            TODO("Do not call this!")
        }

        override val subscriptions: Subscriptions = Subscriptions(
            beforeInteractions = { beforeInteractionsCalled = true }
        )

        override fun callOther(other: InteractingSpaceObject, trans: Transaction) {
            TODO("Not yet implemented")
        }
    }
    val knownObjects = SpaceObjectCollection().also { it.addAll(listOf(object1)) }

    val interactions = Interactions(knownObjects)
    interactions.beginInteractions()

    assertThat(object1.beforeInteractionsCalled).isTrue
}
```

This, will of course, requires some production code.
After discovering that my Kotlin skills are rustier than I'd like, I arrive at this little class:

```kotlin
class Interactions(private val knownObjects: SpaceObjectCollection) {
    fun beginInteractions() {
        knownObjects.forEach { it.subscriptions.beforeInteractions() }
    }
}
```

That makes the test green and I commit: Towards first-class interactions: beginInteractions calls beforeInteractions on each object.
I get a warning because of the TODOs and [detekt](https://detekt.dev/) does not like a wildcard import.
But I do, need to configure that away...

I think the test would be nicer if there were at least two objects in the `SpaceObjectCollection`.
That would add another `object` to the test which does not exactly improve readability.
To remove the noise, I make a new class for my test doubles that implements `ISpaceObject`:

```kotlin
class InteractionTestObject : ISpaceObject {
    var beforeInteractionsCalled: Boolean = false

    override fun update(deltaTime: Double, trans: Transaction) {
        TODO("Do not call this!")
    }

    override val subscriptions: Subscriptions = Subscriptions(
        beforeInteractions = { beforeInteractionsCalled = true }
    )

    override fun callOther(other: InteractingSpaceObject, trans: Transaction) {
        TODO("Not yet implemented")
    }
}
```

And now the test goes like this:

```kotlin
@Test
fun `beginInteractions should notify all objects`() {
    val object1 = InteractionTestObject()
    val object2 = InteractionTestObject()
    val knownObjects = SpaceObjectCollection().also { it.addAll(listOf(object1, object2)) }

    val interactions = Interactions(knownObjects)
    interactions.beginInteractions()

    assertThat(object1.beforeInteractionsCalled).isTrue
    assertThat(object2.beforeInteractionsCalled).isTrue
}
```

Much better. Time to commit: Cleanup test to improve readability.

Object, Meet Object.
--------------------

Next, for each pair of `ISpaceObject` interactions need to be executed.
Rons game uses a [double dispatch](https://en.wikipedia.org/wiki/Double_dispatch) via `callOther` of `ISpaceObject` and the various `interactWithX` lambdas of `Subscriptions`.
I want all of that to go away.

Instead, I want to be able to write something like this: `interactions.executeInteraction(object1, object2)`, where `executeInteraction` looks up what to do based on the kinds of objects involved.
For now, I'll just use the classes of the objects, I think.
Let's make a test!

```kotlin
@Test
fun `executing interactions can call methods on objects`() {
    val object1 = InteractionTestObject()
    val object2 = InteractionTestObject()
    val knownObjects = SpaceObjectCollection().also { it.addAll(listOf(object1, object2)) }

    val interactions = Interactions(knownObjects)
    interactions.executeInteraction(object1, object2)

    assertThat(object1.interactionExecuted).isTrue
    assertThat(object2.interactionExecuted).isTrue
}
```

Now `executeInteraction` and `interactionExecuted` do not exist. The latter must be a new field in the `InteractionTestObject` and the former I just let IDEA make for me.
Running the test informs me that `executeInteraction` is "Not yet implemented" which is what IDEA generated for me.

Looking at this, I realise that the step size is way too large.
I could try to make that work.
I'm sure I'd manage. Eventually.
But it might take a while and involve all sorts of debugging.
And that isn't fun.

Go Faster With Smaller Steps
----------------------------

Let's change track for a bit and do the lookup thingy first. Another test then...[^3]

First, I want a type alias:

```kotlin
typealias Interaction = (object1: ISpaceObject, object2: ISpaceObject, transaction: Transaction) -> Unit
```

That will make all the following much more readable...

```kotlin
@Test
fun `interaction lookup yields null by default`() {
    val object1 = InteractionTestObject()
    val object2 = InteractionTestObject()
    val knownObjects = SpaceObjectCollection().also { it.addAll(listOf(object1, object2)) }

    val interactions = Interactions(knownObjects)

    assertThat(interactions.findInteraction(object1, object2)).isNull()
}
```

So, when the game goes through every possible combination of objects in the mix and there is no interaction defined for this combination of types, `findInteraction` returns nothing, i.e. `null`.
That will require me to do a null check later, but Kotlin will remind me.
It's very good at that.

That is easily made green, just add this to the class `Interactions`:

```kotlin
fun findInteraction(object1: ISpaceObject,object2: ISpaceObject): (Interaction)? {
    return null
}
```

Now the more interesting case. I need some way to define what an interaction between two objects of a particular combination of types actually does.
Let's try this:

Now, let's have a test for registering and lookup.

```kotlin
@Test
fun `interaction lookup yields previously registered lambda`() {
    val object1 = InteractionTestObject()
    val object2 = InteractionTestObject()
    val knownObjects = SpaceObjectCollection().also { it.addAll(listOf(object1, object2)) }

    val interactions = Interactions(knownObjects)
    val interaction = { _: ISpaceObject, _: ISpaceObject, _: Transaction -> }
    interactions.register(InteractionTestObject::class, InteractionTestObject::class, interaction)

    assertThat(interactions.findInteraction(object1, object2)).isSameAs(interaction)
}
```

Looking good enough to me.

Now, I modify `Interactions` to keep the registered lambdas in a mutable map.
Then registration stores the provided lambda in that map under a key which is just a `Pair` of the classes.
`Pair` is a convenience class from the Kotlin standard library. Very Handy.
And in Kotlin one can do reflection similarly to Java.
So the class of any object can be accessed at run-time and is just another object (of class `KClass`).

The `findInteraction` method then just makes a lookup in the map with such a key constructed from the given `KClass` objects.

```kotlin
class Interactions(private val knownObjects: SpaceObjectCollection) {
    private val registeredInteractions =
        mutableMapOf<Pair<KClass<out ISpaceObject>, KClass<out ISpaceObject>>, Interaction>()

    fun beginInteractions() {
        knownObjects.forEach { it.subscriptions.beforeInteractions() }
    }

    fun findInteraction(object1: ISpaceObject,object2: ISpaceObject): (Interaction)? {
        return registeredInteractions[Pair(object1::class, object2::class)]
    }

    fun register(class1: KClass<out ISpaceObject>, class2: KClass<out ISpaceObject>, interaction: Interaction) {
        registeredInteractions[Pair(class1, class2)] = interaction
    }
}
```

That works just fine. I see some potential for refactoring: construction of the key `Pair` is duplicated, but I'll let that be as-is for now.
I could have committed one test earlier, but now I really should: Register and look up lambdas for interacting objects of a combination of classes. 

OK, now that I have a few stepping stones, the step that was too big should work just fine.

I settled on a slightly different test though. Instead of expanding my test dummy class, I will use the `Transaction`:
the interaction lambda removes both objects via `Transaction::remove` and adds a third one via `Traansaction::add`.
I can assert that easily in the test like so:

```kotlin
@Test
fun `executing interactions can use transactions`() {
    val object1 = InteractionTestObject()
    val object2 = InteractionTestObject()
    val object3 = InteractionTestObject()
    val knownObjects = SpaceObjectCollection().also { it.addAll(listOf(object1, object2)) }

    val interactions = Interactions(knownObjects)
    interactions.register(InteractionTestObject::class, InteractionTestObject::class) { o1, o2, transaction ->
        transaction.remove(o1)
        transaction.remove(o2)
        transaction.add(object3)
    }
    val transaction = Transaction()
    interactions.executeInteraction(object1, object2, transaction)

    assertThat(transaction.adds).containsExactly(object3)
    assertThat(transaction.removes).containsExactlyInAnyOrder(object1, object2)
}
```

With the registration and lookup already in place, this is easily made green by adding the missing `executeInteraction` method.

```kotlin
fun executeInteraction(object1: ISpaceObject, object2: ISpaceObject, transaction: Transaction) {
    val interaction = findInteraction(object1, object2)
    interaction?.let { it(object1, object2, transaction) }
}
```

That totally works, test is green. Cool. Commit that: Registered interaction lambdas can be executed and use transactions.

But ... The lookup of interaction lambdas is still very weak: the test uses a pair of the same class, so it does not matter which one comes as the first or second parameter. Since we will be faced with arbitrary orderings in the interacting objects, that needs to change. Let's expand the test:

```kotlin
@Test
fun `interaction lookup yields previously registered lambda`() {
    val object1 = InteractionTestObject()
    val object2 = object : InteractionTestObject() {}
    val knownObjects = SpaceObjectCollection().also { it.addAll(listOf(object1, object2)) }

    val interactions = Interactions(knownObjects)
    val interaction = { _: ISpaceObject, _: ISpaceObject, _: Transaction -> }
    interactions.register(object1::class, object2::class, interaction)

    assertThat(interactions.findInteraction(object1, object2)).isSameAs(interaction)
    assertThat(interactions.findInteraction(object2, object1)).isSameAs(interaction)
}
```

Now `object1` and `object2` are of different types and |I added a second assert that does the lookup with reversed order of objects. That test now fails, of course: the second lookup yields null.

No there are multiple ways to fix that. One is to just look up both. But then one could still register two interaction lambdas, one for combination of classes `A` and `B` and one for the combination `B` and `A`. So what I'll do instead, is to define an ordering on the `KClass` objects that go into the `Pair` which is the key to the map. For that, let's first refactor the key pair creation. I have a little problem here, since my test is now red. So what I'll do is a `git stash` to temporarily roll back the changes to the test, then refactor, then re-apply the changes with `git stash pop`.

This gives me a new method on class `Interactions`:
```kotlin
private fun makeKey(class1: KClass<out ISpaceObject>, class2: KClass<out ISpaceObject>) = Pair(class1, class2)
```

Commit: Extract creation of lookup keys for interaction lambdas. Then pop the stashed changes.

After some experiments I settle on comparing the `hashCode` of each `KClass` for ordering.
That does not seem ideal since I have no Idea whether those can collide.
I wanted to use the `qualifiedName`, but that may be null. Too bad.
Now `makeKey` is this and the test green.

```kotlin
private fun makeKey(
    class1: KClass<out ISpaceObject>,
    class2: KClass<out ISpaceObject>
): Pair<KClass<out ISpaceObject>, KClass<out ISpaceObject>> {
    return if (class1.hashCode() < class2.hashCode())
        Pair(class1, class2)
    else
        Pair(class2, class1)
}
```

One last commit for now: Registration and lookup of interaction lambdas do not care about order of parameters.

Wrapping Up For Now
-------------------

This may read like I worked on it continuously (I hope), but was in fact spread out over three days.
Time to warp up for now so I can get out what I have so far.

I am fairly satisfied with the direction but have no idea where this is going to end up.
I suspect by now, that the game logic will be mostly in a ton of different lambdas...

I also have an inkling that there are two unfamentallu different kinds of game object interactions going on:

1. Immediate interactions where the whole result is clear as soon as I know which types of objects are involved and their states, such as a collision of an asteroid and the ship.
2. And then there are stateful interactions that initialise at the beginning of the interaction cycle, update with each interaction that they are interested in and draw to a conclusion only at the end of the cycle. E.g. `WaveMaker` has this kind of interaction with asteroids.

Maybe this will show more obviously in the code at some point. Maybe the state of the interactions elsewhere. I don't kn ow yet.

Anyway, enough for now. Next time I will likely complete the interaction cycle and notify each `ISpaceObject` of that.

so long...

PS: Here is the [repository on GitHub](https://github.com/aspargillus/kotlin-asteroids) for the whole endeavour.


[^1]: As mentioned before, what I call *hook* here, is not really a method of a game object.
Instead, the interface `ISpaceObject` demands a field of type `Subscriptions` which in turn has a number of fields that hold lambdas which have default values that are empty functions.
That is a trick by [GeePaw Hill](https://www.geepawhill.org/) to avoid implementation inheritance, i.e. inheritance from a base class with concrete methods that may be partially overridden.
Because implementation inheritance is generally not a good thing. Because it is hard to understand.
In this case, I quite honestly don't see how this scheme, while neat, is any simpler than straight implementation inheritance overriding those empty methods if needed. 

[^2]: No, I don't write this out in one go just like that.
I usually start in the middle bit that interacts with the production code that doesn't exist yet.
In this case, the creation of the new `Interactions` object and the call to `beginInteractions`.
Then I think what the test should actually test here (the `asssert`).
Now I can give the test a proper name that tells us what it does.
Then I fill out everything that is required to get to the middle part.

[^3]: I just leave the failing test lying around. That may not be the best way to go, but I never promised to do everything just right, have I?