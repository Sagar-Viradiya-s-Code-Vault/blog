---
title: "Compose Animation, Under The Hood - Part II"
summary: A deep dive on Animatable API.
date: 2024-11-30
weight: 1
tags: ["Compose", "Animation"]
cover:
  image: images/header.jpeg 
  caption: "[Generated With AI](https://www.bing.com/images/create/generate-header-image-for-blog-explaining-internal/1-66ffb250fccc43f588e2e34db8724eb4?id=JJRlbQKelN%2fMt%2f3JS0rXFw%3d%3d&view=detailv2&idpp=genimg&thId=OIG1.NpAOOqqUxD4ksSM_1iMa&skey=nRgIOQdbVScL_nCqQ1_pUwJdKJObIw0F24OkIDRyqH0&FORM=GCRIDP&mode=overlay)"
  hiddenInList: false
---

## Context

In [Part-I]({{< ref "/posts/compose-animation-part-02/index.md" >}}), I explained how the Compose animation system leverages the low-level animation API to calculate the animation value for a given frame (playtime of animation). If you haven’t read that I would highly recommend reading that first before continuing here.

Things will get interesting in this part as we will explore how the Compose Animation system was built on top of Animation API. We will see how Animation API we saw in Part I gets consumed by Compose's Animatable API.

## Animatable

Animatable is the stateful API. This means it has a Composable state wrapped in it that gets updated periodically for the duration of the animation. The state value exposed by Animatable is something end users use within Composable. Since Compose recomposes on every state change, the value exposed by Animatable causes rapid change in UI resulting in animation.

This API also keeps track of the progress of the animation. It knows where animation has reached based on the time passed since animation started and the duration of animation provided by the user.

The real beauty of this API is coroutine. Calling `animateTo` function causes thread suspension until it finishes the animation. This unlocks the possibility of running multiple animations parallelly or sequentially. Here is the cool animation that I found on [Lottie](https://lottiefiles.com/free-animation/laptop-animatiion-BBDW2PdPPr) and reimplemented in Compose.

<video controls autoplay loop muted style="max-width: 100%; height: auto;">
    <source src="/videos/animation.mp4" type="video/mp4">
    Your browser does not support the video tag.
</video>

If you want to check implementation of above animation and more such sample animations then check out my open-source project [koreography](https://github.com/sagar-viradiya/koreography).

With this basic context on API, let's quickly see the code sample to know how you can consume this API.

```kotlin
// Compose scope

val isVisible by remember { mutableStateOf(false) }

val alphaAnimatable = remember { Animatable(0f) }

LaunchedEffect(isVisible) {
    // Coroutine Scope
    alphaAnimatable.animateTo(if (isVisible) 1f else 0f)  // Suspend function
}

Image(
    modifier = Modifier.graphicsLayer { alpha = alphaAnimatable.value },
    painter = painterResource(id = R.drawable.image),
    contentDescription = "Image"
)
```

Here we are triggering alpha animation on `isVisible` state change. Within launched effect, we call `animateTo` on `alphaAnimatable`.  This will cause alpha to be animated towards 0 or 1 based on the `isVisible` state.

Notice how we are consuming alpha value exposed by Animatable within Image composable.  The `alphaAnimatable.value` is essentially the value of the state wrapped in Animatable.

Nothing complicated here, this is just a Composable going through multiple recompositions (or relayout and redraw since we are using graphics layer API in the example to consume state) for a fixed duration of Animation. But how does this state changes happens?

Let's understand this with the following visuals.

<video controls autoplay loop muted style="max-width: 100%; height: auto;">
    <source src="/videos/animatable_frame_listner_flow.mp4" type="video/mp4">
    Your browser does not support the video tag.
</video>

Animatable wraps three things within. Animation, State, and FrameListner. 

1.  FrameListener listens for the frames. When the next frame is ready to render, it gets a callback with frame time in nanoseconds.
2.  Animatable then calculates the playtime of animation. By simply subtracting the first frame time (when the animation starts it stores the frame time) from the current frame time.
3.  This playtime is then fed to Animation API. In Part I, we saw Animation API is a function of time. It accepts the playtime and calculates the value of animation at that particular playtime. Animation API then outputs the calculated value.
4.  The calculated value is then applied to the state.

The above 4 steps happen for each frame for the duration of Animation. Syncing with the system frame plays a crucial role here in keeping track of the progress of animation as well as calculating playtime.

Alright, that's all on the theory behind Animatable API. Let's see the above four steps in terms of the implementation perspective. Let's start with the entry point, the `animateTo` function call from the example earlier.

> *For the scope of this blog, I am only covering target-based animation. I will leave decay-based animation up to you to explore. Hopefully, it will be easier to explore after reading this blog.*

## animateTo 🧐

```kotlin
suspend fun animateTo(
    targetValue: T,
    animationSpec: AnimationSpec<T> = defaultSpringSpec,
    initialVelocity: T = velocity,
    block: (Animatable<T, V>.() -> Unit)? = null
): AnimationResult<T, V> {
    val anim = TargetBasedAnimation(
	animationSpec = animationSpec,
	initialValue = value,
	targetValue = targetValue,
	typeConverter = typeConverter,
	initialVelocity = initialVelocity
    )
    return runAnimation(anim, initialVelocity, block)
}
```

If we see the implementation of `animateTo` function, all it does is create  `TargetBasedAnimation` passing `AnimationSpec`, target value, and other params. It then kicks in animation calling the internal function `runAnimation` passing newly created Animation and other params.

Alright, Let's dig further and see where calling `runAnimation` leads.

```java
.
.
.
while (lateInitScope!!.isRunning) {
    val durationScale = coroutineContext.durationScale
    animation.callWithFrameNanos { frameTimeInNano ->
        lateInitScope!!.doAnimationFrameWithScale(
            frameTimeInNano, durationScale, animation, this, block
        )
    }
}
.
.
.

```

> *For simplicity, I trim down code to focus on important things.*

Remember I mentioned earlier?, the four steps we saw keep repeating for each frame for the duration of the Animation. Well, above while loop takes care of that. 

The `lateInitScope` is of type `AnimationScope`. `AnimationScope` is a class that provides all the animation related info specific to an animation run. Wrapping many things such as if animation is in progress (`isRunning` check in while loop above), current velocity of the animation, and Compose state that gets updated for each frame, etc.

`isRunning` is true until the animation is in progress and at the last frame of animation it gets toggle to false. This would terminate while loop leading to the end of state updates as well.

Let's enter the while loop. We have a call to `callWithFrameNanos` function accepting lambda that gets frame time in nanoseconds. This lambda will get invoked for each frame. This is essentially the first step we saw earlier, where we listen for the system frame. Zooming in further on  `callWithFrameNanos` function leads us to a function call on the following.

```kotlin
suspend fun <R> withFrameNanos(onFrame: (frameTimeNanos: Long) -> R): R {
    return coroutineContext.monotonicFrameClock.withFrameNanos(onFrame)
}
```

`withFrameNanos` function delegates call to `monotonicFrameClock`. Following is the abstraction of `monotonicFrameClock`.

```kotlin
interface MonotonicFrameClock: CoroutineContext.Element {
    /**
    * Suspends until a new frame is requested, immediately invokes [onFrame]   
    * with the frame time in nanoseconds in the calling context of frame       	
    * dispatch, then resumes with the result from [onFrame].
    */
    suspend fun <R> withFrameNanos(onFrame: (frameTimeNanos: Long) -> R): R
    .
    .
    .
}
```

The word abstraction is important here. We will see Android implementation in a bit. Before that, I want to address Compose multiplatform branching off from here. Each platform (iOS, Web, Windows, macOS, and Linux) has an implementation of this interface to listen for system frames. This makes listening to system frames on different platforms possible.

Coming back to Android implementation, Choreographer is being used to listen to frames. Choreographer is a framework-level utility class that coordinates frame rendering by syncing with device refresh rate. So for a 60 hz device, it schedules frame rendering every 16.67 ms. Besides scheduling it also exposes API to listen to the next frame. Following is the implementation of  MonotonicFrameClock on Android.

```kotlin
class AndroidUiFrameClock internal constructor(
    val choreographer: Choreographer,
    private val dispatcher: AndroidUiDispatcher?
) : androidx.compose.runtime.MonotonicFrameClock {

    override suspend fun <R> withFrameNanos(onFrame: (Long) -> R): R 	{
        .
        .
        return suspendCancellableCoroutine { co ->
            val callback = Choreographer.FrameCallback { frameTimeNanos ->
            co.resumeWith(runCatching { onFrame(frameTimeNanos) })
        }
        .
        .
    }
}
```

The implementation above converts callback-based API from Choreographer to a suspending call. The function suspends waiting for the next frame and once it is ready it invokes `onFrame` lambda passing frame time in nanoseconds.

Alright, so now we have frame time in nanoseconds. Let's resume on `onFrame` lambda. Remember the place where we pass this lambda?

Let me help you.

```java
.
.
.
while (lateInitScope!!.isRunning) {
    val durationScale = coroutineContext.durationScale
    animation.callWithFrameNanos { frameTimeInNano ->
        lateInitScope!!.doAnimationFrameWithScale(
            frameTimeInNano, durationScale, animation, this, block
        )
    }
}
.
.
.
```

`callWithFrameNanos` function above accepts the lambda. This is where we dig the rabbit hole to see how frame time is being listened on Android.

With frame time let's see how it is being used to calculate the animation playtime. This is the second step from the four steps we saw earlier. `doAnimationFrameWithScale` function call above does that.

```kotlin
private fun doAnimationFrameWithScale(
    frameTimeNanos: Long,
    durationScale: Float,
    anim: Animation<T, V>,	
    state: AnimationState<T, V>,
    block: AnimationScope<T, V>.() -> Unit
) {
    val playTimeNanos = if (durationScale == 0f) {
        anim.durationNanos
    } else {
        ((frameTimeNanos - startTimeNanos) / durationScale).toLong()
    }
    doAnimationFrame(frameTimeNanos, playTimeNanos, anim, state, block)
}
```

The above function calculates playtime by subtracting the current frame time from the first frame time when the animation started. Notice this function also adjusts playtime to scale up or down based on the duration scale passed. This is something useful during the testing if you want to make your animation slow or fast. 

Finally, calculated playtime is passed to `doAnimationFrame` function. Here is the function implementation.

```kotlin
private fun doAnimationFrame(
    frameTimeNanos: Long,
    playTimeNanos: Long,
    anim: Animation<T, V>,
    state: AnimationState<T, V>,
    block: AnimationScope<T, V>.() -> Unit
) {
    lastFrameTimeNanos = frameTimeNanos
    value = anim.getValueFromNanos(playTimeNanos)
    velocityVector = anim.getVelocityVectorFromNanos(playTimeNanos)
    val isLastFrame = anim.isFinishedFromNanos(playTimeNanos)
    if (isLastFrame) {
        finishedTimeNanos = lastFrameTimeNanos
        isRunning = false
    }
    updateState(state)
}
```

This function feeds playtime to `getValueFromNanos` function on TargetBasedAnimation which calculates animation value. This is essentially the Animation API we saw in Part I which is a function of time. The black box I talked about in Part I is being used here.

Also, it calls  `isFinishedFromNanos` function on TargetBasedAnimation to check if the current frame is the last frame of animation. If it is then it will set the `isRunning` to false. This will terminate the while loop to stop listening for frame further and terminate the animation.

At last, `updateState`  function updates the Compose state that causes the recompisition. This completes the full circle for a frame. This keeps on repeating for the duration of Animation.

## Parting Thoughts

Damn! That's a hell lot of things just to render one frame of animation. Hopefully, this was interesting and insightful. We saw system frame listener and Animation API are the cores powering Animatable API.

In the next part, we will explore animates*AsState and Transition APIs. Stay tuned!