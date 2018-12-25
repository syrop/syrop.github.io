---
layout: post
title:  "Lifecycle-aware Rx subscriptions"
date:   2018-12-25 10:52:19 +0100
categories: jekyll update
---
In this article I will explain how to use RxJava subscriptions so they will automatically unsubscribe on lifecycle events, for example when the relevant activity stops.

In the article I use code snippets from the project [Wiktor-Navigator][navigator].

The following code handles lifecycle events. It is present in `navigator/src/main/kotlin/pl/org/seva/navigator/main/Rx.kt`:



{% highlight kotlin %}

fun <T> Observable<T>.subscribe(lifecycle: Lifecycle, onNext: (T) -> Unit) =
        subscribe(onNext).observeLifecycle(lifecycle)

private fun Disposable.observeLifecycle(lifecycle: Lifecycle) =
        lifecycle.addObserver(RxLifecycleObserver(lifecycle, this))

private class RxLifecycleObserver(
        private val lifecycle: Lifecycle,
        private val subscription: Disposable) : LifecycleObserver {
    private val initialState = lifecycle.currentState

    @Suppress("unused")
    @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
    private fun onEvent() {
        if (!lifecycle.currentState.isAtLeast(initialState)) {
            subscription.dispose()
        }
    }
}

{% endhighlight %}

In the above code there is one global `subscribe` function that takes in a `lifecycle` instance and a lambda expression as parameters.

It then creates the subscription by calling the original `subscribe` fuctnion form RxJava, and calls a private extenstion method `observeLifecycle` on it to react to lifecycle events.

The method `observeLifecycle`, in turn, creates an instance of a class extending `LifecycleObserver`, which is here called `RxLifecycleObserver`, and passes `lifecycle` itself to it, as well as the `subscription` created in the previous step.

Then, the `RxLifecycleObserver` in its own constructon remembers the initial state the lifecycle is in to compare it with subsequent states.

When the state becomes **less than** the initial state, (for example when `Activity#onStop() onStop` was called if the initial state was `CREATED`, or when `Activity#onPause() onPause` was called if the initial state was `STARTED`), the subscription is automatically disposed.

## Calling

To use the lifecycle-aware function, just find in your code all calls of the `subscribe` function, and add the parameter `lifecycle` to it. For example this code:

{% highlight kotlin %}

observable.subscribe {
    println("Hello, World!")
}

{% endhighlight %}

becomes:

{% highlight kotlin %}

observable.subscribe(lifecycle) {
    println("Hello, World!")
}

{% endhighlight %}

You may have to manually import the `subscribe` extenston function, for example if the function is located in the file `navigator/src/main/kotlin/pl/org/seva/navigator/main/Rx.kt`, you import it like this:

{% highlight kotlin %}

import pl.org.seva.navigator.main.subscribe

{% endhighlight %}

## Other extensions

You may want to write other extension function, to create lifecycle-aware versions of the original RxJava's `subscribe` function:

{% highlight kotlin %}

fun <T> Observable<T>.subscribe(lifecycle: Lifecycle, onNext: (T) -> Unit, onError: (Throwable) -> Unit) =
        subscribe(onNext, onError).observeLifecycle(lifecycle)

fun <T> Observable<T>.subscribe(lifecycle: Lifecycle, onNext: Action1<in T>) =
        subscribe(onNext).observeLifecycle(lifecycle)

fun <T> Observable<T>.subscribe(lifecycle: Lifecycle, onNext: Observer<in T>) =
        subscribe(onNext).observeLifecycle(lifecycle)

{% endhighlight %}

They all call the private `observeLifecycle` function presented in one of the code snippets above, so you can create as many versions of the `subscribe` function as you like.

## More advanced implementation

In the remaining portion of the article I will describe how to write code that will be execuded only if the lifecycle is in the appropriate state, **without using the above extensions**.

You can use this technique if you do not really want to dispose the whole subscription if the activity is being destroyed. You may want to use it, for example, to encapsulate only the fragments of code that do depend on the activity.

{% highlight kotlin %}

@MainThread
inline fun <reified T>LifecycleOwner.inLifecycle(crossinline f: (T) -> Unit) =
    MutableLiveData<T>().apply {
        observe(this@inLifecycle, Observer<T>  { f(it) })
    }

@MainThread
operator fun <T> MutableLiveData<T>.invoke(param: T) {
    value = param
}

@MainThread
operator fun MutableLiveData<Unit>.invoke() {
    value = Unit
}

{% endhighlight %}

Usage of `@MainThread` annotation in the above code is optional, it depends on how you want to call it.

The operator `invoke()` and `invoke(param: T)` cause the passed value (or `Unit` in the parameterless version) to be set to an instance of `LiveData`.

Because of the way `LiveData` works, no code will be executed if the `lifecycle` is not in the correct state.

You then call it this way:

{% highlight kotlin %}

val onSuccess = inLifecycle<Unit> {
    	println("This is called depending on the lifecycle")
}

val onError = inLifecycle<Throwable> {
	ErrorMessageHelper.showError(activity, it)
}

observable.subscribe { response ->
	if (response.isError) {
		val throwable = response.error()
		throwable.printStackTrace()
        	onError(throwable)  // won't perform any action if not in the correct state
        }
        else {
		println("This executes independently of the lifecycle state")
		onSuccess()  // won't perform any action if not in the correct state
	}
}

{% endhighlight %}

The method `inLifecycle` needs to be called **specifically outside of the** `subscribe` **block**, as it then starts observing the `LiveData`, and therefore may cancel the observation, **before the** `subscribe` **block is called**, so that any instances of `Activity` or other lifecycle-dependent classes will not be retained when they are no longer relevant.

Depending on your situation you can use the `inLifecycle` method to create code that depends or does not depend on passed parameters.

You state the parameter type in angle brackets, if you want to use one, or just write `<Unit>` if you do not need it.

To call the code you use the parameterless `()` operator, defined in the code as `operator fun MutableLiveData<Unit>.invoke()`, or the parameterized version, which in the snippet above is called as `onError(throwable)`, and is defined as `operator fun <T> MutableLiveData<T>.invoke(param: T)`.


[navigator]: https://github.com/syrop/Wiktor-Navigator

