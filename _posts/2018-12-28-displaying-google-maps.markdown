---
layout: post
title:  "Displaying Google Maps"
date:   2018-12-28 20:00:00 +0100
categories: jekyll update
---

I will explain in this article how I display Google Maps, so that as little code as possible is in the `Fragment`.

In this article I assume that you use [Android Jetpack][jetpack] for [Navigation][navigation], therefore you put all of your own graphical components in `Fragment`s, as opposed to `Activity`s. In case you do not know how to set it up, adding [Navigation][navigation] is explained [here][navigation-article], in this blog.

In the article I will use the example of my project [Wiktor-Navigator][navigator].

## Adding the Fragment

The following snippet comes from the file [fragment_navigation.xml][fragment-navigation]:

{% highlight xml %}

<fragment
        android:id="@+id/map"
        android:name="com.google.android.gms.maps.SupportMapFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".navigation.NavigationFragment"/>

{% endhighlight %}

Instead of using Google Maps directly in its `Fragment` class [NavigationFragment][navigation-fragment], I will use it it in its own class, [MapHolder][map-holder].

Here is the line I use to keep a reference to the instance of `MapHolder` inside its `Fragment`:

{% highlight kotlin %}

private lateinit var mapHolder: MapHolder

{% endhighlight %}

Here is the code I use to create the instance of `MapHolder` in the method `onViewCreated()`:

{% highlight kotlin %}

mapHolder = createMapHolder {
            init(savedInstanceState, root, navigatorModel.contact.value)
            checkLocationPermission = this@NavigationFragment::ifLocationPermissionGranted
            persistCameraPositionAndZoom = this@NavigationFragment::persistCameraPositionAndZoom
        }

{% endhighlight %}

What is notable is that inside of the lambda it passes to the `MapHolder` to function from the `Fragment` that will be called later, so that the `MapHolder` doesn't have to hold references to the whole `Fragment`.

The code creates an instance of `MapHolder` and immediately initialize it by calling the lambda expression on it. Here is the global function that that creates the instance:

{% highlight kotlin %}

fun Fragment.createMapHolder(f: MapHolder.() -> Unit): MapHolder = MapHolder().apply(f).also {
    val mapFragment = childFragmentManager.findFragmentById(R.id.map) as SupportMapFragment
    mapFragment.getMapAsync { map -> it withMap map }
}

{% endhighlight %}

It calls `MapHolder` construction and calls on it the lambda expression passed to it.

The difference between functions `apply` and `also` in the above code is such that when `apply` is called, it understands `this` as an instance of `MapHolder` just created, so that the lamba expression `f` is executed on `MapHolder`. (That's why the type of this lambda is written as `MapHolder.() -> Unit`). The function `also` sees `this` as an instance of `Fragment` (the `Fragment` that the function `createMapHolder` is called on), and it sees `it` as the instance of `MapHolder`.

In other words, the function `createMapHolder` creates an instance of `MapHolder`, calls the function `f` on it, then calls some code on the `Fragment` it was called from.

Inside of the `also` function it has access to `childFragment` from inside the class `NavigationFragment`. It uses it to obtain an instance of `SupportMapFragment` defined in [fragment_navigation.xml][fragment-navigation]. It then asychronoulsy gets an instance of `GoogleMap` and then uses that insnance to set it to `MapHolder` my calling the `infix` function:

{% highlight kotlin %}

infix fun withMap(map: GoogleMap) = map.onReady()

{% endhighlight %}

In turn it calls an extension fuction of `GoogleMaps` that itself has a couple of local functions (functions inside a function):

{% highlight kotlin %}

@SuppressLint("MissingPermission")
private fun GoogleMap.onReady() {
    infix fun LatLng.isDifferentFrom(other: LatLng?) = if (other == null) true
    else Math.abs(latitude - other.latitude) > FLOAT_TOLERANCE ||
            Math.abs(longitude - other.longitude) > FLOAT_TOLERANCE

    fun onCameraIdle() = map!!.cameraPosition.let {
        zoom = it.zoom
        lastCameraPosition = it.target
        if (lastCameraPosition isDifferentFrom peerLocation) {
            moveCamera = ::moveCameraToLast
        }
        persistCameraPositionAndZoom()
    }

    this@MapHolder.map = apply {
        setOnCameraIdleListener { onCameraIdle() }
        checkLocationPermission { isMyLocationEnabled = true }
    }
    peerLocation?.putPeerMarker()
    moveCamera()
    moveCamera = this@MapHolder::moveCameraToPeerOrLastLocation
}

{% endhighlight %}

Local functions are used above to avoid using too many `private` fuctions inside the class. I always chose to use local function instead of `private` functions that are called by one funnction only.

Please note that this function uses the annotation `@SuppressLint("MissingPermission")`. Other code in the projects assure that it will only be called **after** this permission has been already granted. If you would like to find out more about the way permissions are handled in this project, consider reading a [dedicated article][permissions] I included in this blog.

Please note also the usage of the expression `this@MapHolder` inside this code block. When `this` is written this way, it indicates `this` form the instance of `MapHolder` class. Because it is inside of an extension function, all other usages of `this` indicate an instance of `GoogleMaps`, for example this line:

{% highlight kotlin %}

this@MapHolder.map = apply {

{% endhighlight %}

means that the first `this` is an instance of `MapHolder`, and that `apply` is called on another `this` (here not written) that the second time indicates an instance of `GoogleMaps`.

## Overlays

Because in this project map is used to display informaction about the contact (one of the contacts from the list of friends stored in the application's data), I chose to include the code that display overlays regarding contacts inside of `MapHolder` class. Please note that inside of the `Fragment` class they would have been just as valid.

{% highlight kotlin %}

private fun updateHud() = view.hud_following.run {
        alpha = 1.0f
	...
    }

{% endhighlight %}

The field `view` in the above code refers to the `ViewGroup` that is the root view of the `Fragment` this `MapHolder` is in. It is set right in the initialization phase that was already mentioned above in this article, here repeated for reference:

{% highlight kotlin %}

mapHolder = createMapHolder {
            init(savedInstanceState, root, navigatorModel.contact.value)
            checkLocationPermission = this@NavigationFragment::ifLocationPermissionGranted
            persistCameraPositionAndZoom = this@NavigationFragment::persistCameraPositionAndZoom
        }

{% endhighlight %}

The field `alpha` (as in the code `alpha = 1.0f`) is the alpha value field of `hud_following`, which is a view insite of the root view of the `Fragment`.

## Camera movement

Because the instance of `GoogleMaps` is held inside of `MapHolder` instead of directly in the `Fragment`, the code for moving the camera should be also inside of `MapHolder`, so that there are not too many lines of code in the `Fragment` itself:

{% highlight kotlin %}

private fun LatLng.moveCamera() {
    val cameraPosition = CameraPosition.Builder().target(this).zoom(zoom).build()
    if (animateCamera) {
        map?.animateCamera(CameraUpdateFactory.newCameraPosition(cameraPosition))
    } else {
        map?.moveCamera(CameraUpdateFactory.newCameraPosition(cameraPosition))
    }
    animateCamera = false
}

{% endhighlight %}

## Conclusion

At the time of wrining this article, the code of the class `MapHolder` is 214 lines long, while the code of `NavigationFragment` is 313 lines.

Before I created this pattern I would just stick all code for handling the map inside of one `Fragment` (or `Activity`), so that here it would create one class that would be 500+ lines of code long.

I was inspired to pick this name by the class [RecyclerView.ViewHolder][view-holder], and my map-holding class was really called `ViewHolder` once, before I decided to settle on the name `MapHolder`.

I guess that either solution is correct, whether you want to keep the code haldling Google Maps inside of the Fragment, or inside of its own class. I chose the latter in order to promote [separation of concerns][separation-of-concerns].

This design patterns could be improved upon by including only initialization and camera handling code inside this class, and all other usage-specific code, like displaying overlays, inside classes that extend it, but because I show only one map in this project, and it is always used to display information in the overlay, I chose not do do so.

I hope that by thoroughly explaining my design pattern in this article I helped the reader in case they want to display a map that serves their own particular use case, but without necessarily including all the code inside the `Fragment`.

## Donations

If you've enjoyed this article, consider donating some bitcoin: bc1qncxh5xs6erq6w4qz3a7xl7f50agrgn3w58dsfp (you can also look at [donations page][donations]).

[jetpack]: https://developer.android.com/jetpack/
[navigation]: https://developer.android.com/topic/libraries/architecture/navigation.html
[navigation-article]: https://syrop.github.io/jekyll/update/2018/12/26/architecture-components-and-searching.html
[navigator]: https://github.com/syrop/Wiktor-Navigator
[fragment-navigation]: https://github.com/syrop/Wiktor-Navigator/blob/master/navigator/src/main/res/layout/fragment_navigation.xml
[navigation-fragment]: https://github.com/syrop/Wiktor-Navigator/blob/master/navigator/src/main/kotlin/pl/org/seva/navigator/navigation/NavigationFragment.kt
[map-holder]: https://github.com/syrop/Wiktor-Navigator/blob/master/navigator/src/main/kotlin/pl/org/seva/navigator/navigation/MapHolder.kt
[permissions]: https://syrop.github.io/jekyll/update/2018/12/26/permissions-rxjava-and-lifecycle.html
[donations]: https://syrop.github.io/donate/
[view-holder]: https://developer.android.com/reference/android/support/v7/widget/RecyclerView.ViewHolder
[separation-of-concerns]: https://en.wikipedia.org/wiki/Separation_of_concerns
