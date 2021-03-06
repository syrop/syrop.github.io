---
layout: post
title:  "MV* (disambiguation attempt)"
date:   2019-06-30 21:34:00 +0200
categories: jekyll update
---

This article attempts to explain the differences between [MVP], [MVVM] and [MVC].

## Terminology

The *V* in [MVP], [MVVM] and [MVC] doesn't really have to stand for *view*. *View* is just a special case of something that is receiving information from the other parts of the architecture, for example the *view model*.

Thruout the present article, therefore, I will often use the term *consumer*, borrowed from the [producer-consumer problem][producer-consumer]. (Buffering data will not be discussed in the article, though).

I am not going to change the original names [MVP], [MVVM] and [MVC], however, I may occasionally use the term *consumer* interchangeably with *view* to indicate that either of these may not only display data on the GUI, but also record them in a database, a log etc.

## MVP

In [MVP] the *presenter* contains a reference to the *view*, and knows everything there is to know about it.

The *view* may be dependency-injected.

The *presenter* may ask the *view* what it wants: what kind of data, how often. For example, logging data in a text file may require a different data format from one used in the cloud.

In case of Kotlin, the *view*, which in the present article I also call *consumer*, may come in the form of a [sealed class][sealed-class], so that behavior of the *presenter* may depend on the type of the *view*.

I think that `suspend` functions actually support [MVP], as they automatically create a callback, which is known at the time the coroutine is started. The following example comes from my project [Compass]:

```kotlin
private suspend fun getLastLocation(): LatLng? = lastLocation ?:
        suspendCancellableCoroutine { continuation ->
            client.lastLocation.addOnSuccessListener {
                continuation.resume(it?.toLatLng())
            }
        }
```

The above code first checks whether `lastLocation` is already buffered, and if it is not, requests it asynchronously from `FusedLocationProviderClient`.

The `getLastLocation()` function does use a listener inside, but hides it from the calling code. It is called with the line:

```kotlin
lastLocation = getLastLocation()?.also { channel.sendBlocking(it) }
```

The above statement calls `getLastLocation()`, but as soon as the `suspend` function returns, control is passes to `also()` (if the result is not `null`), and eventually the data is assigned to `lastLocation`.

In the above code callbacks, such as `also()`, and eventual assigning of the produced value to `lastLocation`, are already known at the time of writing the code that calls `getLastLocatiot()`, so there isn't much need to add an extra code that handles adding and removing listeners, or marking a *producer* as *active* or *inactive*.

In such cases when it is obvious what is going to happen to the data, and there is no need for creating a *subscription*, or making the data as *active*, I would be comfortable with using [MVP]: Decide beforehand what you want to do with the data. If you must - inject callbacks. Let the *presenter* know beforehand what the *view* (or *consumer*) is going to be like.

If I wanted to wrap the above function in an [MVP] context, I could just use the following code:

```kotlin
class LastLocationPresenter(lastLocationView: (LatLng?) -> Unit) {

    init {
        GlobalScope.launch {
            lastLocationView(getLastLocation())
        }
    }

    private suspend fun getLastLocation(): LatLng? = ...

}
```

The above code snippet involves a *presenter*, a *view* and dependency injection. The `getLastLocation()` function should probably obtain the location from a *model*.

## MVVM

[MVVM] is good for the time when the *consumer* (*view*) is not immediately known, or when the *producer* (here: *view model*) is not aware of the *consumer* (*view*) at all.

The *view model* of [MVVM] is only notified when it becomes *active* (there is at least one *observer*) or *inactive* (the last *observer* has disappeared, and an optional timeout has elapsed).

Observation of the data coming from the *view model* may be, for example, implemented with RxJava or with `LiveData`. The present article demonstrates the latter case.

When `LiveData` is extended, the developer may use the functions [`onActive()`][on-active] or [`onInactive()`][on-inactive] to be notified whether the data becomes *active* or otherwise. I documented such approach in an [article][extending-article] in this blog:

```kotlin
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
```

In the above code a `Channel` is created only when there is at least one *observer*, or the `LiveData` becomes *active*, and both the `Channel` and the corresponding `Job` is canceled when the last *observer* disappears, or the `LiveData` becomes *inactive*.

Alternatively, the developer may choose to use the function [`liveData()`][livedata-function], which I described in a [separate article][wrapping-article] in this blog:

```kotlin
fun <T> channelLiveData(block: () -> ReceiveChannel<T>): Lazy<LiveData<T>> = lazy {
    liveData {
        val channel = block()
        try {
            while (true) {
                emit(channel.receive())
            }
        }
        finally {
            channel.cancel()
        }
    }
}
```

Here, in the above code, the `Channel` is only created when thus created `LiveData` becomes *active*. When it is *inactive*, and the default timeout elapses, the code throws `CancellationException` and closes the channel in the `finally` block.

## Testing MVVM

[Another article][testing-article] in this blog explains how to test [MVVM]:

```kotlin
@Test
fun distanceTest() = runBlocking {

    var distance = 0.0
    var lastDistance = distance

    suspend fun progress() {
        delay(INTERVAL)
        assertTrue(distance > lastDistance)
        lastDistance = distance
    }

    ...

    val liveDataJob = Job()

    val vm = CompassViewModel(
            mockRotationChannelFactory,
            mockLocationChannelFactory,
            coroutineContext + liveDataJob)
    vm.setDestination(DestinationModel(HOME, ""))

    val destination = vm.direction

    val distanceObserver = Observer<DirectionModel> {
        distance = it.distance
    }
    destination.observeForever(distanceObserver)
    delay(STABILIZE_DELAY)
    progress()
    progress()
    assertFalse(locationChannel.isClosedForReceive)
    assertFalse(locationChannel.isClosedForSend)
    liveDataJob.cancel()
    delay(STABILIZE_DELAY)
    assertTrue(locationChannel.isClosedForReceive)
    assertTrue(locationChannel.isClosedForSend)
}
```

The above code does the following:

1. Create an instance of `CompassViewModel`. Inject three dependencies into it.
2. Call `observeForever()` to create an `Observer` that will observe distances emitted by the `CompassViewModel`.
3. Wait a short time and assert that the distance has changed. Repeat.
4. Cancel the `Job` used internally by `CompassViewModel`.
5. Assert that the `Channel` is closed.

## Inactive LiveData

If the developer wants to test `CompassViewModel` with lifecycle, using the function `observe()` instead of `observeForever()`, they need to be aware that the [`liveData()`][livedata-function] function used instead only throws `CancellationException` after a timeout. The timeout is by default equal to 5 seconds.

The same is true when the developer wants to remove the observer by calling `removeObserver()` or `removeObservers()`. Implementing `CompassViewModel` with [`liveData()`][livedata-function] has been explained in a [separate article][wrapping-article] in this blog.

To avoid having to deal with this timeout, the developer may manually set it to 0 seconds, or decline using [`liveData()`][livedata-function] altogether in the implementation of `CompassViewModel`. They may do so by extending `LiveData` manually.

When `LiveData` is extended manually by overriding the functions [`onActive()`][on-active] and [`onInactive()`][on-inactive], the way I explained in [another article][extending-article] in this blog, the `Channel` is closed immediately, so the developer may write tests without taking timeout into consideration. Doing so is outside of the scope of the present article, though.

## MVC

I don't think I have ever consciously used [MVC] in Android. The Wikipedia image below shows that in [MVC] the *view* doesnt't have a relationship with the *controller*. All that the *view* does, according to my understanding of the diagram, is it displays data that has been made available to it:

[![MVC](https://upload.wikimedia.org/wikipedia/commons/thumb/a/a0/MVC-Process.svg/200px-MVC-Process.svg.png)][MVC]

I don't think [MVC] has been fully implemented in Android. In Android views may update their own state, and other parts of the architecture may listen to those changes.

I hadn't really seen any problem with that approach until I saw a [talk] from Google I/O 2019 about [Jetpack Compose][jetpack-compose]. The timestamp [23:40][flow] talks specifically about data flow in [Jetpack Compose][jetpack-compose]:

> The application passes data, in this case a list of stories, into the newsfeed. The newsfeed is then going to iterate over this list of stories, extracting each of the pieces of story data from the list, and passing each story data to each story widget.
>
> The story widget, then, reads the title, the image and the content from the story data.
>
> The important thing here is that **the parent is always in control of the data for this child**. If the data needs to be shared between multiple widgets, the data should be hoisted up to a common ancestor, and passed down to each of the widgets that needs it. Children should not be reading from global variables or global data stores. **They should be side-effect free, and should not do anything, except what their parent tells them to do.**

According to the above quote, views in [Jetpakc Compose][jetpack-compose] can't modify their own state, or write data directly to a global state. Instead, there is a *controller* (in the talk called *parent*) that takes events from the user, processes them, and only then decides whether to update values stored in a global data store (the *model*) and displayed in the *view*.

The reader is welcome to scroll the [talk] to [7:35][events] when the speaker discusses event handling in [Jetpack Compose][jetpack-compose].

While the [talk] does not say that [Jetpack Compose][jetpack-compose] is [MVC], I think that it is pretty close to it, as it introduces an extra layer (*controller*) that intercepts event coming from the user even before the updated values are displayed on the screen.

## Conclusion

I hope to have correctly explained [MVP], [MVVM] and [MVC]. If the reader finds any errors, please contact me by [submitting a ticket][ticket], and I will be glad to edit the article and give them due credit.

[mvvm]: https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel
[mvp]: https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter
[mvc]: https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller
[producer-consumer]: https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem
[sealed-class]: https://kotlinlang.org/docs/reference/sealed-classes.html
[compass]: https://github.com/syrop/Compass
[on-active]: https://developer.android.com/reference/android/arch/lifecycle/LiveData.html#onActive()
[on-inactive]: https://developer.android.com/reference/android/arch/lifecycle/LiveData.html#onInactive()
[extending-article]: https://syrop.github.io/jekyll/update/2019/06/02/extending-livedata.html
[livedata-function]: https://developer.android.com/topic/libraries/architecture/coroutines#livedata
[wrapping-article]: https://syrop.github.io/jekyll/update/2019/06/04/wrapping-channel-in-livedata.html
[testing-article]: https://syrop.github.io/jekyll/update/2019/06/21/viewmodel-mockito.html
[talk]: https://www.youtube.com/watch?v=VsStyq4Lzxo
[jetpack-compose]: https://developer.android.com/jetpack/compose/
[flow]: https://youtu.be/VsStyq4Lzxo?t=1420
[events]: https://youtu.be/VsStyq4Lzxo?t=455
[ticket]: https://github.com/syrop/syrop.github.io/issues/new

