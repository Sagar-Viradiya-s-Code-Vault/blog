---
title: "Compose Animation, Under The Hood - Part I"
summary: A deep dive on internals of Compose Animation
date: 2024-10-07
weight: 1
tags: ["Compose", "Animation"]
cover:
  image: images/header.jpeg 
  caption: "[Generated With AI](https://www.bing.com/images/create/generate-header-image-for-blog-explaining-internal/1-66ffb250fccc43f588e2e34db8724eb4?id=JJRlbQKelN%2fMt%2f3JS0rXFw%3d%3d&view=detailv2&idpp=genimg&thId=OIG1.NpAOOqqUxD4ksSM_1iMa&skey=nRgIOQdbVScL_nCqQ1_pUwJdKJObIw0F24OkIDRyqH0&FORM=GCRIDP&mode=overlay)"
  hiddenInList: false
---

## Context

I have been dabbling with Compose Animation API for quite some time. Moving pixels on the screen how you want is such a joy! During this journey, I explored implementing some cool animations, which led to my curiosity about “what’s going on behind the scenes?”  It was a perfect opportunity for me to dig inside the internals.

After learning how everything is wired together, I was fascinated. I knew that sharing this with the community was a must, so I gave a talk on this at Droidcon Lisbon 2022. I wanted to share this with a wider audience at Droidcon London this year, too, but unfortunately, my talk didn’t make it through.

I decided to write about this and this is the first part of this series. So sit back, grab your favorite drink, and enjoy!

## Why?

First, why is it essential to know the internals of Compose Animation? I also asked the same question when I decided to investigate. There were a couple of convincing reasons.

1. Efficient Animation - Animation can lead to performance implications if not implemented efficiently. Things like unnecessary recompositions or frame drops can easily make your UI ugly. Knowledge of surface level API that you consume mostly will help you to at least think about performance implications while implementing complex animation.
2. Choosing the right tool for the Job - Knowing which is a suitable API to implement animation in hand will help you to use the right tool for the right job.
3. Debugging - Something you cannot run away from 😅 No matter how much attention to detail is given while implementing something, a day will come when you need to investigate what’s wrong. You can fly through debugging something if you know that thing very well. Compose Animation is no different. 
4. Thin wrapper over API - Sometimes you need a thin wrapper over API to consume it how you want. This also unlocks loose coupling with API so that you can swap it with other APIs in the future.
5. Master of one, Jack of none 🤪 - Sometimes you are just curious and want to master one thing.

Alright, I hope the above reasons were enough to convince you and you are curious enough to read further 🙂 Let’s deep dive!

Following is the order in which we will explore animation API internals.

1. Compose animation in Nutshell
2. Compose animation Hierarchy
3. Low level Animation API
4. Animatable API (Coroutine based API)
5. Transition API
6. `animate*AsState` API

For scope of this part we will focus on first three points.

## Compose Animation in Nutshell

Animating is nothing but changing values from source to target rapidly for a given duration of animation. In the compose world this would be changing state values that your composable is reading which will cause a re-layout and redraw of your composable (I didn’t mention recomposition as this is something you don’t want. More on this later). Let’s see one example.

Let’s say we want to animate from 0 to 1 for 500 ms. In compose this would be a sequence of state changes at a fixed duration for 500 ms.

{{< figure src="images/animation_in_nutshell.svg" >}}

Any guesses on how many times the state changes during 500 ms? 🤔

Well, it depends on the device's refresh rate. For a 60 Hz device, each frame gets 16.67 ms. Doing simple math would give us a number of state changes. You just have to divide 500 by 16.67. Which is ~30 times state updates. This increases with the device refresh rate. So for a 90 Hz panel, it would be ~46, and for a 120 Hz panel, it would be ~61.

These numbers also go up as the duration of animation. That’s why it is essential to also know how you are consuming value within composable. You can leverage graphics layer API to avoid executing all three phases of compose during recomposition due to value change. In most cases, you only need re-layout and redraw. In some cases, you can even avoid re-layout (Animating background color). More on graphics layer API in part - II.

Now we know the high-level concept of animation in Compose let’s have a quick look at the API hierarchy.

## API Hierarchy

{{< figure src="images/animation_api_hierarchy.svg" >}}

At a low level, we have Animation API which is the core engine powering animation. Since this is a low-level API you won’t be dealing with it mostly unless you are writing something from the ground up. 

On top of Animation, we have two APIs, Animatable and Transition, which leverage the power of coroutines to unlock animating many things either parallelly or sequentially.

At a very high level, we have animate*AsState which is a convenient composable wrapper to initiate animation on state change.

Please note, this is not a complete hierarchy as we have more high-level APIs such as AnimateVisibility that depend on Transition. I believe understanding APIs in the diagram above unlocks understanding other high-level API so for the scope of this blog I am not covering them. I hope you would be curious at the end and do your homework 😉

Let’s explore the above hierarchy in more detail.

## Animation API

This is the core engine powering Compose animation. API has nothing to do with compose though as this is completely decoupled from compose and essentially you can consume this and come up with your own Animation system. Compose Animation APIs we will see, are built on top of this API.

Animation API is stateless. That means it doesn’t know anything about the current state of animation. It is not responsible for managing things like changing the target if the user interrupts the animation or tracking the current progress of the animation. These are the responsibilities of high-level APIs.

This API is a function of time. All it knows is given the playtime of animation what would be the value of animation at a given playtime. A perfect analogy for this would be scrabbling through video playback. You can jump to a specific time in the video and it will give you a frame at that timestamp.

Let’s see one example to understand this better. Going back to the same example we saw earlier. We are animating a value from 0 to 1 for 500 ms. If you feed 250 ms to Animation API it will give you the value of animation at that playtime.

{{< figure src="images/animation_playtime_blackbox.svg" >}}

So Animation API is essentially a function of time. Given a playtime what would be value at that playtime?

{{< figure src="images/function_of_playtime.png" >}}

Here is the abstraction.

```kotlin
interface Animation<T, V : AnimationVector> {
    .
    .
    .
    // Returns the value of the animation at the given play time
    fun getValueFromNanos(playTimeNanos: Long): T
    .
    .
    .
}
```

Looking at the abstraction we can see it is a type agnostic. It can animate any value you feed in. That’s why there is `AnimationVector`. Which is the type that animation API deals with. It cannot operate on the type of values you provide. For example, If you are animating Color it will convert Color into something called `AnimationVector3D(v1: Float, v2: Float, v3: Float)` as Color has three dimensions (RGB). Animating Color will then result in animating RGB values.

Notice returned value is of your original type. It converts back calculated values from AnimationVector to the type you feed in. There for, you need to also provide `TwoWayConverter` if you are animating your custom type.

Looking at the abstraction it is quit straightforward.

```kotlin
/**
 * [TwoWayConverter] class contains the definition on how to convert from an arbitrary type [T] to a
 * [AnimationVector], and convert the [AnimationVector] back to the type [T]. This allows animations
 * to run on any type of objects, e.g. position, rectangle, color, etc.
 */
public interface TwoWayConverter<T, V : AnimationVector> {
    /**
     * Defines how a type [T] should be converted to a Vector type (i.e. [AnimationVector1D],
     * [AnimationVector2D], [AnimationVector3D] or [AnimationVector4D], depends on the dimensions of
     * type T).
     */
    public val convertToVector: (T) -> V
    /**
     * Defines how to convert a Vector type (i.e. [AnimationVector1D], [AnimationVector2D],
     * [AnimationVector3D] or [AnimationVector4D], depends on the dimensions of type T) back to type
     * [T].
     */
    public val convertFromVector: (V) -> T
}
```

Coming back to Animation API, There are two implementations of this. `TargetBasedAnimation` and `DecayAnimation`.

### TargetBasedAnimation

`TargetBasedAnimation` animates values to a specific target. If the target is known, in the above example it was 1. As playtime approaches the duration of animation so does the value approaching the target.

### DecayAnimation

`DecayAnimation` animates values where the target is unknown. For example, based on initial fling velocity if we want to move something we don’t know where it will end up once velocity becomes zero. Animation here decay over a time.

Let’s zoom in further. How does Animation API calculate the value at a given playtime? 🤔

{{< figure src="images/function_of_function_of_playtime.png" >}}

Internally it delegates to AnimationSpec. So real hero doing heavy lifting is not Animation but AnimationSpec. Here is the abstraction.

```kotlin
interface VectorizedAnimationSpec<V : AnimationVector> {
    .
    .
    .
    /**
    * Calculates the value of the animation at given playtime, 
    * with the provided start/end values, and start velocity.
    */
    fun getValueFromNanos(
        playTimeNanos: Long,
        initialValue: V,
        targetValue: V,
        initialVelocity: V
    ): V
    .
    .
    .
}

```

Notice it accepts vector values and spits out a vector value. This is exactly what I mentioned earlier, animation internally deals with values of `AnimationVector` type.

I kinda lied that Animation API powers Compose Animation. Ultimately complex math is in AnimationSpec. Let's see one of the implementations of AnimationSpec. Brace yourself for some scary Math! 👻

For the scope of this blog, I am going to cover `FloatTweenSpec`. This is one of the specs you get out of the box. Before we jump into implementation details, let's first understand the key concept of `FloatTweenSpec`

1. Duration: This specifies how long the animation will take to complete, typically in milliseconds (ms).
2. Easing: It controls the rate of change of the animation. Common easing functions include linear interpolation, deceleration, acceleration etc.
3. Delay: You can introduce a delay before the animation starts, allowing more control over when the animation begins.
4. Interpolation: At the core of `FloatTweenSpec` is the interpolation of values from the start to the end over the animation's duration.

Let's see the implementation details and how it calculates the value.

> *This is not full implementation. I trimmed this to focus on parts we are interested in. You can check full implementation [here](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/FloatAnimationSpec.kt;l=201?q=TweenSpec&ss=androidx%2Fplatform%2Fframeworks%2Fsupport).*

```kotlin
public class FloatTweenSpec(
    public val duration: Int = DefaultDurationMillis,
    public val delay: Int = 0,
    private val easing: Easing = FastOutSlowInEasing
) : FloatAnimationSpec {
    .
    .
    .
    override fun getValueFromNanos(
        playTimeNanos: Long,
        initialValue: Float,
        targetValue: Float,
        initialVelocity: Float
    ): Float {
        val clampedPlayTimeNanos = clampPlayTimeNanos(playTimeNanos)
        val rawFraction = if (duration == 0) 1f else clampedPlayTimeNanos / durationNanos.toFloat()
        val fraction = easing.transform(rawFraction)
        return lerp(initialValue, targetValue, fraction)
    }
    .
    .
    .
}
```

1. It first calculates how much time has passed (playTime - delayNanos), and clamps this value between 0 and durationNanos. `clampedPlayTimeNanos` function above, exactly does that.
2. The next step is to compute the fraction of the animation's progress (`rawFraction = clampedPlayTimeNanos / durationNanos`). If provided duration is zero then the animation would stop immediately with target value.
3. Then, it applies the easing function (`fraction = easing.transform(rawFraction)`) to modify the fraction according to the specified easing curve.
4. Finally, it interpolates between the start and end values using the `lerp()` function, which performs a linear interpolation between the two values based on the eased fraction.

To boildown this, easing functions convert raw fraction of animation progress into a fraction that follows a specific curve. You can read about this in more detail [here](https://medium.com/androiddevelopers/easing-in-to-easing-curves-in-jetpack-compose-d72893eeeb4d).

Following are some more built-in specs that you might want to check out. I will leave that up to you to explore 🙂

- `SpringSpec`
- `RepeatableSpec`
- `InfiniteRepeatableSpec`

## Parting Thoughts

That’s all for this part. I know this would be a bit overwhelming. I would suggest following this blog along with Animation API open within Android Studio or [cs.android.com](cs.android.com).

So far we have just seen a blackbox which is function of time powering Compose animation. I hope this sets a nice foundation for the next part. In the next part we will see how Compose Animation API leverage this blackbox. Stay tuned!
