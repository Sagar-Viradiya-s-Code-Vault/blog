---
title: "Talks"
tags: ["talks"]
ShowWordCount: false
ShowReadingTime: false
UseHugoToc: false
showtoc: false
---

## Dissecting Compose Animation (Not recorded)

Animation plays a crucial role in App UI/UX. If not done correctly it can bite back leading to a bad app experience. A key to good animation is knowing API well and when to use which one.

While it is crucial to know animation APIs on the consumer side it is essential to know the internals of Compose animation to be able to debug or optimize something which requires internal knowledge.

This talk aims to throw some light on the internals of the following APIs

- Low-level animation API - Animation & AnimationSpecs
- animation*AsState API
- Transition API
- Animatable API
- AnimatedVisibility & AnimatedContent API
- Choreographing complex animation through Coroutines API

At the end, we will learn the internals of Compose Animation which will prepare you to optimize your App UI/UX experience! 

{{< figure src="images/dissecting_compose_animation.png" link="https://speakerdeck.com/sagarviradiya/dissecting-compose-animation" align=center target="_blank" >}}

## Uncoiling the COIL

Image loading is hard but luckily this problem has already been addressed on Android. There are many libraries out there that handle image loading seamlessly. COIL is the new kid to the club and you may be wondering why do we need another image loading library?
COIL is Kotlin first library backed by Kotlin Coroutines. It gives you an easy and concise API to deal with image loading. Oh and also it takes less space! In a nutshell, it's an easy to use, fast, lightweight, and modern library.
This talk will address:
- Why you should consider this library for image loading
- Understanding its API
- How it works. By covering its entire image loading pipeline.

By the end, you'll walk away with the advantages of using COIL and its image loading pipeline.

{{< youtube JCQTbKvcr90 >}}