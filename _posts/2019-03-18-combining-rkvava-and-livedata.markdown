---
layout: post
title:  "Combining RxJava and LiveData (advanced)"
date:   2019-03-18 9:29:00 +0100
categories: jekyll update
---

This article explains how to combine RxJava's `Observable` and Android Jetpack's `LiveData`.

## In the previous episode

In the [previous article][previous-article] I explained how to *convert* the `Observable` into an instance of `LiveData`. The conversion was executed this way:

{% highlight kotlin %}

deniedSubject
    .filter { it.requestCode == requestCode && it.permission == permission.permission }
    .toLiveData()
    .observe(fragment) { permission.onDenied() }

{% endhighlight %}

The disadvantage of such solution was that `toLiveData` created a `Disposable` that was never disposed. It didn't leak the entire `Activity` or `Fragment`, but it did leave something begind, namely the RxJava's `Disposable` and the `LiveData` itself.

While I was writing the previous article I considered it a good trade off, as you were gaining a convenient way combine `LiveData` with `Observable` while at the same time avoiding `Activity` leak.

## Improved solution

In the present article I will explain how to use `LiveData` in a way similar to the one described previously, but this time avoiding any leaks at all, and disposing the `Disposable` as soon as `Activity` or `Fragment` is destroyed.

The solution is present project [Victor Events][victor-events] in its most up-to-date state on GitHub.

This time you skip the `toLiveData()` function, invoked in the above code snippet, and invoke `observe()` directly:

{% highlight kotlin %}

deniedSubject
    .filter { it.requestCode == requestCode && it.permission == permission.permission }
    .observe(fragment) { permission.onDenied() }

{% endhighlight %}

Code of `observe()`:

{% highlight kotlin %}

fun <T> Observable<T>.observe(owner: LifecycleOwner, observer: (T) -> Unit) {
    val liveData = MutableLiveData<T>()
    liveData.observe(owner, observer)
    val disposable = subscribe { liveData.value = it }
    owner.lifecycle.addObserver(RxLifecycleObserver(disposable))
}

private class RxLifecycleObserver(private val disposable: Disposable) : LifecycleObserver {

    @Suppress("unused")
    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    private fun onEvent() {
        disposable.dispose()
    }
}

{% endhighlight %}

The function creates an instance of `MutableLiveData`, observes it, then creates an RxJava's `Disposable` to update its value, and finally disposes the `Disposable` when the `LifecycleOwner` is destroyed.

I decided to call the above function `observe()` as opposed to `subscribe()`, so that its name suggests that eventually `observed` is called directly on an instance of `LiveData`. The actual invocation is defined in another file:


{% highlight kotlin %}

fun <T> LiveData<T>.observe(owner: LifecycleOwner, f: (T) -> Unit) =
        observe(owner, Observer<T> { f(it) })
{% endhighlight %}

## Advantages

The advantage of the present solution is that it allows bypassing calling `subscribe()` manually, and also handles lifecycle, therefore preventing all leaks.

While `Activity` or `Fragment` is paused, no values will be passed to the UI. When it is resumed, the UI will only receive the most up-to-date value.

## Threading

The above solution only works if the `Observable` is triggered in UI thread. If you think that your `Observable` may be triggered in other threads, you have to apply one of the following solutions:

{% highlight kotlin %}

val disposable = observeOn(AndroidSchedulers.mainThread())
    .subscribe { liveData.value = it }

{% endhighlight %}

The above ensures that the code passed to `subscribe` is only executed on UI thread. If you want to apply this particular solution, you have to use this dependency:

{% highlight groovy %}

implementation 'io.reactivex.rxjava2:rxandroid:2.0.2'

{% endhighlight %}

Another solution is to use the mechanism that is already present in `LiveData`, namely instead of setting the value, you can `post` it:

{% highlight kotlin %}

val disposable = subscribe { liveData.postValue(it) }

{% endhighlight %}

This way you will ensure that the value will be set only on main thread, and you avoid having to add another dependency.

[previous-article]: https://syrop.github.io/jekyll/update/2019/03/07/rx-observable-to-livedata.html
[victor-events]: https://github.com/syrop/Victor-Events

