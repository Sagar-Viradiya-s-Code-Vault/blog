---
title: "Android Jetpack - NavigationUI"
summary: Attaching jetpack navigation to Material components
date: 2018-07-09
weight: 1
tags: ["Kotlin", "Android", "Jetpack", "Navigation"]
cover:
  image: images/header.webp 
  caption: "[Photo by rawpixel.com from Pexels](https://www.pexels.com/photo/adult-book-business-cactus-297755/)"
  hiddenInList: false
---

You might be wondering why another blog post on Navigation Component while there are many out there ? Well I promise you this is different and requires basic knowledge on navigation, so if you are not familiar with navigation component I would recommend going through following series of blog posts where 
[Dario Mungoi](https://medium.com/u/9d9c86920a50) explained it very well. Those who already have some idea about navigation component can continue…

[Android Navigation Component - Part 1](https://medium.com/google-developer-experts/android-navigation-components-part-1-236b2a479d44)

[Android Navigation Component - Part 2](https://medium.com/google-developer-experts/android-navigation-components-part-2-ca643eb301e3)

[Android Navigation Component - Part 3](https://medium.com/google-developer-experts/android-navigation-components-part-2-ca643eb301e3)

With navigation component you can reduce boilerplate code for fragment transactions(and avoid possible bug if you don’t do it properly) and starting activity. All you need to do is to create navigation graph for your app(Please refer [part 1](https://medium.com/google-developer-experts/android-navigation-components-part-1-236b2a479d44) of above series to know how to create navigation graph) and call navigate method of [NavController](https://developer.android.com/reference/androidx/navigation/NavController) by passing Destination ID or Action ID on user action(like button press) and it will take care of either fragment transaction or starting activity depending on your destination.

NavigationUI library further simplify your app’s navigation by hooking [NavigationView](https://developer.android.com/reference/android/support/design/widget/NavigationView) and [BottomNavigationView](https://developer.android.com/reference/android/support/design/widget/BottomNavigationView) material components to navigation graph. This means it will map destinations to items within them and on selection of an item it will open destination associated with item.

> It is also possible to hook your custom view to navigation graph. Please refer [part 3](https://medium.com/google-developer-experts/android-navigation-components-part-3-19554ec9ae83) of above series for that.

Let’s see how you can hook NavigationView and BottomNavigationView to navigation graph. I have created [sample app](https://github.com/sagar-viradiya/NavigationUIDemo) to demonstrate this. Before we dive into implementation it is important to understand UI structure of an app that I am going to use as example. Below are the screenshots of app so that you get idea about app UI structure. At high level we have two screens home having BottomNavigationView and info screen. You can navigate between them through NavigationView. Within home screen we have 3 sub-screens and you can navigate between them through BottomNavigationView.

|{{< figure src="images/app1.webp" >}}|{{< figure src="images/app2.webp" >}}|{{< figure src="images/app3.webp" >}}|
|--------------|--------------|--------------|

To get started add following dependencies in your build.gradle file

```groovy
implementation "android.arch.navigation:navigation-fragment-ktx:1.0.0-alpha02"
implementation "android.arch.navigation:navigation-ui-ktx:1.0.0-alpha02"
```

> Note the **ktx** suffix in both dependencies. Both are kotlin extension wrapper around actual **android.arch.navigation:navigation-fragment** and **android.arch.navigation:navigation-ui** dependency and internally depends on these two dependencies. If you are using java then use dependencies without ktx.

## Hooking NavigationView to navigation graph

Below is the navigation graph that we want to attach to our NavigationView. As you can see it has two fragments one for home screen and one for info screen and our start destination is home screen.

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/nav_main"
    app:startDestination="@id/bottomNavFragment">

    <fragment
        android:id="@+id/bottomNavFragment"
        android:name="com.example.sagar.navigationuidemo.BottomNavFragment"
        android:label="Home"
        tools:layout="@layout/fragment_bottom_nav" />

    <fragment
        android:id="@+id/infoFragment"
        android:name="com.example.sagar.navigationuidemo.InfoFragment"
        android:label="Info"
        tools:layout="@layout/fragment_info" />
</navigation>
```

To attach above graph simply get [NavController](https://developer.android.com/reference/androidx/navigation/NavController) associated with [NavHostFragment](https://developer.android.com/reference/androidx/navigation/fragment/NavHostFragment) which is essentially container fragment hosting home and info fragment and call extension function `setupWithNavController` of NavigationView which takes [NavController](https://developer.android.com/reference/androidx/navigation/NavController) and internally calls static method [`setupWithNavController`](https://developer.android.com/reference/androidx/navigation/ui/NavigationUI.html#setupWithNavController(android.support.design.widget.NavigationView,%20androidx.navigation.NavController)) of NavigationUI.

```kotlin
val navController = Navigation.findNavController(this, R.id.mainNavHostFragment)
navigationView.setupWithNavController(navController)
```

But how do NavigationUI figure out mapping between menu item of NavigationView and fragment destination ? 🤔

{{< iframe src="https://giphy.com/embed/3o84U6421OOWegpQhq" width="480" height="271" frameBorder="0" class="giphy-embed" >}}

Well nothing magical here, if you notice below in menu resource of NavigationView each item ID is matching with fragment ID in navigation graph. By this NavigationUI will figure out mapping between item and destination and it will perform fragment transaction on item selection.

```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:id="@+id/bottomNavFragment"
        android:title="@string/bottomnav_menu_title"/>
    <item
        android:id="@+id/infoFragment"
        android:title="@string/info_menu_title"/>
</menu>
```

## Hooking BottomNavigationView to navigation graph

Hooking BottomNavigationView is same as NavigationView. As you can see in screenshot above home fragment is the host of bottom navigation. So it will have another [NavHostFragment](https://developer.android.com/reference/androidx/navigation/fragment/NavHostFragment) within it hosting new navigation graph for bottom navigation. Below is the navigation graph for bottom navigation.

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/nav_secondary"
    app:startDestination="@id/bottomNavFragmentOne">

    <fragment
        android:id="@+id/bottomNavFragmentOne"
        android:name="com.example.sagar.navigationuidemo.BottomNavFragmentOne"
        tools:layout="@layout/fragment_bottom_nav_fragment_one"/>

    <fragment
        android:id="@+id/bottomNavFragmentTwo"
        android:name="com.example.sagar.navigationuidemo.BottomNavFragmentTwo"
        tools:layout="@layout/fragment_bottom_nav_fragment_two"/>

    <fragment
        android:id="@+id/bottomNavFragmentThree"
        android:name="com.example.sagar.navigationuidemo.BottomNavFragmentThree"
        tools:layout="@layout/fragment_bottom_nav_fragment_three"/>

</navigation>
```

To attach above graph call extension function setupWithNavController of BottomNavigationView as shown in code snippet below.

```kotlin
//Attach navigation graph after view creation of fragment
override fun onViewCreated(view: View, savedInstanceState: Bundle?){        
    val navController = Navigation.
    findNavController(requireActivity(), R.id.bottomNavHostFragment)            
    bottomNavigation.setupWithNavController(navController)
}
```

Menu for BottomNavigationView should also follow same rule as I mentioned in NavigationView.

```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">

    <item
        android:id="@+id/bottomNavFragmentOne"
        android:icon="@drawable/ic_home_black_24dp"
        android:title="@string/home_bottom_nav_item_title"/>
    <item
        android:id="@+id/bottomNavFragmentTwo"
        android:icon="@drawable/ic_exposure_plus_1_black_24dp"
        android:title="@string/screen_bottom_nav_title"/>
    <item
        android:id="@+id/bottomNavFragmentThree"
        android:icon="@drawable/ic_exposure_plus_2_black_24dp"
        android:title="@string/screen_bottom_nav_title"/>

</menu>
```

That’s it, we are done with hooking NavigationView and BottomNavigationView to navigation graph.

## Conclusion

Navigation component makes our life easy by handling fragment transaction and attaching UI components to navigation graph. This will reduce boilerplate code we write to wire up user actions with UI navigation. I recently refactored one of my [side project](https://github.com/sagar-viradiya/AndroidPhysicsAnimation) to use navigation component and my MainActivity went down from 112 lines of code to 34 as you can see below.

{{< gist sagar-viradiya 38529ce9401e09829da5ed3ec01d857e MainActivity.kt >}}

{{< gist sagar-viradiya 85ed7b000b9e5a79815659552b0ebbf6 MainActivity.kt >}}

Overall [Jetpack](https://developer.android.com/jetpack/) makes android development easy and it helps you to focus more on your app’s business logic rather than spending more time on framework specific implementations. That’s all I have for this blog post.

Find the [source code](https://github.com/sagar-viradiya/NavigationUIDemo) of sample app on github.

I would like to know your thoughts on navigation component. If you have any questions or suggestions please feel free to comment below or reach out to me on [twitter](https://twitter.com/viradiya_sagar).

That’s it for now folks :)

*Thanks [Ian Lake](https://medium.com/u/51a4f24f5367) for proofreading this*

> *This was originally posted on [Medium](https://medium.com/proandroiddev/android-jetpack-navigationui-a7c9f17c510e)*