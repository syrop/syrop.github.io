---
layout: post
title:  "Extending LiveData (implementing a compass)"
date:   2019-06-02 20:35 +0200
categories: jekyll update
---

This article explains how to write your own implementation of `LiveData`.

## The project

The project used in this article is [Compass]. It is available on [Google Play][compass-play].

The app first displays a compass. When the user clicks on an empty *Address* field, Google Map is shown where the user can enter their destination location by long-pressing anywhere on the map. When destination is thus selected, the home screen will display the destination's street address, current distance and a large arrow indicating the direction on the top of the compass.

When the map is displayed for the first time, the app probably doesn't have location permission yet. That's why the user can't automatically scroll the map to the present location of the device. However, once they scroll the map, or change its zoom, these properties are persisted, so that the same position of the map is displayed the next time.

When the user selects the destination and returns to the home screen, the app requests location permission, in order to display the distance and direction. Once location permission is granted, the destination selection maps allow automatic scrolling to the present location, although it shouldn't be necessary, because one can quickly scroll the map to their favourite city, and the position and zoom of the camera will pe persisted.

## Credits

I began learning how to implement a compass by reading an [article][ninja] in another blog. I hardly retained anything of the original code, but I did retain the rotation animation code, together with its magic numbers found in the article. Here is my Kotlin version, wrapped in a custom extension of `ImageView`:

```kotlin
class RotationImageView @JvmOverloads constructor(
        context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : ImageView(context, attrs, defStyleAttr) {

    private var lastRotation = 0f

    fun rotate(rotation: Float) {
        // Rotation code credit:
        // https://www.androidcode.ninja/android-compass-code-example/
        startAnimation(RotateAnimation(
                lastRotation,
                -rotation,
                Animation.RELATIVE_TO_SELF,
                0.5f,
                Animation.RELATIVE_TO_SELF,
                0.5f).apply {
            duration = DURATION_MS
            fillAfter = true
        })
        lastRotation = -rotation
    }

    companion object {
        private const val DURATION_MS = 210L
    }
}
```

The image of the compass is downloaded from <http://www.pngall.com/compass-png/download/16293> and the image of the arrow from <https://www.wpclipart.com/small_icons/pointers_large/arrow_green_up.png.html>. They both have an appropriate license.

## Previously in this blog

Displaying Google Maps has already been described in these two articles:

* [Displaying Google Maps][displaying-maps]
* [Selecting location][selecting-location]

Handling permissions has also already been discussed in these two articles:

* [Permissions, RxJava and lifecycle][permissions]
* [Permissions handling (refactoring)][permissions-refactoring]

The above architecture areas have been nicely addressed previously in this blog, and do not require further explanation in the present article.

## The source of truth

This section explains how to procure two kinds of data: rotation of the device (azimuth) and the location.

This is the code that requests location updates and sends them to a `Channel`:

```kotlin
class LocationChannelFactory(ctx: Context) {

    private val client = LocationServices.getFusedLocationProviderClient(ctx)
    private val request = LocationRequest.create().apply {
        interval = INTERVAL
        priority = LocationRequest.PRIORITY_HIGH_ACCURACY
    }

    private var lastLocation: LatLng? = null

    @ExperimentalCoroutinesApi
    fun getLocationChannel(): ReceiveChannel<LatLng> = Channel<LatLng>(Channel.CONFLATED).also { channel ->
        val lastLocationJob = GlobalScope.launch {
            lastLocation = getLastLocation()?.also { channel.sendBlocking(it) }
        }
        val callback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                lastLocationJob.cancel()
                lastLocation = result.lastLocation.toLatLng().also { channel.sendBlocking(it) }
            }
        }

        client.requestLocationUpdates(request, callback, Looper.myLooper())
        channel.invokeOnClose { client.removeLocationUpdates(callback) }
    }

    private suspend fun getLastLocation(): LatLng? = lastLocation ?:
            suspendCancellableCoroutine { continuation ->
                client.lastLocation.addOnSuccessListener(OnSuccessListener {
                    continuation.resume(it?.toLatLng())
                })
            }

    private fun Location.toLatLng() = LatLng(latitude, longitude)

    companion object {
        private const val INTERVAL = 1000L
    }
}
```

Notice the `getLastLocation()` function. It checks whether last location has already been cached and only suspends if it hasn't.

It is called asynchronously by `Globalscope.launch`, but it is canceled when the present location is received from a `LocationCallback`. Either way, the location is cached and send to the `Channel`.

When the channel is closed, the request is canceled. Please note that the function `invokeOnClosed()` called on the `Channel` is experimental, so I used the annotation `@ExperimentalCoroutinesApi` to avoid compiler warnings.

This is the code that calculates the rotation of the device (azimuth) and sends it to another `Channel`:

```kotlin
class RotationChannelFactory(ctx: Context) {

    private val manager = ctx.getSystemService(Context.SENSOR_SERVICE) as SensorManager
    private val magneticSensor: Sensor? = manager.getDefaultSensor(Sensor.TYPE_MAGNETIC_FIELD)
    private val accelSensor: Sensor? = manager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)

    @ExperimentalCoroutinesApi
    fun getRotationChannel(): ReceiveChannel<Float> = Channel<Float>(Channel.CONFLATED).also { channel ->
        if (magneticSensor == null || accelSensor == null) return@also
        var valuesMagnet: FloatArray? = null
        var valuesAccel: FloatArray? = null
        val rotationMatrix = FloatArray(9)
        val orientation = FloatArray(3)

        val callback = object : SensorEventCallback() {
            override fun onSensorChanged(event: SensorEvent) {
                when (event.sensor.type) {
                    Sensor.TYPE_MAGNETIC_FIELD -> valuesMagnet = event.values
                    Sensor.TYPE_ACCELEROMETER -> valuesAccel = event.values
                }

                SensorManager.getRotationMatrix(
                        rotationMatrix,
                        null,
                        valuesAccel ?: return,
                        valuesMagnet ?: return)

                SensorManager.getOrientation(rotationMatrix, orientation)
                val azimuth = orientation[0]
                channel.sendBlocking((azimuth / Math.PI * 180.0).toFloat())
            }
        }
        manager.registerListener(callback, magneticSensor, SensorManager.SENSOR_DELAY_NORMAL)
        manager.registerListener(callback, accelSensor, SensorManager.SENSOR_DELAY_NORMAL)
        channel.invokeOnClose { manager.unregisterListener(callback) }
    }
}
```

The reader can learn the meaning of the above code from the [getOrientation()][get-orientation] doco, as well as from [overview of the sensors][sensors-overview].

The noteworthy part of the above code is the use of the Elvis `?:` operator. The [`getRotationMatrix()`][get-rotation-matrix] function cannot have any of its last two parameters set to `null`, yet two of the arrays may be set at two different times, in an unknown order, and are otherwise `null`.

The Elvis operator above checks whether either of the values is `null`, and `return`s if it is.

Because both `return` statements are inside a function `onSensorChanged()`, and not inside a lambda, they can be called by themselves, and they will `return` from the function before `SensorManager.getRotationMatrix()` is invoked, and therefore prevent a `NullPointerException`.

You may notice that at the top of the above code block, in the first line of the implementation of `getRotationChannel()`, there is another type of `return` statement: `return@also`. It is so because this statement is inside a lambda, so the `@` label is an indication that the `return` statement only `returns` from the lambda, and not from the entire function.

Even though the two other `return` statements are also inside of the same lambda, they belong to a separate class, and to a separate function, and may be therefore written without a `@` label.

## LiveData

This is the code that calculates the distance to the chosen destination, as well as the direction:

```kotlin
class DirectionLiveData(private val scope: CoroutineScope) : LiveData<DirectionModel>() {

    private var locationChannel: ReceiveChannel<LatLng>? = null
    private var locationJob: Job? = null
    lateinit var destinationLocation: LatLng

    @ExperimentalCoroutinesApi
    override fun onActive() {
        locationChannel = locationChannelFactory.getLocationChannel()
        locationJob = scope.launch {
            while (true) {
                val location = locationChannel!!.receive()
                val distance = distance(location)
                val bearing = bearing(location)
                postValue(DirectionModel(distance.await(), bearing.await()))
            }
        }
    }

    override fun onInactive() {
        locationJob?.cancel()
        locationChannel?.cancel()
    }

    private fun CoroutineScope.distance(location: LatLng) = async {
        val dLat = Math.toRadians(destinationLocation.latitude - location.latitude)
        val dLon = Math.toRadians(destinationLocation.longitude - location.longitude)
        val radLatLoc = Math.toRadians(location.latitude)
        val radLatDest = Math.toRadians(destinationLocation.latitude)
        val a = sin(dLat / 2).pow(2) +
                sin(dLon / 2) * sin(dLon / 2) * cos(radLatLoc) * cos(radLatDest)
        val c = 2 * asin(sqrt(a))
        RADIUS_KM * c
    }

    private fun CoroutineScope.bearing(location: LatLng) = async {
        val longDiff = destinationLocation.longitude - location.longitude
        val y = sin(longDiff) * cos(destinationLocation.latitude)
        val x = cos(location.latitude) *
                sin(destinationLocation.latitude) - sin(location.latitude) *
                cos(destinationLocation.latitude) * cos(longDiff)
        ((Math.toDegrees(atan2(y, x)) + 360 ) % 360).toFloat()
    }

    companion object {
        const val RADIUS_KM = 6371.0
    }
}
```

I assumed that because calculation of the distance and bearing is rather expensive, but independent from each other, both can be run asynchronously. As I will explain further in the article, to take advantage of the multi-core processor that will be probably running this code, I recommend using `Dispatchers.Default` for running the code. This is set by passing an appropriate `CoroutineScope` in the constructor of this class, and will be discussed further in the present article.

`destinationLocation` has been marked as `lateinit`, so the application will crash when this `LiveData` is observed before the `destinationLocation` is set. This is by design. Because the app allows changing the destination, the value cannot be passed in the constructor.

Even though the app allows deleting the set destination, there is no need to set the value of `destinationLocation` to `null`. It's enough to stop observing this `LiveData`, so `onInactive()` will cancel the `Channel`, and therefore free up the sensors. However, because I marked `destinationLocation` as `lateInit`, It will become very obvious, and therefore easy to fix, when this `LiveData` is used inappropriately.

In the project I assume that the Earth is round, because equations corresponding to this model are relatively easy to find on the Internet. Even though there is [compelling evidence to the contrary][flat-earthers], I am not aware at the moment of any corresponding alternative equations. My personal stance on the matter is beyond the scope of the present article.

Calculating rotation (azimuth) of the device has already been discussed in the previous section. The code wrapping it in `LiveData` is very simple:

```kotlin
class RotationLiveData(private val scope: CoroutineScope) : LiveData<Float>() {

    private var rotationChannel: ReceiveChannel<Float>? = null
    private var rotationJob: Job? = null

    @ExperimentalCoroutinesApi
    override fun onActive() {
        rotationChannel = rotationChannelFactory.getRotationChannel()
        rotationJob = scope.launch {
            while (true) {
                postValue(rotationChannel!!.receive())
            }
        }
    }

    override fun onInactive() {
        rotationJob?.cancel()
        rotationChannel?.cancel()
    }
}
```

## The ViewModel

This is the code of the ViewModel. Notice that when it needs to pass an instance of `LiveData` that require a `CoroutineScope`, it uses the construction `viewModelScope + Dispatchers.Default`. (It must do so, because by default `viewModelScope` uses `Dispatchers.Main`, which runs on main thread).

```kotlin
class CompassViewModel : ViewModel() {

    private val mutableDestination by lazy { MutableLiveData<DestinationModel?>() }
    val rotation by lazy { RotationLiveData(viewModelScope + Dispatchers.Default) }
    val direction by lazy { DirectionLiveData(viewModelScope + Dispatchers.Default) }
    val destination get() = mutableDestination as LiveData<DestinationModel?>

    fun setDestination(destination: DestinationModel?) {
        if (destination != null) {
            direction.destinationLocation = destination.location
        }
        mutableDestination.value = destination
    }
}
```

It has one extra function `setDestination()` that sets the destination in the `DirectionLiveData` (only when it is not `null`), and also sets it in `mutableDestination` (the latter is observed by the `Fragment`, but only to display the address, and start observing other `LiveData`s if location is set). Because this function is called only from the main thread (when the user manuall selects location), I can use the more direct function `setValue()` to set the value of `mutableDestination`. To assign values in functions that are called from other threads (for example, when a value is received from a source other than the user), I have to use a more indirect function `postValue()` to assure that the value is anyway only assigned in the main thread.

## The Fragment

This is the most important function of the `Fragment` that combines all types of data (rotation of the device, direction and distance to the destination):

```kotlin
override fun onActivityCreated(savedInstanceState: Bundle?) {
    super.onActivityCreated(savedInstanceState)
    var bearing: Float? = null
    address { nav(R.id.action_compassFragment_to_destinationPickerFragment) }
    (viewModel.destination to this) { destination ->
        address set (destination?.address)
        if (destination == null) {
            distance_layout.visibility = View.GONE
            arrow.visibility = View.GONE
            viewModel.direction.removeObservers(this)
            bearing = null
        }
        else {
            requestLocationPermission {
                distance_layout.visibility = View.VISIBLE
                arrow.visibility = View.VISIBLE
                (viewModel.direction to this) { direction ->
                    distance set direction.distance.toString()
                    bearing = direction.degrees
                }
            }
        }
    }
    (viewModel.rotation to this) { rotation ->
        bearing?.let { arrow.rotate(rotation - it) }
        compass.rotate(rotation)
    }
}
```

I use a handy shortcut that starts obseving `LiveData` when it is combined with a `LifecycleOwner` in a `Pair`, and `invoke()` operator is called on it:

```kotlin
operator fun <T> Pair<LiveData<T>, LifecycleOwner>.invoke(observer: (T) -> Unit) = first(second, observer)

operator fun <T> LiveData<T>.invoke(owner: LifecycleOwner, observer: (T) -> Unit) =
        observe(owner) { observer(it) }
```

Another handy shortcut is setting `OnClickListener` with another `invoke()` operator:

```kotlin
operator fun View.invoke(l: (View) -> Unit) = onClick(l)

infix fun View.onClick(l: (View) -> Unit) = setOnClickListener(l)
```

Other than setting a listener and displaying the address of the selected location the `onActivityCreated()` function sets up observation of two instances of `LiveData`.

First, destination is observed. When it is null, the code hides the view that otherwise shows distance, and the arrow, and stops observing direction.

When destination is not null, the code checks whether the app has location permission. If it is so, the code in the lambda is called immediately. When the app doesn't already have the permission, it firsts requests it and calls the code in the lambda only when it is granted.

If destination is set, and location permission is granted, it starts observing direction. It shows the distance on the screen and caches bearing.

Permission handling is quite a complex matter, especially when one wants to pass just one lambda that might be called either immediately, or as a callback. Permission handling has been discussed in other articles in this blog, and the reader might want to navigate to them using the links presented at the top of the present text.

Because device's rotation is updated way more often (many times per second) than its location, the arrow needs to be rotated (simultaneously with the compass) on each such rotation update. That's why it is first cached when location is received, and only then rotated when the device's rotation is also known.

The call `bearing?.let { arrow.rotate(rotation - it) }` checks whether `bearing` is not null, and when it is so, makes a copy of its value in the `it` constant.

I cannot just use `if (bearing != null)` and then refer to `bearing` again, because it might have been changed before then. I always use the `?.let` syntax when I want to access a nullable variable, because it not only makes the check, but also makes a constant copy.

## Conclusion

I hope to have successfully presented a three-tier architecture for presenting data coming asynchronously from location or from sensors.

To recapitulate, the three tiers are as follows:

1. Create a `Channel` that will be used to convey the values obtained from either source. (Alternatively, you might want to use RxJava, but its usage is beyond the scope of the present article).
2. Wrap the values received from this `Channel` in a `LiveData`. Please note that unless you use `MutableLiveData`, the functions `setValue()` and `postValue()` are `protected`, so `LiveData` has been specifically designed this way: it is meant to be extended, and values are to be assigned internally.
3. Observe the data inside your `Fragment`. The reader might want to use my preferred syntax `(liveData to this) { ... }` to initiate the observation.

Because of the chosen architecture, for instance the `Fragment` is not aware whether I am using coroutines or RxJava, so the GUI shouldn't break when I migrate between one and the other. Also, the code retrieving the actual data isn't aware of lifecycles, so I could use exactly the same code to obtain the location even if I want it to continue to run in the background.

Lastly, the purpose of the article has been to explain to the reader how to write their own classes extending `LiveData`, so that they do not have to rely on `MutableLiveData` to convey the values obtained from location and sensors, but use `LiveData`'s internal mechanisms instead.

## Donations

If you've enjoyed this article, consider donating some bitcoin at the address below. You may also look at my [donations page][donations].

BTC: bc1qncxh5xs6erq6w4qz3a7xl7f50agrgn3w58dsfp

[compass]: https://github.com/syrop/Compass
[compass-play]: https://play.google.com/store/apps/details?id=pl.org.seva.compass
[ninja]: https://www.androidcode.ninja/android-compass-code-example/
[displaying-maps]: https://syrop.github.io/jekyll/update/2018/12/28/displaying-google-maps.html
[selecting-location]: https://syrop.github.io/jekyll/update/2019/01/09/selecting-location-advanced-topic.html
[permissions]: https://syrop.github.io/jekyll/update/2018/12/26/permissions-rxjava-and-lifecycle.html
[permissions-refactoring]: https://syrop.github.io/jekyll/update/2019/04/24/permissions-handling-refactoring.html
[get-orientation]: https://developer.android.com/reference/android/hardware/SensorManager.html#getOrientation(float[],%20float[])
[get-rotation-matrix]: https://developer.android.com/reference/android/hardware/SensorManager.html#getRotationMatrix(float[],%20float[],%20float[],%20float[])
[sensors-overview]: https://developer.android.com/guide/topics/sensors/sensors_overview
[flat-earthers]: https://en.wikipedia.org/wiki/Flat_Earth#Modern_Flat-Earthers
[donations]: https://syrop.github.io/donate/

