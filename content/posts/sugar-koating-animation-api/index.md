---
title: "Sugar koating Physics based Animation API"
summary: A syntactic sugar on Android's Physics based Animation API
date: 2018-11-09
weight: 1
tags: ["Kotlin", "Android", "Android Animations"]
cover:
  image: images/header.jpeg 
  caption: "Photo by [Sarah Pflug](https://burst.shopify.com/@sarahpflugphoto?utm_campaign=photo_credit&utm_content=Free+Stock+Photo+of+Sugar+Dusted+Pie+—+HD+Images&utm_source=credit) on [Burst](https://burst.shopify.com/thanksgiving?utm_campaign=photo_credit&utm_content=Free+Stock+Photo+of+Sugar+Dusted+Pie+—+HD+Images&utm_source=credit)"
  hiddenInList: false
---

Roughly a year ago I wrote about [android physics based animation](https://android.jlelse.eu/android-physics-based-animation-cf0cc125830f) and also open sourced [sample app](https://github.com/sagar-viradiya/AndroidPhysicsAnimation) to demonstrate its implementation. I have been maintaining that repo and made few changes since then. Lately I decided to revisit it again and try to come up with kotlin extensions for API. In this post we will see few extension functions for [`SpringAnimation`](https://developer.android.com/reference/android/support/animation/SpringAnimation.html) and [`FlingAnimation`](https://developer.android.com/reference/android/support/animation/FlingAnimation.html) that will make code clean and easy to understand. So let’s dive in.

## Extensions for SpringAnimation

If you want to create `SpringAnimation` on a view property there are two ways as shown below.

```kotlin
val springAnimation = SpringAnimation(view, DynamicAnimation.Y)
val springAnimation = SpringAnimation(view, DynamicAnimation.Y, 0f)
```

Only difference between two is **final position** of spring. But this is however boring and we can make it more interesting using extension function so that we can express it in functional way.

```kotlin
fun <K> K.springAnimationOf(
    property: FloatPropertyCompat<K>,
    finalPosition: Float = Float.NaN
): SpringAnimation {
    return if (finalPosition.isNaN()) {
        SpringAnimation(this, property)
    } else {
        SpringAnimation(this, property, finalPosition)
    }
}
```

As you can see above extension function is on all type. So on any type you can call this function. Function takes 2 parameters, `property` to be animated and `finalPosition` which is optional in case if you want to create `SpringAnimation` without `finalPosition` as seen above. Now let’s see how you can create `SpringAnimation` in functional way using above extension function.

```kotlin
val springAnimation = view.springAnimationOf(DynamicAnimation.TRANSLATION_Y)

val springAnimation = view.springAnimationOf(DynamicAnimation.TRANSLATION_Y, 0f)
```

Now some of you might say you are abusing extension function because same thing can be done easily using normal constructor way and there is no point of using extension here.

Now some of you might say you are abusing extension function because same thing can be done easily using normal constructor way and there is no point of using extension here.

{{< iframe src="https://giphy.com/embed/3o7WTIVuAOn8lBYN7W" width="480" height="270" frameBorder="0" class="giphy-embed" >}}

Initially I had same thought but if you see in terms of readability I think using extension function make sense here. Let’s move on to next extension for `SpringAnimation`.

If you create `SpringAnimation` with `finalPosition`, internally `SpringAnimation` class will create `SpringForce` and attach with itself. However you may want to customise or create new `SpringForce` (in case `SpringAnimation` is created without `finalPosition`). It would look something like this.

```kotlin
val springForce = SpringForce(0f)
                    .setStiffness(SpringForce.STIFFNESS_MEDIUM)
                    .setDampingRatio(SpringForce.DAMPING_RATIO_HIGH_BOUNCY)

val springAnimation = SpringAnimation(view, DynamicAnimation.TRANSLATION_X).setSpring(springForce)
```

Let’s see extension function to simplify this.

```kotlin
inline fun SpringAnimation.withSpringForceProperties(func: SpringForce.() -> Unit): SpringAnimation {
  
    if (spring == null) {
        spring = SpringForce()
    }
    spring.func()
    return this
  
}
```

It’s an extension on `SpringAnimation` itself which takes another extension function on `SpringForce` as parameter. Since it is lambda with receiver we have access to all its private data members and methods within it. So we can access **`stiffness`**, **`dampingRatio`** and **`finalPosition`** of `SpringForce` within it. Notice it has null check for `SpringForce` in line number 3 because if your `SpringAnimation` is created without `finalPosition` then it won’t have `SpringForce` so it will create for you in line number 4. In line number 6 it will simply apply function on SpringForce and finally return `SpringAnimation` back in line number 7. Let’s see the usage.

```kotlin
val springAnimation = SpringAnimation(view, DynamicAnimation.TRANSLATION_X)
	.withSpringForceProperties {
		finalPosition = 0f
		stiffness = SpringForce.STIFFNESS_MEDIUM
		dampingRatio = SpringForce.DAMPING_RATIO_HIGH_BOUNCY
	}
```

Code is much cleaner and readable compare to one without extension function.

## Extension for FlingAnimation

Creating `FlingAnimation` is simple and straightforward as shown below.

```kotlin
val flingAnimation = FlingAnimation(view, DynamicAnimation.Y)
```

Same as `SpringAnimation` we can express this in a functional way using extension function.

```kotlin
fun <K> K.flingAnimationOf(property: FloatPropertyCompat<K>): FlingAnimation {
    return FlingAnimation(this, property)
}
```

Let’s see the usage.

```kotlin
val flingAnimation = view.flingAnimationOf(DynamicAnimation.Y)
```

That’s all I have :)

All extensions we saw are going to be part of [Android KTX](https://android.googlesource.com/platform/frameworks/support/+/androidx-master-dev/dynamic-animation/ktx/src/main/java/androidx/dynamicanimation/animation/DynamicAnimation.kt) in the future release. I will drop an update here once it is available. Meanwhile, you can find all extension functions [here](https://github.com/sagar-viradiya/AndroidPhysicsAnimation/blob/master/app/src/main/java/com/example/sagar/physicsanimation/utils/PhysicsAnimationExtensions.kt) in my github repo. I would like to thank [Jake Wharton](https://medium.com/u/8ddd94878165) for suggesting improvements on my initial extension functions.

> Update : All the extensions in this post are now part of Android KTX for dynamic animation. Currently(While updating this note) it’s in alpha 01. Keep an eye on release notes [here](https://developer.android.com/jetpack/androidx/releases/) to keep track of future releases.

I would like to know your thoughts on extension functions we saw. If you have any suggestion or correction to make this extension functions better feel free to drop a comment below or reach out to me on [twitter](https://twitter.com/viradiya_sagar).

That’s it for now folks :)

*Thanks to [Doris Liu](https://medium.com/u/c6093e8a8d5b) and [Nick Butcher](https://medium.com/u/22c02a30ae04) for proofreading.*

> *This was originally posted on [Medium](https://medium.com/proandroiddev/sugar-koating-physics-based-animation-api-263d43a7d312)*



