---
layout: post
title:  "Rx Observables to LiveData"
date:   2019-03-07 15:08:00 +0100
categories: jekyll update
---

This article explains how to convert RxJava's `Observable`s to `LiveData`.

For explanation how to automatically dispose RxJava's `Disposable`s on specific lifecycle events, please read the [previous article][subscriptions]. This time the `Disposable` is not disposed at all, but it only captures one instance of `LiveData`, so that it doesn't cause leak of activities or UI elements.

The code presented here is used in the project [Victor Events][victor-events].

This is the extension function that converts the `Observable` to `LiveData`:

{% highlight kotlin %}

fun <T> Observable<T>.toLiveData() : LiveData<T> =
    MutableLiveData<T>().apply {
        this@toLiveData.subscribe { value = it }
    }

{% endhighlight %}

Notice the expression `this@toLiveData`. Because I use the function `apply()` on the returned value, there are two different kinds of `this` available in the code.

By default `this` indicates the value the `apply()` function is called on, but when I write `this@toLiveData`, the compiler understands it to be the receiver of the entire `toLiveData()` function.

Please also notice that I specify `LiveData<T>` to be the return type of `toLiveData()`. I do this so that the caller doesn't see the return value as an instance of `MutableLiveData`, so that the caller can't set the values.

## Observing

This is the way to convert RxJava's `Observable` to an instance of `LiveData` and observe it:

{% highlight kotlin %}

deniedSubject
    .filter { it.requestCode == requestCode && it.permission == permission.permission }
    .toLiveData()
    .observe(fragment) { permission.onDenied() }

{% endhighlight %}

I can take advantage of RxJava's `filter` operator, and then I do not have to care about disposing the `Disposable`, because `LiveData` takes care of the lifecycle for me.

This is the extension function I used to simplify observing `LiveData`:

{% highlight kotlin %}

fun <T> LiveData<T>.observe(owner: LifecycleOwner, f: (T) -> Unit) =
    observe(owner, Observer<T> { f(it) })

{% endhighlight %}

## Conclusion

In this article I described the most simple way of combining the versatility of RxJava with the ease of use of `LiveData`.

In the [previous article][subscriptions] I explained how to implement several extension functions you can use to handle lifecycle events direcly with RxJava's `Disposable`s. However, I think that as [Android Jetpack][jetpack] gains popularity, you may need a more generic way that can be applied both to SDKs that already return `LiveData`, and the ones that return `Observable`s that can be converted to `LiveData` in the way I just described.

[victor-events]: https://github.com/syrop/Victor-Events
[subscriptions]: https://syrop.github.io/jekyll/update/2018/12/25/lifecycle-aware-rx-subscriptions.html
[jetpack]: https://developer.android.com/jetpack/

