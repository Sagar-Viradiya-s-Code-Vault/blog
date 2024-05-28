---
title: "Kotlin DSL for runtime permissions"
summary: A kotlin syntactic sugar on runtime permissions
date: 2019-10-07
weight: 1
tags: ["Android", "Kotlin", "Runtime Permissions"]
cover:
  image: images/header.webp
  
  caption: "Shot on my Pixel 3a XL"
  hiddenInList: false
---

A while back I open-sourced an Android library [**eazypermissions**](https://github.com/sagar-viradiya/eazypermissions) for runtime permissions which allows you to request permission within coroutines (no callback yay 🎉) and has support for LiveData as well. If you are interested you can read more about it 👇

[Eazy permissions]({{< ref "/content/posts/eazy-permissions/index.md" >}})

The library now offers a Kotlin DSL to request permissions. In this post, we will discuss the DSL API and how to request permissions using the DSL. Let’s get started.

## Kotlin DSL for requesting permissions

While requesting permissions three things that matter are permissions, the request code, and the result. The DSL API focuses on these three things and allows you to request permissions cleanly and concisely. Let’s see how.

If you are requesting permission in Activity/Fragment you can request permissions as shown below.

```kotlin
requestPermissions(
    Manifest.permission.ACCESS_FINE_LOCATION,
    Manifest.permission.READ_CONTACTS,
    Manifest.permission.CAMERA
) {
    requestCode = 4
    resultCallback = {
        when (this) {
            is PermissionResult.PermissionGranted -> {
                Toast.makeText(requireContext(), "Granted!", Toast.LENGTH_LONG).show()
            }
            is PermissionResult.PermissionDenied -> {
                Toast.makeText(requireContext(), "Denied", Toast.LENGTH_SHORT).show() 
            }
            is PermissionResult.ShowRational -> {
                Toast.makeText(requireContext(), "Rational", Toast.LENGTH_SHORT).show()
            }
            is PermissionResult.PermissionDeniedPermanently -> {
                Toast.makeText(requireContext(), "Denied permanently", Toast.LENGTH_SHORT).show()
            }
        }
    }
}
```

Now let’s understand the above DSL.

The [`requestPermissions`](https://github.com/sagar-viradiya/eazypermissions/blob/8e2bdc0b3905cc9f09df5160ada30ca4eb424e70/dslpermission/src/main/java/com/eazypermissions/dsl/extension/Extensions.kt#L28) function is an extension function on Activity and Fragment so you can call it directly from activity and fragment. It takes `vararg` of permissions that you want to request and lambda with a receiver on PermissionRequest. Since it is a lambda with a receiver on [`PermissionRequest`](https://github.com/sagar-viradiya/eazypermissions/blob/master/dslpermission/src/main/java/com/eazypermissions/dsl/model/PermissionRequest.kt) you have direct access to members of [`PermissionRequest`](https://github.com/sagar-viradiya/eazypermissions/blob/master/dslpermission/src/main/java/com/eazypermissions/dsl/model/PermissionRequest.kt). Within lambda, you initialize two members of [`PermissionRequest`](https://github.com/sagar-viradiya/eazypermissions/blob/master/dslpermission/src/main/java/com/eazypermissions/dsl/model/PermissionRequest.kt), `requestCode` and `resultCallback`.
Here is the definition of [`PermissionRequest`](https://github.com/sagar-viradiya/eazypermissions/blob/master/dslpermission/src/main/java/com/eazypermissions/dsl/model/PermissionRequest.kt) class.

```kotlin
/**
* Represents permission request encapsulating [requestCode] and
* [resultCallback] for a permission request.
*/
class PermissionRequest (
    var requestcode: Int? = null,
    var resultcallback: (PermissionResult.() -> Unit)? = null
)
```

`resultCallback` is again a lambda with a receiver on [`PermissionResult`](https://github.com/sagar-viradiya/eazypermissions/blob/master/common/src/main/java/com/eazypermissions/common/model/PermissionResult.kt) and the library will invoke this callback for a result. Inside the lambda, you can directly refer permission result as `this`. [`PermissionResult`](https://github.com/sagar-viradiya/eazypermissions/blob/master/common/src/main/java/com/eazypermissions/common/model/PermissionResult.kt) is nothing but the simple sealed class which wraps all possible outcomes i.e Permission granted, denied, denied permanently and show rational.

Here are the signatures of `requestPermissions` extension functions on Activity and Fragment.

```kotlin
/**
* @param permissions vararg of all the permissions for request.
* @param requestBlock block constructing [PermissionRequest] object for permission request.
*/
inline fun AppCompatActivity.requestPermissions(
    vararg permissions: String,
    requestBlock: PermissionRequest.() -> Unit
)

/**
* @param permissions vararg of all the permissions for request.
* @param requestBlock block constructing [PermissionRequest] object for permission request.
*/
inline fun Fragment.requestPermissions(
    vararg permissions: String,
    requestBlock: PermissionRequest.() -> Unit
)
```

If you are requesting permission outside your Fragment/Activity you can request permissions as shown below.

```kotlin
PermissionManager.requestPermissions( 
    fragment, //Instance of Fragment or Activity
    Manifest.permission.ACCESS_FINE_LOCATION,
    Manifest.permission.READ_CONTACTS,
    Manifest.permission.CAMERA
) {
    requestCode = 4
    resultCallback = {
        when (this) {
            is PermissionResult.PermissionGranted -> {
                Toast.makeText(requireContext(), "Granted!", Toast.LENGTH_LONG).show()
            }
            is PermissionResult.PermissionDenied -> {
                Toast.makeText(requireContext(), "Denied", Toast.LENGTH_ SHORT).show()
            }
            is PermissionResult.ShowRational -> {
                Toast.makeText(requireContext(), "Rational", Toast.LENGTH_SHORT).show()
            }
            is PermissionResult.PermissionDeniedPermanently -> {
                Toast.makeText(requireContext(), "Denied permanently", Toast.LENGTH_SHORT).show()
            }
        }
    }
}
```

The only difference here is you need to call `PermissionManager.requestPermissions` which takes an instance of Fragment/Activity as an additional parameter. The extension function we saw previously internally delegates call to this function.

Here are the signatures of `PermissionManager.requestPermissions` functions.

```kotlin
/**
* A static factory inline method to request permission for activity.
*
* @param activity an instance of [AppCompatActivity]
* @param permissions vararg of all permissions for request
* @param requestBlock [PermissionRequest] block for permission request
*
*/
inline fun requestPermissions(
    activity: AppCompatActivity,
    vararg permissions: String,
    requestBlock: PermissionRequest.() -> Unit
)

/**
* A static factory inline method to request permission for fragment.
*
* @param fragment an instance of [Fragment]
* @param permissions vararg of all permissions for request
* @param requestBlock [PermissionRequest] block for permission request
*
*/
inline fun requestPermissions(
    fragment: Fragment,
    vararg permissions: String,
    requestBlock: PermissionRequest.() -> Unit
)
```

How you include the library in your project.

```groovy
implementation 'com.sagar:dslpermission:2.0.0'
```

That’s all I have for this new Kotlin DSL feature. Head over to the GitHub repo below for more details about the library.

https://github.com/sagar-viradiya/eazypermissions

If you have any suggestions please feel free to comment below or you can reach out to me on [Twitter](https://twitter.com/viradiya_sagar).

That’s it, for now, folks 🙂 Until next time 👋

Happy coding 😃

> *This was originally posted on [Medium](https://medium.com/android-news/kotlin-dsl-for-runtime-permissions-ba04dbe0de2c)*