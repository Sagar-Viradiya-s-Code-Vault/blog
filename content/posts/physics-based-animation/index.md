---
title: "Android physics based animation"
summary: "May the force be with you"
date: 2017-08-17
weight: 1
tags: ["Animation", "Physics based animation", "Kotlin"]
cover:
  image: images/header.jpg 
  caption: Photo by [Brian McGowan](https://unsplash.com/@sushioutlaw?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/photos/white-robot-toy-on-black-background-ggg_B1MeqQk?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)
  hiddenInList: false
---

> *May the mass * acceleration be with you*

Android has nice animation API. One can easily choreograph animation based on inbuilt interpolator or could implement own to get nice effect in animation. One thing that was missing till now was dynamic nature of animation just like how real life object behave when we apply force on it and in between if we perform some action on it how that force changes. Using **Animator** API we can animate object in smooth way. For example you can initially accelerate and decelerate object(and other such effects using interpolators). But animator API is static (once you start animation you can’t modify animation), has fix duration and there is no force associate with it. What if in middle of animation you want to change target of object ?. You can’t modify current animation. You have to cancel current one, create new animation and start it. Overall animation won’t be smooth and natural. There will be abrupt change in animation. So how do we make such transition smooth ?

{{< iframe src="https://giphy.com/embed/xT9KVmZwJl7fnigeAg" width="480" height="259" frameBorder="0" class="giphy-embed" >}}

This year at google IO, android team announced new animation API called [**Physics Based Animation**](https://developer.android.com/topic/libraries/support-library/preview/physics-based-animation.html). It is part of support library so that it is backward compatible till API 16. So let’s explore what is physics based animation and how it is different from normal animation.

## Physics based animation vs. Non-Physics based animation

If you want to create normal animation all you need is duration, start value, end value and interpolator. Animation is static and duration is fix. Velocity of animation is govern by duration of animation and interpolator. There is no external force involve here which is controlling progress of animation. Physics based animation is driven by force. Force you apply controls velocity of animation. Also you can modify target value of animation while animation is going on and physics based animation takes care of applying new force on existing velocity, which makes a continuous transition to the new target. To better understand difference between two look at the following figure.

{{< iframe src="images/physics-based-animation.gif" width="700" height="500" frameBorder="0" class="giphy-embed">}}

As you can see in case of object animator(Left) there is abrupt change in animation if target value changes and physics based animation on the other hand is smooth. So far I have covered basics of physics based animation and how it is different from normal animation. Now it’s time to get your hand dirty and learn how to implement physics based animation.

{{< iframe src="https://giphy.com/embed/13GIgrGdslD9oQ" width="480" height="288" frameBorder="0" class="giphy-embed">}}

Android physics based animation API provides two types of animation.

### Spring Animation

Animation of view property is driven by spring force. The properties of a spring, the value, and the velocity are used in creating a spring-based animation.

### Fling Animation

Animation of view property is driven by frictional force. Fling animation has an initial momentum and gradually slows down.

Let’s explore both animation

## Spring Animation

Spring animation is driven by spring force which depends on spring *stiffness, damping ratio and object’s final position*. It’s a same effect you get when you pull object tied to spring and release. Object bounce back and forth and finally comes to it’s initial position. Before diving into API it is important to understand properties of spring that determines spring force.

**Stiffness** measures the strength of the spring. In simple word you will require more force to pull object tied to spring which has high stiffness. Basically it tells how tight spring is.

**Damping ratio** describes a gradual reduction in a spring oscillation. In simple word spring is more bouncy if it has damping ratio less than zero.

In a spring-based animation, the SpringForce class lets you customise spring's stiffness, its damping ratio, and its final position. As soon as the animation begins, the spring force updates the animation value and the velocity on each frame. The animation continues until the spring force reaches an equilibrium.

[`SpringAnimation`](https://developer.android.com/reference/android/support/animation/SpringAnimation.html) class help you to create a spring animation for an object. Now let’s see how to achieve following animation.

{{< iframe src="images/spring_animation.gif" width="480" height="640" frameBorder="0" class="giphy-embed">}}

To build a spring animation, you need to create an instance of the [`SpringAnimation`](https://developer.android.com/reference/android/support/animation/SpringAnimation.html) class and provide an object, an object’s property that you want to animate, and an optional final spring position where you want the animation to rest.

> *At the time of creating a spring animation, the final position of the spring is optional. Though, it must be defined before starting the animation.*

```kotlin
private val springForce: SpringForce by lazy(LazyThreadSafetyMode.NONE) {
    SpringForce(0f).apply {
        stiffness = SpringForce.STIFFNESS_MEDIUM
        dampingRatio = SpringForce.DAMPING_RATIO_HIGH_BOUNCY
    }
}

private val springAnimationTranslationX: SpringAnimation by lazy(LazyThreadSafetyMode.NONE) {
    SpringAnimation(android_bot, DynamicAnimation.TRANSLATION_X).setSpring(springForce)
}

private val springAnimationTranslationY: SpringAnimation by lazy(LazyThreadSafetyMode.NONE) {
    SpringAnimation(android_bot, DynamicAnimation.TRANSLATION_Y).setSpring(springForce)
}
```

As you can see we have created spring animation instance for x and y translation of view android_bot(ImageView) and attached spring force with each instance. Now all we want is to implement touch listener on image so that we can move the image when we drag image. Finally when we release image we will start two spring animation on image(one for translationX and one for translationY).

```kotlin
private fun setupTouchListener() {

    android_bot.setOnTouchListener { view, motionEvent ->

        when(motionEvent?.action) {

            MotionEvent.ACTION_DOWN -> {
                xDiffInTouchPointAndViewTopLeftCorner = motionEvent.rawX - view.x
                yDiffInTouchPointAndViewTopLeftCorner = motionEvent.rawY - view.y

                springAnimationTranslationX.cancel()
                springAnimationTranslationY.cancel()
            }

            MotionEvent.ACTION_MOVE -> {
                android_bot.x = motionEvent.rawX - xDiffInTouchPointAndViewTopLeftCorner
                android_bot.y = motionEvent.rawY - yDiffInTouchPointAndViewTopLeftCorner
            }

            MotionEvent.ACTION_UP -> {
                springAnimationTranslationX.start()
                springAnimationTranslationY.start()
            }
        }

        true
    }
}
```

That’s it, you done with spring animation!!!

## Fling Animation

Fling animation is driven by frictional force. If friction is high object will come to rest quickly and if it is low object will travel more. It has an initial momentum, which is mostly received from the gesture velocity, and gradually slows down.

The [`FlingAnimation`](https://developer.android.com/reference/android/support/animation/FlingAnimation.html) class lets you create a fling animation for an object. To build a fling animation, create an instance of the [`FlingAnimation`](https://developer.android.com/reference/android/support/animation/FlingAnimation.html) class and provide an object and the object's property that you want to animate. Let’s see how to implement following animation.

{{< iframe src="images/fling_animation.gif" width="480" height="640" frameBorder="0" >}}

```kotlin
val flingAnimationX: FlingAnimation by lazy(LazyThreadSafetyMode.NONE) {
    FlingAnimation(android_bot, DynamicAnimation.X).setFriction(1.1f)
}

val flingAnimationY: FlingAnimation by lazy(LazyThreadSafetyMode.NONE) {
    FlingAnimation(android_bot, DynamicAnimation.Y).setFriction(1.1f)
}
```

Here we have created two instance of fling animation for animating x and y position of image. Friction for both animation is set to 1.1. Now whenever we perform fling gesture on image we want to animate our image in gesture direction. Let’s see code for that

```kotlin
val gestureListener = object : GestureDetector.SimpleOnGestureListener() {

    override fun onDown(e: MotionEvent?): Boolean {
        return true
    }

    override fun onFling(e1: MotionEvent?, e2: MotionEvent?, velocityX: Float, velocityY: Float): Boolean {

        flingAnimationX.setStartVelocity(velocityX)
        flingAnimationY.setStartVelocity(velocityY)

        flingAnimationX.start()
        flingAnimationY.start()

        return true
    }
}

val gestureDetector = GestureDetector(context, gestureListener)

android_bot.setOnTouchListener { _, motionEvent ->
    gestureDetector.onTouchEvent(motionEvent)
}
```

Using SimpleOnGestureListener we can easily detect fling gesture on image. Callback to onFling method will give us fling gesture velocity in X and Y directions. We use these velocities to set start velocity of each fling animation and finally we start both animation. One thing to note here is we don’t want our animation to go beyond our screen size. To avoid that we need to set min and max value for each fling animation.

```kotlin
android_bot.viewTreeObserver.addOnGlobalLayoutListener(object : ViewTreeObserver.OnGlobalLayoutListener {
    override fun onGlobalLayout() {
        flingAnimationX.setMinValue(0f).setMaxValue((screenSize.x - android_bot.width).toFloat())
        flingAnimationY.setMinValue(0f).setMaxValue((screenSize.y - android_bot.height).toFloat())
        android_bot.viewTreeObserver.removeOnGlobalLayoutListener(this)
    }
})
```

Here min value for both animation is set to zero which is min value for x and y on screen. Max value for both animation is max value of x and y on screen minus image width and hight respectively.

Woo hoo!!! We are done with both type of physics based animation. Next we will see how to achieve chain animation where one animation depends on other.

## Chained Spring Animation

Physics based animation API has two ways to start animation. First we already saw is using *`start()`* method. There is another way you can start animation and i.e using *`animateToFinalPosition()`* method. This method perform two tasks. First it will set the target value of animation and second it will start the animation, if it has not started. This will allow us to change the target value in middle of animation. In order to implement following chained spring animation this method will be helpful.

{{< iframe src="images/chained_spring_animation.gif" width="480" height="640" frameBorder="0">}}

Since animation of last image depends on second and animation of second image depends on first we have to change target value of both spring animation in between animation if target value of first image changes. For such an animation, it’s more convenient to use *`animateToFinalPosition()`* method. Let’s see how to implement chained spring animation.

```kotlin
val firstSpringAnimationX by lazy(LazyThreadSafetyMode.NONE) {
    createSpringAnimation(android_bot1, DynamicAnimation.TRANSLATION_X)
}

val firstSpringAnimationY by lazy(LazyThreadSafetyMode.NONE) {
    createSpringAnimation(android_bot1, DynamicAnimation.TRANSLATION_Y)
}

val secondSpringAnimationX by lazy(LazyThreadSafetyMode.NONE) {
    createSpringAnimation(android_bot2, DynamicAnimation.TRANSLATION_X)
}

val secondSpringAnimationY by lazy(LazyThreadSafetyMode.NONE) {
    createSpringAnimation(android_bot2, DynamicAnimation.TRANSLATION_Y)
}

private fun <K> createSpringAnimation(view: K, property: FloatPropertyCompat<K>) : SpringAnimation {
    return SpringAnimation(view, property).setSpring(SpringForce())
}
```

First we create spring animation for 2nd and 3rd image on translationX and translationY property. Next we have to implement translationX and translationY animation for first image so that when we drag it will move.

```kotlin
private fun setupOnTouchListener() {

    ...

    android_bot.setOnTouchListener { view, motionEvent ->

        if(motionEvent.action == MotionEvent.ACTION_MOVE) {

            val deltaX = motionEvent.rawX - lastX
            val deltaY = motionEvent.rawY - lastY

            view.translationX = deltaX + view.translationX
            view.translationY = deltaY + view.translationY

            firstSpringAnimationX.animateToFinalPosition(view.translationX)
            firstSpringAnimationY.animateToFinalPosition(view.translationY)

        }

        lastX = motionEvent.rawX
        lastY = motionEvent.rawY

        return@setOnTouchListener true
    }
}
```

As we animate first image, animation of second image begin immediately. And if second image’s animation is already going on change in target value will be applied and animation will be updated in next frame. Now finally we want third image to listen to change in animation of second image and update it’s animation.

```kotlin
firstSpringAnimationX.addUpdateListener { _, value, _ ->
    secondSpringAnimationX.animateToFinalPosition(value)
}

firstSpringAnimationY.addUpdateListener { _, value, _ ->
    secondSpringAnimationY.animateToFinalPosition(value)
}
```

TranslationX and translationY of second image also applied to third image and it will start animating third image.

Imagine implementing animation which is driven by force without physics based animation library. I am not saying it won’t be possible but it will be time consuming and difficult to maintain. Thanks to google and android team for such a good animation API. Having force controlling your animation makes it more natural.

To explore more about physics based animation watch following IO session.

{{< youtube BNcODK-Ju0g >}}

You can find source code of sample app I have created to show physics based animation on [github](https://github.com/sagar-viradiya/AndroidPhysicsAnimation). And if you are kotlin fan like me good news for you is it is 100% written in kotlin.

You can find me on [**twitter**](https://twitter.com/viradiya_sagar), [**G+**](https://plus.google.com/u/1/+SagarViradiya_Profile) and [**LinkedIn**](https://www.linkedin.com/in/sagarviradiya/).

That’s it for now folks, Happy coding :)

> *This was originally posted on [Medium](https://medium.com/android-news/android-physics-based-animation-cf0cc125830f)*

