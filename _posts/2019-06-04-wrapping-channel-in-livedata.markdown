---
layout: post
title:  "Wrapping Channel in LiveData (refactoring)"
date:   2019-06-04 13:22 +0200
categories: jekyll update
---

This article explains how to write write an extension function that wraps a `Channel` in a `LiveData` and deals with closing it.

## Source material

This article relies heavily on the function [`liveData`][livedata], which I learned about by watching the Google I/O 2019 [YouTube video][youtube]. The speaker (Yigit Boyar) starts explaining this functionality at 15:48.

## The problem

In the [previous article][article] I explained how one can write their own extension class of `LiveData` that handles the `CoroutineScope`, transforms the data it receives and closes the source `Channel` when it is no longer observed.

This time I will demonstrate how to write only one function that wraps any `Channel` in LiveData, and closes it (after a timeout) when the LiveData is no longer observed.

Thanks to doing that there is less code that deals with sensitive asynchronous operations, and therefore fewer opportunities to forget to close a Channel, and less code to maintain.

## The project

The project used in this article is [Compass]. It is available on [Google Play][compass-play] (pending approval by Google).

You can see this particular refactoring discussed in the present article in a [commit].

In this solution I use the following dependency:

```groovy
implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.2.0-alpha01'
```

## Creating the LiveData

This is the function that creates the LiveData:

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

Each time the `LiveData` becomes active (the number of active observers change to 1 from 0), the lambda passed to `liveData()` is invoked again, so a new instance of the `Channel` is created.

It is important to note that `liveData()` uses `Dispatchers.Main` by default, so that the channel is created on main thread. This is important, because the function `requestLocationUpdates()` on `FusedLocationProviderClient` can be also only called from the main thread.

## Using the LiveData

As it has been shown in the previous section, the function `channelLiveData()` retuns an instance of `Lazy`, which can be used to lazily initiate the `LiveData`.

(In case you need eager initialization, you can use the [`value`][value] property).

This is the code that creates two instances of `LiveData` using the function described in the previous section:

```kotlin
@ExperimentalCoroutinesApi
val rotation by channelLiveData { rotationChannelFactory.getRotationChannel() }

@ExperimentalCoroutinesApi
val direction by channelLiveData {
    locationChannelFactory.getLocationChannel().map { location ->
        withContext(Dispatchers.Default) {
            val distance = distance(location)
            val bearing = bearing(location)
            DirectionModel(distance.await(), bearing.await())
        }
    }
}
```

One of the lambdas shown above that are passed to `channelLiveData()` asynchronously map the values received from one channel in order to create another channel that is later wrapped in the created LiveData.

This is normal, and demonstates how in coroutines the `CoroutineDispatcher` can be switched at will. The channel is created on the main thread, so for example `getLocationChannel()` is invoked on the main thread, but the processor-intensive mapping operation is executed using `Dispatchers.Default` to take advantage of multi-core architecture.

Calculating `distance()` and `bearing()` was already discussed in the [previous article][article]. During the present refactoring the code of these two functions is moved to the `ViewModel`, what you can see in the [commit].

## Conclusion

The present article has demonstrated how a piece of information found in a [YouTube video][youtube] can be quickly used to create an interesting refactoring.

The article deliberately promotes the view that employees should be both allowed and expected to watch YouTube at work, at least in a company that wants to create opportunities for professional growth.

I hope that by performing and documenting this refoctoring I've clarified how to handle `CoroutineScope`s and switch between `CoroutineDispatcher`s.

I have also tried to further promote separation of concerns, so that the instances of `LiveData` do not have to be aware of the `viewModelScope`, but instead use `LiveDataScope` from `lifecycle-livedata-ktx`.

The project is at the moment finished, so I do not expect further impromements in the code, and at the moment I do not see a need to introduce these changes into my main project, [Victor Events][events].

I hope, however, that by writing the present article I've provided a demonstration on how to document an instance of [refactoring][commit], so that other developers may understand the author's rationale for the changes, which can help them to decide whether they want to keep them, build on them or revoke them.



[livedata]: https://developer.android.com/topic/libraries/architecture/coroutines#livedata
[youtube]: https://youtu.be/BOHK_w09pVA?t=948
[article]: https://syrop.github.io/jekyll/update/2019/06/02/extending-livedata.html
[compass]: https://github.com/syrop/Compass
[compass-play]: https://play.google.com/store/apps/details?id=pl.org.seva.compass
[value]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-lazy/value.html
[events]: https://github.com/syrop/Victor-Events
[commit]: https://github.com/syrop/Compass/commit/ae310f3af228271246995126bdf898f94e55578f

