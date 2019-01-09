---
layout: post
title:  "Selecting location (advanced topic)"
date:   2019-01-09 10:24:00 +0100
categories: jekyll update
---

This article will explain how to select a location by long-clicking its location on Google Maps, and how to get its address.

The code presented herein is being used in my GitHub project [Victor-Events].

I use the design pattern I called `MapHolder`, which I already introduced in the previous article, '[Displaying Google Maps][displaying-google-maps]'. What is the most interesting in the previous article is how I handle location permissions in `MapHolder`.

## Conventions in the code

I use functions called `with*()` they always return ether `Unit` or the instance of the class they are called on. They take only one parameter, and therefore can be `infix`. They are used for initialization, and are called when the first time the object that is used for the initialization becomes available. They always use the Kotlin function `with` (therefore their name). Example:

{% highlight kotlin %}

override fun withFragment(fragment: Fragment): MapHolder {
    super.withFragment(fragment)
    with (fragment) {
        viewModel = ViewModelProviders.of(activity!!).get(CreateEventViewModel::class.java)
    }
    return this
}

{% endhighlight %}

## Setting up

Navigation using [Android Jetpack][jetpack] is used in the code. If you need introduction to Jetpack's architecture components, please see [one of the previous articles][navigation-article] in this blog, which talks specifically about navigation using Jetpack.

This time, for brevity, I use this extension function to navigate to another `Fragment` using an `Int`:

{% highlight kotlin %}

fun Fragment.navigate(@IdRes resId: Int) = findNavController().navigate(resId)

{% endhighlight %}

I will explain in this article how I display Google Maps, so that as little code as possible is in the `Fragment`.

## Layouts

There is no difference in layout file between placing `MapHolder` and `InteractiveMapHolder`, as the difference is only in the code. Here is the generic code snippet you can use to put GoogleMap in your layout:

{% highlight xml %}

<FrameLayout
        android:id="@+id/map_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_below="@+id/address_layout"
        android:visibility="invisible">

        <fragment
            android:id="@+id/map"
            android:name="com.google.android.gms.maps.SupportMapFragment"
            android:layout_width="match_parent"
            android:layout_height="match_parent"/>

    </FrameLayout>

{% endhighlight %}

In the above snippet I put it inside of a `FrameLayout`, so that it can be made invisible when no location is available.

Because `MapHolder` is not interactive by default, its visibility status must be set externally. `InteractiveMapHolder` shall be always visible, because you can set a location in it by long clicking on the map.

## Creating an instance of MapHolder

The global function for creating an instance of `MapHolder` is such:

{% highlight kotlin %}

fun Fragment.createMapHolder(f: MapHolder.() -> Unit = {}): MapHolder =
        MapHolder().apply(f) withFragment this

{% endhighlight %}

It calls the constructor, calls the passed lambda expression on it, and eventually initializes it with an instance of the `Fragment`.

In the case of `MapHolder`, initialization with the `Fragment` simply means calling `getMapAsync()` on it:

{% highlight kotlin %}

open infix fun withFragment(fragment: Fragment): MapHolder {
    with (fragment) {
        val mapFragment = childFragmentManager.findFragmentById(R.id.map) as SupportMapFragment
        mapFragment.getMapAsync { map -> this@MapHolder withMap map }
    }
    return this
}

{% endhighlight %}

Before `MapHolder` can be initialized with an instance of the `Fragment`, the function `createMapHolder` calls on it any block of code that might have been passed to it as parameter. Here `createMapHolder` is called this way:

{% highlight kotlin %}

mapHolder = createMapHolder {
    checkLocationPermission = this@CreateEventFragment::checkLocationPermission
    onMapAvailable = {
        viewModel.observeLocation(this@CreateEventFragment, this@createMapHolder)
    }
}

{% endhighlight %}

Checking location permission in `MapHolder` has been explained in a [separate article][displaying-google-maps] in this blog, so it will be skipped here.

Apart from setting the handle to permissions-related code, callback is set that will be called when an instance of `GoogleMap` becomes available. It is used inside of `MapFragment` in the code:

{% highlight kotlin %}

@SuppressLint("MissingPermission")
protected open infix fun withMap(map: GoogleMap) {
    with (map) {
        this@MapHolder.map = this
        moveCamera(CameraUpdateFactory.newLatLngZoom(LatLng(DEFAULT_LAT, DEFAULT_LON), DEFAULT_ZOOM))
        onMapAvailable?.invoke(this)
        checkLocationPermission?.invoke {
            isMyLocationEnabled = true
        }
    }
}

{% endhighlight %}

Please note that because `onMapAvailable` is a lambda with receiver (it expects an extention function) that is set to null by default, the correct way of calling it **only if it is not null** is by using the operator `invoke`:

{% highlight kotlin %}

onMapAvailable?.invoke(this)

{% endhighlight %}

If you do not want to deal with nullable lambda expressions, you can just set it to an empty block by default:

{% highlight kotlin %}

var onMapAvailable: GoogleMap.() -> Unit = {}

{% endhighlight %}

And then call it like a regular extension function, without any parameter:

{% highlight kotlin %}

onMapAvailable()

{% endhighlight %}


## ViewModel

The `ViewModel` used both in `MapHolder` and `InteractiveMapHolder` (although not in the same way) is defined as such:

{% highlight kotlin %}

class CreateEventViewModel : ViewModel() {

    val time by lazy {
        MutableLiveData<LocalTime>()
    }

    val date by lazy {
        MutableLiveData<LocalDate>()
    }

    val location: MutableLiveData<EventLocation?> by lazy {
        MutableLiveData<EventLocation?>()
    }

    fun observeLocation(owner: LifecycleOwner, mapHolder: MapHolder) {
        location.observe(owner, Observer { mapHolder.putMarker(it?.location) })
    }
}

{% endhighlight %}

The first two fields, `time` and `date` are being used mostly do display time and date in text fields, so they will not be discussed here in this article.

The third field, `location`, is especially relevant to using an interactive map, so it is discussed it this section. It uses the data class:

{% highlight kotlin %}

data class EventLocation(val location: LatLng?, val address: String)

{% endhighlight %}

The `ViewModel` also contains a function that is used to observe the location and put a relevant marker on the map when location is available:

{% highlight kotlin %}

fun putMarker(latLng: LatLng?) {
    with (map!!) {
        clear()
        latLng ?: return
        addMarker(MarkerOptions().position(latLng))
                .setIcon(BitmapDescriptorFactory.defaultMarker(0f))
    }
}

{% endhighlight %}

It starts with the call `with(map!!)`, so that only one check has to be made whether `map` is null. Otherwise the IDE would complain that because `map` is both nullable and mutable, it has to be checked for `null` every time it is used.

I am sure this function will never be called when `map` is `null` anyway, because the listener is created (in the above function `observeLocation()`) in the callback that is run only when map is already available. Let me quote again the code snippet (already pasted above) that ensures this:

{% highlight kotlin %}

mapHolder = createMapHolder {
    ...
    onMapAvailable = {
        viewModel.observeLocation(this@CreateEventFragment, this@createMapHolder)
    }
}

{% endhighlight %}

The function `putMarker()` that is defined in `MapHolder` first unconditionally removes any prior markers from the map. This line deserves special explanation:

{% highlight kotlin %}

latLng ?: return

{% endhighlight %}

It ensures that `return` is called when `lanLng` is null, therefore I can be sure that it is **not** null further in the code block. (Because of Kotlin's Smart Casts I do not have to use the operator `!!` or `?` further in the code).

## Summary of MapHolder

`MapHolder` contains the code that initiates getting an instance of `GoogleMap` asynchronously and calls a callback on it (if the callback is set). It also handles location permission (if a lambda expression is set that handles the permission). It also moves the camera to the default hardcoded location and zoom.

It does not need to use a `ViewModel`, although a `ViewModel` can be used externally (here in the `Fragment`) to put a marker on the map.

Because in the project [Victor-Events][victor-events] I used the `MapHolder` design pattern specifically with the intention of making it extendable, and therefore allowing adding interaction to it in its subclasses, it is implemented in a slightly different way that it has beed described in the article '[Displaying Google Maps][displaying-google-maps]' in this blog, although you are encouraged to read the other article as well, especially if you want to find out more about location permission handling.

In the present implementation there is no permission handling in `InteractiveMapHolder` (it doesn't display the present location of the phone, but only the location set to it manually), so learting the article '[Displaying Google Maps][displaying-google-maps]' is probably the best way of learning about it.

## Creating an instance of InteractiveMapHolder

An instance of `InteractiveMapHolder` is created this way:

{% highlight kotlin %}

fun Fragment.createInteractiveMapHolder(f: InteractiveMapHolder.() -> Unit = {}): InteractiveMapHolder =
        (InteractiveMapHolder().apply(f) withFragment this) as InteractiveMapHolder

{% endhighlight %}

It looks almost exactly as the function `creativeMapHolder()` explained above in the section that explained creation of an instance of `MapHolder`.

The difference is that because `withFragment` in the superclass returns `MapHolder`, here its result needs to be cast to `InteractiveMapHolder`.

Here the overriden function looks this way:

{% highlight kotlin %}

override fun withFragment(fragment: Fragment): MapHolder {
    super.withFragment(fragment)
    with (fragment) {
        viewModel = ViewModelProviders.of(activity!!).get(CreateEventViewModel::class.java)
    }
    return this
}

{% endhighlight %}

Because instances of this type are meant to be interactive, here a `ViewModel` needs to be procured, to enable reporting the selection of a location back to the `Fragment` where this `InteractiveViewHolder` is located.

Other than procuring an instance of `ViewModel` this function just calls the `super`, which in turns initiates getting an instance of `GoogleMap` asynchronously:

{% highlight kotlin %}

open infix fun withFragment(fragment: Fragment): MapHolder {
    with (fragment) {
        val mapFragment = childFragmentManager.findFragmentById(R.id.map) as SupportMapFragment
        mapFragment.getMapAsync { map -> this@MapHolder withMap map }
    }
    return this
}

{% endhighlight %}

Please note that you do not have to repeat the keyword `infix` in the overriden function. In Kotlin functions that override an `infix` function are always also `infix`.

## Interactively setting the location

This is the implementation of the function `withMap()` that is called when an instance of `GoogleMap` has been obtained asynchronously:

{% highlight kotlin %}

override fun withMap(map: GoogleMap) {
    fun onMapLongClick(latLng: LatLng) {
        val address = try {
            with(geocoder.getFromLocation(
                    latLng.latitude,
                    latLng.longitude,
                    1)[0]) {
                    getAddressLine(maxAddressLineIndex)
            }
        } catch (ex: Exception) {
            DEFAULT_ADDRESS
        }
        viewModel.location.value = EventLocation(latLng, address)
    }

    super.withMap(map)
    with (map) {
        setOnMapLongClickListener { onMapLongClick(it) }
    }
}

{% endhighlight %}

I use `Geocoder` to obtain the string representation of the location (here: the last line of the address).

Obtaining an instance of `GeoCoder` requires passing an instance of `Context` in its constructor, so if I wanted I could create one once the `InteractiveMapHolder` is initiated with its parent `Fragment` in the function `withFragment`.

Instead of doing that I chose to use dependency injection (or dependency retrieval) with [Kodein][kodein]:

{% highlight kotlin %}

private val geocoder: Geocoder = instance()

{% endhighlight %}

If you want to read other articles in this blog about this Kotlin-specific dependency injection library, you can look at my documentation of the design patterns I use for [testing][testing] and [logging][logging] in my projects.

When configuring Kodein binding for `Geocoder` I use the following line of code:

{% highlight kotlin %}

bind<Geocoder>() with singleton { Geocoder(ctx, Locale.getDefault()) }

{% endhighlight %}

You can look up the whole configuration of my Kodein module in the class [KodeinModuleBuilder][kodeinmodulebuilder] by clicking the link in this paragraph, that will take you directly to this file in my GitHub repository of the [Victor-Events project][victor-events].

In code of `OnMapLongClickListener` I just catch all `Exception`s and set the address to default value (an empty string) if one is thrown.

If you want to see a more detailed list of exceptions that may be thrown when you obtain addresses with a `Geocoder`, you can look it up directly in [Google documentation][geocoder-google].

Please note that here I **do not** include the code that puts the location marker on the map inside of `OnMapLongClickListener`, although it would have also been correct to do so.

Instead, I let the parent `Fragment` to call the fuction `putMarker()` defined in `MapHolder`, that has already been discussed above in the present article.

## Showing location picker

An instance of `InteractiveMapHolder` has been used in the class [LocationPickerFragment][location-picker]. It extends the regular `androidx.fragment.app.Fragment`.

By making this choice I broke the convention used by `DatePickerFragment` and `TimePickerFragment` presented in [Google's documentation][pickers], which extend `androidx.fragment.app.DialogFragment`, and which I also use in [Victor-events project][victor-events].

By doing so I secured more real estate for displaying the map, as I prefer displaying maps in `Fragment`s that take up the whole screen, but you can easily adapt my [LocationPickerFragment][location-picker] to extend `DialogFragment`. Click the link in this paragraph to show its source on GitHub.

This is the way to display the `Fragment` using Android Jetpack:

{% highlight kotlin %}

fun showLocationPicker() =
        navigate(R.id.action_createEventFragment_to_locationPickerFragment)

{% endhighlight %}

This is the code of the `onViewCreated()` method that creates the `ViewModel`, sets the `Observer`s and presets the relevant pickers on request:

{% highlight kotlin %}

@SuppressLint("SetTextI18n")
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    fun showTimePicker() = TimePickerFragment().show(fragmentManager, TIME_PICKER_TAG)
    fun showDatePicker() = DatePickerFragment().show(fragmentManager, DATE_PICKER_TAG)
    fun showLocationPicker() = navigate(R.id.action_createEventFragment_to_locationPickerFragment)
    fun onTimeChanged(t: LocalTime) = time.setText("${t.hour}:${t.minute}")
    fun onDateChanged(d: LocalDate) = date.setText("${d.year}-${d.monthValue}-${d.dayOfMonth}")
    fun onLocationChanged(l: EventLocation?) {
        location.setText(l?.address ?: "")
        map_container.visibility = if (l == null) View.INVISIBLE else View.VISIBLE
    }

    viewModel = ViewModelProviders.of(activity!!).get(CreateEventViewModel::class.java)
    time.setOnClickListener { showTimePicker() }
    date.setOnClickListener { showDatePicker() }
    location.setOnClickListener { showLocationPicker() }
    viewModel.time.observe(this, Observer<LocalTime> { onTimeChanged(it) })
    viewModel.date.observe(this, Observer<LocalDate> { onDateChanged(it) })
    viewModel.location.observe(this, Observer { onLocationChanged(it) })

    ...
}

{% endhighlight %}

The above code is contained in the class [CreateEventFragment][create-event-fragment]. (The preceding link will take you to GitHub). It also uses an instance of `MapHolder` (one that is not interactive) discussed above in this article, as well as in [its own separate article][displaying-google-maps].

With the knowledge presented in both articles you should be able to display various instances of Google Maps in your projects, some of which will be static, some will respond to user imputs, some perhaps will respond to other events related to location, such as changes in location downloaded from the Internet or from devices connected to the phone.

## Mocking location


If you want to learn how to mock location in your projects, you can look at my 2016 project described it the article '[Testing with dependency injection][testing]'. (Please note, however, that that particular project does not use the `MapHolder` pattern).

[victor-events]: https://github.com/syrop/Victor-Events
[displaying-google-maps]: https://syrop.github.io/jekyll/update/2018/12/28/displaying-google-maps.html
[jetpack]: https://developer.android.com/jetpack/
[navigation-article]: https://syrop.github.io/jekyll/update/2018/12/26/architecture-components-and-searching.html
[kodein]: http://kodein.org/Kodein-DI/
[testing]: https://syrop.github.io/jekyll/update/2018/12/25/testing-with-dependency-injection.html
[logging]: https://syrop.github.io/jekyll/update/2018/12/30/logging.html
[geocoder-google]: https://developer.android.com/training/location/display-address
[kodeinmodulebuilder]: https://github.com/syrop/Victor-Events/blob/master/events/src/main/kotlin/pl/org/seva/events/main/KodeinModuleBuilder.kt
[location-picker]: https://github.com/syrop/Victor-Events/blob/master/events/src/main/kotlin/pl/org/seva/events/event/LocationPickerFragment.kt
[pickers]: https://developer.android.com/guide/topics/ui/controls/pickers
[create-event-fragment]: https://github.com/syrop/Victor-Events/blob/master/events/src/main/kotlin/pl/org/seva/events/event/CreateEventFragment.kt

