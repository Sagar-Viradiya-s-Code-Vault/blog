---
title: "Callback-ktx"
summary: Introduction to callback-ktx library
date: 2021-10-05
weight: 1
tags: ["Coroutines", "Callback"]
cover:
  image: images/header.jpg 
  caption: "Photo by [CJ Dayrit](https://unsplash.com/@cjred?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/photos/woman-walking-through-downstairs-xX2aYSBsyKo?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)"
  hiddenInList: false
---

This blog is about the [**library**](https://github.com/sagar-viradiya/callback-ktx) that came into existence after reading [Chris Banes](https://chrisbanes.me/) blog series on [Suspending over Views](https://chris.banes.dev/suspending-views/). The intention behind this blog is to throw some light on this library and get feedback from the community. So enjoy this short one! :)

Android ❤️s callback. All framework and jetpack libraries APIs uses it to notify events, but it becomes messy when you have nested callbacks. It’s difficult to understand and scale. Also, the chances of bugs are very high when you’re dealing with nested callbacks.

The good thing is we can easily avoid it through Kotlin’s coroutine APIs. [callback-ktx](https://github.com/sagar-viradiya/callback-ktx) is an attempt to wrap the potential framework and jetpack callback-based APIs into suspending extension functions. In case of multiple callbacks, it exposes `Flow` to observe all callbacks.

It would be kinda redundant if I explain all the benefits you will get by wrapping callbacks into suspending functions as Chris Banes blog series already covered them nicely. If you haven’t read it I would suggest reading it before continuing this one.

It is not difficult at all to wrap callback-based APIs into suspending function. What was challenging though was to go through all framework and jetpack APIs and figure out which API could be benefited. Let’s see what this library is capable of as of now.

## What is it?
Currently, the library covers the following APIs from framework and jetpack.

- Animation
- Location
- RecyclerView
- Sensor
- View
- Widget(TextView)

Let’s see one example of observing location updates.

To get the last location all you need to do is to call awaitLastLocation which is an extension function over [`FusedLocationProviderClient`](https://developers.google.com/android/reference/com/google/android/gms/location/FusedLocationProviderClient)

```Kotlin
viewLifecycleOwner.lifecycleScope.launch {
  val location = fusedLocationProviderClient.awaitLastLocation() // Suspend coroutine
  // Use location
}
```

To observe location update it exposes flow of location.

```Kotlin
viewLifecycleOwner.lifecycleScope.launch {
  fusedLocationProviderClient.locationFlow(locationRequest, lifecycleOwner).collect { location ->
    // Consume location
  }
}
```

Note that extension takes lifecycle owner along with location request. This makes flow lifecycle aware so it won’t fire location updates while the app is in the background. You can visit the [library](https://github.com/sagar-viradiya/callback-ktx) on GitHub to see more examples.

There are two essential things while wrapping callback into suspending function to avoid potential memory leak and unwanted resource use by coroutines

1. If coroutine gets cancelled for some reason, the callback needs to be unregistered.
2. If an async operation for which callback is expected gets cancelled, the coroutine needs to be terminated.

The library takes care of both of these. Also, Callback extensions are divided across different modules based on the category they fall under. For example, all framework APIs would fall under the core module. Anything not related to the framework is in its separate module. So depending on the need user can depend on a specific maven artifact.

## What’s next?
As I mentioned earlier, the big challenge while working on this library was to figure out all APIs that could be benefited. I would love to see contributions from the community in terms of filing issues for new extensions, bug fixes and optimisations. You can find repo [here](https://github.com/sagar-viradiya/callback-ktx?source=post_page-----64f57b9fc6cc--------------------------------)


Until next time 👋🏻

> *This was originally posted on [Google Developer Expert Medium publication](https://medium.com/google-developer-experts/callback-ktx-64f57b9fc6cc)*