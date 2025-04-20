---
title: "Compose Animation, Under The Hood - Part III"
summary: A deep dive on animate*AsState and Transition API.
date: 2025-04-06
weight: 1
tags: ["Compose", "Animation"]
cover:
  image: images/header.jpeg 
  caption: "[Generated With AI](https://www.bing.com/images/create/generate-header-image-for-blog-explaining-internal/1-66ffb250fccc43f588e2e34db8724eb4?id=JJRlbQKelN%2fMt%2f3JS0rXFw%3d%3d&view=detailv2&idpp=genimg&thId=OIG1.NpAOOqqUxD4ksSM_1iMa&skey=nRgIOQdbVScL_nCqQ1_pUwJdKJObIw0F24OkIDRyqH0&FORM=GCRIDP&mode=overlay)"
  hiddenInList: false
---

## Context

In [Part-II]({{< ref "/posts/compose-animation-part-02/index.md" >}}) we explored the low-level [`Animatable`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animatable.kt;drc=a3f12e1332e6df5a3da7cb979733c58d8eb9364e;l=50) API, which leverages [`Animation`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animation.kt;drc=975e20a49c9f51099ee4b8e302fd92add31e27a2;l=41) API and coroutines to unlock the possibility of composing complex animations running parallelly or sequentially.

While this API offers fine control over the animation, most of the time you want simple animations. Things like scaling, rotating or changing alpha, based on some state change within compose. Sometimes you need to run multiple animations together based on state change.

[`animate*AsState`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/AnimateAsState.kt;drc=e5cf676e339f2807cf29bf28fe7d8364931a48eb;l=389) and [`Transition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=888) APIs built on top of [`Animatable`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animatable.kt;drc=a3f12e1332e6df5a3da7cb979733c58d8eb9364e;l=50) and [`Animation`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animation.kt;drc=975e20a49c9f51099ee4b8e302fd92add31e27a2;l=41), respectively, are the two high-level APIs that allow you to implement such animations. 
In this part we will see the implementation details of these two APIs.

If you haven't read the first two parts I would suggest reading them before continuing here.

1. [Part-I]({{< ref "/posts/compose-animation-part-01/index.md" >}}) covering low level Animation API.
2. [Part-II]({{< ref "/posts/compose-animation-part-02/index.md" >}}) covering Animatable API.

## animate\*AsState

This is just a composable wrapping [`Animatable`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animatable.kt;drc=a3f12e1332e6df5a3da7cb979733c58d8eb9364e;l=50) and exposing its state. Since this is the composable function it can observe state change. Based on state value, it internally triggers the animation towards the new target. So essentially, the input for this API is state, and the output is the state object that your composable observes and recomposes.

Let's understand this using an example.

```kotlin
// Compose Context  
val isVisible by remember { mutableStateOf(false) }

val alphaState: State<Float> = animateFloatAsState(
    targetValue = if (isVisible) 1f else 0f, 
    label = "alphaAnimation"
)

Image(
    modifier = Modifier.graphicsLayer { alpha = alphaState.value },
    painter = painterResource(id = R.drawable.image),
    contentDescription = "Image"
)
```

Here we are, toggling the visibility of image based on state change. Notice `isVisible` above is the state value observed by `animateFloatAsState`. If it is true, then the target value of animation is 1 else 0. `animateFloatAsState` composable returns another state object of float. This state object is then applied to the alpha of the image. 

The above example animates the value of type float, but there are composables to animate Int (`animateIntAsState`), Offset (`animateOffsetAsState`), Rect (`animateRectAsState`), DP(`animateDpAsState`) etc. In fact, you can animate your custom object type if you provide `TwoWayConverter`(Refer to [Part-I]({{< ref "/posts/compose-animation-part-01/index.md" >}}) if you want to know more about `TwoWayConverter`) to the generic `animateValueAsState` function. All the built-in composables, `animateIntAsState`, `animateFloatAsState`, `animateRectAsState`, `animateDpAsState`, etc, internally call `animateValueAsState` only by passing `TwoWayConverter` of respective type.

Hopefully, by now, you have a better understanding of the API and curious to know its implementation. Since everything boils down to `animateValueAsState`, let's zoom into the function.

```kotlin
@Composable
public fun <T, V : AnimationVector> animateValueAsState(
    targetValue: T,
    typeConverter: TwoWayConverter<T, V>,
    animationSpec: AnimationSpec<T> = remember { spring() },
    visibilityThreshold: T? = null,
    label: String = "ValueAnimation",
    finishedListener: ((T) -> Unit)? = null
): State<T> {

    .
    .
    val animatable = remember { Animatable(targetValue, typeConverter, visibilityThreshold, label) }
    .
    .
    .
    val channel = remember { Channel<T>(Channel.CONFLATED) }
    SideEffect { channel.trySend(targetValue) }
    LaunchedEffect(channel) {
        for (target in channel) {
            val newTarget = channel.tryReceive().getOrNull() ?: target
            launch {
                if (newTarget != animatable.targetValue) {
                    animatable.animateTo(newTarget, animSpec)
                    listener?.invoke(animatable.value)
                }
            }
        }
    }
    return toolingOverride.value ?: animatable.asState()
}
```

> *This is the trim-down version of full implementation.*

This composable remembers [`Animatable`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animatable.kt;drc=a3f12e1332e6df5a3da7cb979733c58d8eb9364e;l=50) with an initial target. Remember I said wrapping [`Animatable`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animatable.kt;drc=a3f12e1332e6df5a3da7cb979733c58d8eb9364e;l=50) ? This is exactly what I meant. 

Internally, it uses channel to dispatch target change. The `SideEffect` above gets executed every time there is a change in the target. Within this block, we are dispatching new target values to the channel. 

This channel is then observed within `LaunchedEffect`, and if the new target is different from the existing target, it starts the animation. 

Finally, outside the `LaunchedEffect` at the end, the function returns state from the [`Animatable`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animatable.kt;drc=a3f12e1332e6df5a3da7cb979733c58d8eb9364e;l=50). This state object is updated on each frame by [`Animatable`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animatable.kt;drc=a3f12e1332e6df5a3da7cb979733c58d8eb9364e;l=50) resulting in Animation.

> *`toolingOverride` above is for overriding the actual animation state with the default tooling state. This will be used while composable is in preview mode.*

That concludes [`animate*AsState`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/AnimateAsState.kt;drc=e5cf676e339f2807cf29bf28fe7d8364931a48eb;l=389) API. A simple wrapper over [`Animatable`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animatable.kt;drc=a3f12e1332e6df5a3da7cb979733c58d8eb9364e;l=50).

## Transition

Transition API triggers multiple animations on state change. For each animation, it maintains a state object tracking its current value, whether the animation is finished, etc.  This state object holds the reference of low-level [`Animation`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animation.kt;drc=975e20a49c9f51099ee4b8e302fd92add31e27a2;l=41) API and delegates animation to it. So essentially, Transition API is a wrapper over [`Animation`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animation.kt;drc=975e20a49c9f51099ee4b8e302fd92add31e27a2;l=41) API.

Before we dive into the implementation details, Let's see how you can consume this API.

```kotlin
// Compose Context
var isVisible by remember { mutableStateOf(false) }

val transition = updateTransition(target = isVisible)

val alpha by transition.animateFloat(
    label = "Alpha"
) { isVisible ->
    if (isVisible) 1f else 0f
}

val scale by transition.animateFloat(
    label = "Scale"
) { isVisible ->
    if (isVisible) 1f else 0f
}
```
We have `isVisibile` mutable state that is being passed to [`updateTransition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=0c832c139dc844fcc611c7b5cdb1496ba5071a3a;l=88) composable. This composable internally creates, remembers, and returns an instance of [`Transition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=888). 

With this [`Transition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=888) instance, you can create child animations. In the above example, two animations are created. The first animation is for animating alpha, and the second animates the scale. Notice the trailing lambda of `animateFloat` function. It exposes the `isVisibile` state value that is observed by transition. Based on this state value we decide the target for child animation.

Finally, you can apply this alpha and scale to animate composable. 

```kotlin
Image(
    modifier = Modifier.graphicsLayer { 
	    alpha = alpha 
	    scale = scale
	},
    painter = painterResource(id = R.drawable.image),
    contentDescription = "Image"
)
```

Now that we are clear on how we can consume Transition API, let's zoom into the implementation 🧐

It starts with [`updateTransition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=0c832c139dc844fcc611c7b5cdb1496ba5071a3a;l=88) composable. 

```kotlin
@Composable
public fun <T> updateTransition(targetState: T, label: String? = null): Transition<T> {
    val transition = remember { Transition(targetState, label = label) }
    transition.animateTo(targetState)
    DisposableEffect(transition) {
        onDispose {
            transition.onDisposed()
        }
    }
    return transition
}
```
Nothing complicated here. It just creates an instance of Transition and remembers it. Since this is composable, every time there is a change in the target state that is being passed to this function, it recomposes and calls `animateTo` on Transition passing a new target state. Transition then triggers all child animations towards their new respective targets.

It also takes care of disposing transition when the composition exits. Finally, it returns the created instance of transition.

The above composable is merely a thin wrapper delegating target change to [`Transition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=888). Let's zoom into the [`Transition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=888) implementation.

```kotlin
public class Transition<S> {
    .
    .
    .
    public val animations: List<TransitionAnimationState<*, *>>
        get() = _animations
    .
    .
    .
    internal fun animateTo(targetState: S) {
        .
        .
        .    
        while (isActive) {
            withFrameNanos {
                if (!isSeeking) { frameTimeNanos ->
                    onFrame(frameTimeNanos / AnimationDebugDurationScale, durationScale)
                }
            }
        }    
        .
        .
        .
    }
}
```
> *This is the ultra trim-down version of the actual implementation. There is a lot going on in this class and I encourage you to explore it.*

Two things I want to focus on here are a list of [`TransitionAnimationState`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=1303) and `animateTo` function. 

The list of [`TransitionAnimationState`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=1303) above holds the state of each child animation. This list is populated when you create the child animation on [`Transition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=888). Remember this?

```kotlin
val alpha by transition.animateFloat(
    label = "Alpha"
) { isVisible ->
    if (isVisible) 1f else 0f
}
```

This was one of the child animations we created earlier in the example. When you call `animate*` (`animateInt`, `animateFloat`, `animateIntOffset` etc) on transition, it internally creates [`TransitionAnimationState`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=1303), passing the initial target value and adds it to the list. 

The `animateTo` function is the entry point to [`Transition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=888) API, which will be called by [`updateTransition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=0c832c139dc844fcc611c7b5cdb1496ba5071a3a;l=88) composable function on state change. This function starts listening to new frames from the choreographer. In [Part-II]({{< ref "/posts/compose-animation-part-02/index.md" >}}) of this series, we saw the implementation details of frame listening. I am not covering it again as it is same here as well.

The trailing lambda of `withFrameNanos` will get called for each frame. This lambda gets the frame time in nanoseconds. Later, this frame time is used to calculate the playtime of animation within `onFrame` function. Again, the calculation of play time is very similar to what we saw in [Part-II]({{< ref "/posts/compose-animation-part-02/index.md" >}}). I would leave this to you as an exercise to explore `onFrame` function.

The calculated playtime of the animation is then fed to each child animation represented by [`TransitionAnimationState`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=1303). 

```kotlin
 _animations.fastForEach {
    .
    .
    it.onPlayTimeChanged(scaledPlayTimeNanos, scaleToEnd)
    .
    .
}
```

Now that playtime is available and fed to each child animation, let's see the implementation details of [`TransitionAnimationState`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=1303) to understand how animation value is calculated for a frame.

```kotlin
public inner class TransitionAnimationState<T, V : AnimationVector>) : State<T> {
    .
    .
    .
    public var animation: TargetBasedAnimation<T, V> by
        mutableStateOf(
            TargetBasedAnimation(
                animationSpec,
                typeConverter,
                initialValue,
                targetValue,
                initialVelocityVector
            )
        )
        private set
    .
    .
    .
    override var value: T by mutableStateOf(initialValue)
        internal set
    .
    .
    .
    internal fun onPlayTimeChanged(playTimeNanos: Long, scaleToEnd: Boolean) {
        val playTime = if (scaleToEnd) animation.durationNanos else playTimeNanos
        value = animation.getValueFromNanos(playTime)
        velocityVector = animation.getVelocityVectorFromNanos(playTime)
        if (animation.isFinishedFromNanos(playTime)) {
            isFinished = true
        }
    }
}
```
> *Again, this is the ultra trim-down version of the actual implementation.*

Let’s break down a simplified version of the [`TransitionAnimationState`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=1303) class and understand how it works.

This class represents the internal state of child animation. It implements `State<T>`, making it observable in Compose. Whenever its value changes, the UI will recompose accordingly. This is what you get back when you create child animation on transition.

```kotlin
val alpha by transition.animateFloat(
    label = "Alpha"
) { isVisible ->
    if (isVisible) 1f else 0f
}
```
The `animateFloat` above returns `State<Float>`, which is nothing but [`TransitionAnimationState`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=1303) 

The animation playtime we calculated is fed to `onPlayTimeChanged`, which internally feeds this playtime to [`Animation`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animation.kt;drc=975e20a49c9f51099ee4b8e302fd92add31e27a2;l=41) to calculate the animation value for the frame. This animation value is then applied to the state, resulting in recomposition of your Composable.

> *To understand how [`Animation`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animation.kt;drc=975e20a49c9f51099ee4b8e302fd92add31e27a2;l=41) calculates value based on playtime please head to [Part-I]({{< ref "/posts/compose-animation-part-01/index.md" >}}) of this series, where I detailed this.*

## Summary

This concludes the implementation details of two high-level Animation APIs, which animate things on state change. While these APIs look different from others, they leverage low-level Animation APIs to hide boilerplate code that you need to write otherwise. Let's summarise them.

### animate*AsState

1. This is a composable wrapping [`Animatable`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animatable.kt;drc=a3f12e1332e6df5a3da7cb979733c58d8eb9364e;l=50) and delegates animation to it.
2. Internally it uses channel to publish all state change and observe this channel within `LauchedEffect` to forward state changes to [`Animatable`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animatable.kt;drc=a3f12e1332e6df5a3da7cb979733c58d8eb9364e;l=50)

### Transition

1. [`updateTransition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=0c832c139dc844fcc611c7b5cdb1496ba5071a3a;l=88) composable is the entry point that internally creates a [`Transition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=888) object.
2. [`Transition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=888) react to state changes, start listening for frame changes, and forward them to all child animations
3. Child animations are wrapped in [`TransitionAnimationState`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=1303) object, which reacts to frame change from [`Transition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=888) and internally delegates animation to [`Animation`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Animation.kt;drc=975e20a49c9f51099ee4b8e302fd92add31e27a2;l=41) API.
4. The updated animation values are then applied to [`TransitionAnimationState`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=1303)  which causes recomposition

## Parting Thoughts

I know this is a bit heavy to digest. I would recommend reading this along with the full implementation open on [cs.android.com](cs.android.com).

If you are still here, you are brave! Pet yourself on the back. Honestly, I am unsure if I simplified this enough to sink this in. [`Transition`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/animation/animation-core/src/commonMain/kotlin/androidx/compose/animation/core/Transition.kt;drc=03da7b0f4fe28c1031276b125af612ada99e9e49;l=888) animation was by far difficult to explain as it involves multiple animations and additional state objects.

With this part, I am concluding this series on the internals of Compose animation API. I hope this series was insightful. If you have any suggestions or questions, please drop them in the comment below. Until then, stay curious!