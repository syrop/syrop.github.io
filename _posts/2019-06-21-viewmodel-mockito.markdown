---
layout: post
title:  "Testing ViewModel with Mockito"
date:   2019-06-21 13:38:00 +0200
categories: jekyll update
---

This article explains how to implement [Mockito] tests with coroutines.

## The problem

Add Mockito tests to the [project][compass] using depencency injection (refactoring documented in the [previous article][previously]).

## Credits

I learned how to set up the testing rules so that the code can rely on the main thread from an [answer][stackoverflow1] on StackOverflow. I learned how to use the `@Ruled` annotation in Kotlin from [another answer][stackoverflow2].

## Preparing the project

According to the [doco][dispatchers], this is how one configures the tests so that they can use `Dispatchers.Main`:

```kotlin
private val mainThreadSurrogate = newSingleThreadContext("UI thread")

@Before
fun setUp() {
    Dispatchers.setMain(mainThreadSurrogate)
}

@After
fun tearDown() {
    Dispatchers.resetMain() // reset main dispatcher to the original Main dispatcher
    mainThreadSurrogate.close()
}
```

The classes and particular functions that are going to be mocked also need the `open` keyword:

```kotlin
open class LocationChannelFactory(ctx: Context)
```

and:

```kotlin
open fun getLocationChannel(): ReceiveChannel<LatLng>
```

Without `open` next to the class name the test simply wouldn't compile. Without `open` at the function name it would compile, but it would call the actual function, instead of mocking its result, when I write:

```kotlin
`when`(mockLocationChannelFactory.getLocationChannel())
        .thenReturn(locationChannel)
```

## Testing

[Doco] recommends using [`runBlockingTest()`][runblockingtest], which modifies the `CoroutineDispatcher`, so that whenever `delay()` is called, it doesn't wait, but proceeds immediately to the next line. I can't use that, because in the code I use some computationally-intensive calculations, such as measuring the distance between two points. I do need `delay()` to wait several milliseconds, so I can't do that. I decided to use [`runBlocking()`][runblocking] instead:

```kotlin
@Test
fun distanceTest() = runBlocking {

    var distance = 0.0
    var lastDistance = distance
```

I start by setting the initial value of `distance` to `0.0`. I then define a local function that will check whether the distance has increased since the last check:

```kotlin
suspend fun progress() {
    delay(INTERVAL)
    assertTrue(distance > lastDistance)
    lastDistance = distance
}
```

Subsequently I create mock instances of `LocationChannelFactory` and `RotationChannelFactory`. For brevity, only `LocationChannelFactory` is discussed herein:

```kotlin
val mockLocationChannelFactory: LocationChannelFactory = mock(LocationChannelFactory::class.java)
val mockRotationChannelFactory: RotationChannelFactory = mock(RotationChannelFactory::class.java)

val locationChannel = Channel<LatLng>(Channel.CONFLATED)
val rotationChannel = Channel<Float>(Channel.CONFLATED)

`when`(mockLocationChannelFactory.getLocationChannel()).thenReturn(locationChannel)
`when`(mockRotationChannelFactory.getRotationChannel()).thenReturn(rotationChannel)
```

Subsequently there is a definition of the `Job` that will send fake locations to the `Channel`:

```kotlin
launch(Dispatchers.IO) {
    locationChannel.send(HOME)
    var lat = HOME.latitude
    while (true) {
        delay(INTERVAL)
        lat += LATITUDE_STEP
        locationChannel.send(LatLng(lat, HOME.longitude))
    }
}
```

Next, I create the actual `ViewModel` and start observing the `LiveData`:

```kotlin
val liveDataJob = Job()

val vm = CompassViewModel(
        mockRotationChannelFactory,
        mockLocationChannelFactory,
        EmptyCoroutineContext + liveDataJob)
vm.setDestination(DestinationModel(HOME, ""))

val destination = vm.direction

val distanceObserver = Observer<DirectionModel> {
   distance = it.distance
}
destination.observeForever(distanceObserver)
```

In the above code, I need to crate a `Job` that will be later canceled in order to throw `CancellationException` inside the `LiveData`. I need this in order to test whether the `Channel` is closed. Alternatively, I could test that by just removing the `Observer` from the `LiveData`, and waiting a couple of seconds for the timeout to kick in. To save time, I decided to instead manually cancel the `CoroutineContext` used in this `LiveData`. This is the code that creates the `LiveData` that wraps the `Channel` returned by `LocationChannelFactory`:

```kotlin
fun <T> channelLiveData(
        context: CoroutineContext = EmptyCoroutineContext,
        block: () -> ReceiveChannel<T>): Lazy<LiveData<T>> = lazy {
    liveData(context) {
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

The way `LiveData` is created inside the `CompassViewModel` assures that the `Channel` it uses will be closed when the `CoroutineContext` is canceled.

Next, there is the code that waits for the `CompassViewModel` to stabilize (start emitting values) and calls `progress()` two times. This is the local function that waits a bit, and then asserts that the distance calculated by the `CompassViewModel` has increased during that time.

Finally, there is the code that asserts the `Channel` hasn't been closed yet, cancels the `Job` associated with the `LiveData`, waits again for the `CompassViewModel` to stabilize, and asserts that the `Channel` which that `LiveData` was using is closed now.

## Conclusion

The present article has demonstated the use of the following:

* Mockito.
* Depencency injection in constructor.
* Testing coroutines using `runBlocking()`.
* Checking at the end of the test that `Channel` has been closed.

In order to achieve this, I had to replace dependency retrieval, whith I had been using extensively in this blog before that time, with dependency injection. I described the process in the [previous article][previously].

Choosing dependency injection or dependecny retrieval is a matter of personal preference of the developer, as both can be successfully used for testin. I discussed testing with dependency injection in an [early article][testing] in this blog.

## Donations

If the reader has enjoyed the present article, they may want to donate some bitcoin at the address presented below. Readers may also look at my [donations page][donate].

BTC: bc1qncxh5xs6erq6w4qz3a7xl7f50agrgn3w58dsfp

[mockito]: https://site.mockito.org/
[compass]: https://github.com/syrop/Compass
[previously]: https://syrop.github.io/jekyll/update/2019/06/20/dependency-injection.html
[stackoverflow1]: https://stackoverflow.com/a/49840604/10821419
[stackoverflow2]: https://stackoverflow.com/a/32827600/10821419
[dispatchers]: https://github.com/Kotlin/kotlinx.coroutines/tree/master/kotlinx-coroutines-test
[doco]: https://github.com/Kotlin/kotlinx.coroutines/tree/master/kotlinx-coroutines-test
[runblockingtest]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-test/kotlinx.coroutines.test/run-blocking-test.html
[runblocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[testing]: https://syrop.github.io/jekyll/update/2018/12/25/testing-with-dependency-retrieval.html
[donate]: https://syrop.github.io/donate/

