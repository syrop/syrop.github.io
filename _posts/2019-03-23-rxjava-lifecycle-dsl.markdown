---
layout: post
title:  "RxJava, lifecycle and DSL"
date:   2019-03-23 21:23:00 +0100
categories: jekyll update
---

In this article I will explain how to refactor the example given in the [previous article][previous-article] to use a DSL.

The code presented in the present article is already published in my demonstration project [Timer][timer].

You may not understand the present article unless you have also read, and understood, the [previous one][previous-article].

## Combining the Observable and LiveData

This is the code I use to combine the RxJava's `Observable` and Android Jetpack's `LiveData`. In this example I first create a singleton that holds two `BehaviorSubject`s . Because I don't want to share them out, I use a public infix function that combines an `Observable` with `LiveData`, so that other parts of the application will not be able to use the `Subject` to emit values:

{% highlight kotlin %}

infix fun seconds(liveData: MutableLiveData<Int>) =
        seconds.delay(DELAY_MS, TimeUnit.MILLISECONDS) + liveData

infix fun minutes(liveData: MutableLiveData<Int>) =
        minutes.delay(DELAY_MS, TimeUnit.MILLISECONDS) + liveData

{% endhighlight %}

Because of the reasons explained in the [other article][previous-article] I introduce the delay of 200 milliseconds. In production I wouldn't be doing that.

The above code uses the following extension operator:

{% highlight kotlin %}

operator fun <T> Observable<T>.plus(liveData: MutableLiveData<T>) =
        LiveObservable(this, liveData)

{% endhighlight %}

It creates a data class combining the `Observable` and `LiveData`. This is the whole code of the class:

{% highlight kotlin %}

data class LiveObservable<T>(
        private val observable: Observable<T>,
        private val liveData: MutableLiveData<T>) {

    fun observe(owner: LifecycleOwner, observer: (T) -> Unit) {
        liveData.observe(owner, observer)
        val disposable = observable.subscribe { liveData.postValue(it) }
        owner.lifecycle.addObserver(RxLifecycleObserver(disposable))
    }
}

{% endhighlight %}

## Combining with LifecycleOwner

I do not want to call `observe()` directly on the instances of `LiveObservable`, because in this article I demonstrate how to create a DSL. Therefore, I combine `LiveObservable` with a `LifecycleOwner` in a `Pair`:

{% highlight kotlin %}

timer seconds vm.seconds to this

{% endhighlight %}

I did not want to use another `+` operator, because of operator precedence. If I used `+` operator instead of `to` extension function, it would have been called on `vm.seconds` instead of calling it on the result of `timer seconds vm.seconds`.

To get a consistend look of my DSL, I use a similar syntax to create a `Pair` combining `LiveData` and `LifecycleOwner`:

{% highlight kotlin %}

avm.seconds to this

{% endhighlight %}

It is worth noting that in the above examples the `LiveData` comes from a `ViewModel`, so that it is retained either for the lifecycle of `Fragment`, or even of the whole `Activity`.

## Observing

These are the two extension operators on relevant `Pair`:

{% highlight kotlin %}

@JvmName("liveDataLifecycleOwnerObserve")
operator fun <T> Pair<LiveData<T>, LifecycleOwner>.invoke(observer: (T) -> Unit) =
        first.observe(second, observer)

@JvmName("liveObservableLifecycleOwnerObserve")
operator fun <T> Pair<LiveObservable<T>, LifecycleOwner>.invoke(observer: (T) -> Unit) =
        first.observe(second, observer)

{% endhighlight %}

Because both of the extension operators look very similar, at least one of them needs to be annotated with `@JvmName`, so that Kotlin creates a unique Java name. The parameter of `@JvmName` may be arbitrary if you are not planning to call these functions from Java.

Finally, this is the whole invocation that combines RxJava's `Observable`s with `LiveData`, then with `LifecycleOwner`, and observes them:

{% highlight kotlin %}

(timer seconds vm.seconds to this) { seconds.text = it.toString() }
(timer minutes vm.minutes to this) { minutes.text = it.toString() }

(avm.seconds to this) { stable_seconds.text = it.toString() }
(avm.minutes to this) { stable_minutes.text = it.toString() }

{% endhighlight %}

## Comparison and conclusion

Before the refactoring discussed in the present article the code looked like this:

{% highlight kotlin %}

timer.secondsWithLiveData(vm.seconds).observe(this) { seconds.text = it.toString() }
timer.minutesWithLiveData(vm.minutes).observe(this) { minutes.text = it.toString() }

avm.seconds.observe(this) { stable_seconds.text = it.toString() }
avm.minutes.observe(this) { stable_minutes.text = it.toString() }

{% endhighlight %}

I don't know which version looks more readable. The advantage of using the `observe()` function instead of the `invoke()` operator is such that it is consistent with the syntax of the `observe()` function already present in Java:

{% highlight java %}

public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer)

{% endhighlight %}

The purpose of this article has been to demonstrate that you may create a DSL to combine RxJava LiveData, not that you should.

I do not have a preference yet for either syntax, as I haven't used the DSL yet in any other project. I wrote this article mostly to demonstrate how you could create a simple DSL, not to suggest that the DSL is suitable for this particular use case.

[previous-article]: https://syrop.github.io/jekyll/update/2019/03/22/timers-rxjava-viewmodel.html
[timer]: https://github.com/syrop/Timer

