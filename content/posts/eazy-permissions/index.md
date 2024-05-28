---
title: "Eazy permissions"
summary: An introduction to the runtime permission library.
date: 2019-06-17
weight: 1
tags: ["Runtime Permissions", "Kotlin", "Kotlin Coroutines", "LiveData", "Android"]
cover:
  image: images/header.webp
  
  caption: "Shot on my Pixel 3a XL"
  hiddenInList: false
---

Hey folks, good to see you back after a long time 👋. I have been working on the Android library called [`eazypermissions`](https://github.com/sagar-viradiya/eazypermissions) for past one month which allows you to request runtime permission with coroutines and LiveData support. In this post, we will see why I created this library and a basic introduction to the library.

## The motto 🎯

In this year’s Google I/O there were lots of announcements regarding coroutines support for Jetpack libraries. Also since coroutines became stable in [Kotlin 1.3.0](https://blog.jetbrains.com/kotlin/2018/10/kotlin-1-3/) there were also lots of buzz about it in community.

The community is slowly adapting coroutines and it's here to stay for sure. The fact that you can write asynchronous code in a sequential way makes your code clean and concise.

Oftentimes we consume the third party library APIs in our coroutines and those APIs were not designed having coroutines in mind. But things are changing and some popular third-party libraries are started supporting coroutines e.g. Retrofit 2.6.0 now has `suspend` support.

{{< twitter user="adityatelange" id="1136339670302973952" >}}

Runtime permission is callback based API wherein you request for permission and listen for the result. Wouldn’t it be awesome if you don’t need to write boilerplate callback code for the result and consume it in a sequential way within coroutines? Well, this is exactly why I wrote this library focusing on clean and concise API that you can easily consume within coroutines (No callbacks yay 🎉). We will see how in a bit.

Coming to LiveData support, Architecture Components has been around for quite some time. Observing runtime permission results through LiveData could be one of the interesting use-case. I thought it would be nice to have runtime permission API around LiveData. The library will expose LiveData of result and consumer can observe and decide what to do upon the result. We will see how in a bit.

## Introduction to Eazy Permissions

Based on your need (coroutines support or LiveData support) you can include the library in your project as shown below.

```groovy
//For coroutines
implementation 'com.sagar:coroutinespermission:1.0.0'
//For LiveData
implementation 'com.sagar:livedatapermission:1.0.0'
```

### Coroutines support

Requesting permission is just a simple function call to suspending function **requestPermissions** of [`PermissionManager`](https://github.com/sagar-viradiya/eazypermissions/blob/master/coroutinespermission/src/main/java/com/eazypermissions/coroutinespermission/PermissionManager.kt) from your coroutines or other suspending function which will return [`PermissionResult`](https://github.com/sagar-viradiya/eazypermissions/blob/master/common/src/main/java/com/eazypermissions/common/model/PermissionResult.kt). It takes 3 parameters.

1. An instance of AppCompactActivity or Fragment depending on from where you are requesting permission.
2. Request id.
3. varargs of permission you want to request.

Here is the signature of `requestPermissions` method.

{{< gist sagar-viradiya 9ffea9ba6b14aaafa5595318a08709a9 CoroutinesPermissionManager.kt >}}

This is how you would request for permission within coroutines and get result sequentially.

{{< gist sagar-viradiya d01c20910e9d78fc2398c44d52772dd5 Fragment.kt >}}

You can request permission from coroutines launched using any dispatcher (IO/Default/Main).

Library exposes [`PermissionResult`](https://github.com/sagar-viradiya/eazypermissions/blob/master/common/src/main/java/com/eazypermissions/common/model/PermissionResult.kt) as a result of permission request which is nothing but simple sealed class which wraps all possible outcomes.

{{< gist sagar-viradiya 69928654f40774efdfeb4320b7bac5f1 PermissionResult.kt >}}

### LiveData support

Just in case of coroutines we saw above requesting permission is just a simple method call to [`PermissionManager`](https://github.com/sagar-viradiya/eazypermissions/blob/master/livedatapermission/src/main/java/com/eazypermissions/livedatapermission/PermissionManager.kt) from your Activity/Fragment. It takes 3 parameters.

1. An instance of AppCompactActivity or Fragment depending from where you are requesting permission.
2. Request id.
3. varargs of permission you want to request.

Here is the signature of `requestPermissions` method.

{{< gist sagar-viradiya 1a9df6b97dc52ffbbe7ed18816a8fcb2 LiveDataPermissionManager.kt >}}

This is how you would request permissions from your Activity/Fragment.

{{< gist sagar-viradiya b83b2f02d65f20978b1df93b83a7882f Fragment.kt >}}

#### Observing result

With just one simple step (implementing an interface) you are ready to observe the result of the request. Your Activity/Fragment must implement `**setupObserver**` method of [`PermissionObserver`](https://github.com/sagar-viradiya/eazypermissions/blob/a69d7d221e60f3d431b60fcc911aef69514ac04b/livedatapermission/src/main/java/com/eazypermissions/livedatapermission/PermissionManager.kt#L141) interface which exposes LiveData<[`PermissionResult`](https://github.com/sagar-viradiya/eazypermissions/blob/master/common/src/main/java/com/eazypermissions/common/model/PermissionResult.kt)>. Here is the definition of interface.

{{< gist sagar-viradiya d8c4b603f67506aa6250913185bd657b PermissionObserver.kt >}}

The library will only call `setupObserver` method when you are requesting permission for the first time. All the successive call to requestPermissions method will use the same observer.

Just as you would observe other LiveData you can observe LiveData<[`PermissionResult`](https://github.com/sagar-viradiya/eazypermissions/blob/master/common/src/main/java/com/eazypermissions/common/model/PermissionResult.kt)> as shown below.

{{< gist sagar-viradiya ffe87f2013ac187f0d593e03f38b2a6d PermissionObserver.kt >}}

That’s it, that’s all I had to share with you folks about this library. I will be maintaining this library regularly and I am hoping for your suggestions and contributions to improve this library. Head over to the GitHub repo below for more details about library.

[Eazy permission](https://github.com/sagar-viradiya/eazypermissions)

Do give it a shot and let me know your feedback in comments below or you can reach out to me on [Twitter](https://twitter.com/viradiya_sagar).

That’s it, for now, folks 🙂 Until next time 👋

Happy coding 😃

*Thanks, [Nate Ebel](https://medium.com/u/696aff678e4) and [Nemi Shah](https://medium.com/u/8970f5acadd8) for proofreading this.*

> *This was originally posted on [Medium](https://medium.com/proandroiddev/eazy-permissions-c574809bd682)*

