---
layout: post
title:  "Repository and DSL (refactoring)"
date:   2019-04-30 06:08:00 +0200
categories: jekyll update
---

This article shows how how to watch changes in your repository using a DLS.

Alternatively, you could listen to the updates in real time, but this technique is not going to be discussed in this article.

## Previous articles

Repository pattern was introduced in [one of the previous articles][repository-article] in this blog.

Then, in the [next article][refreshing-article], I explained how to use a `CoroutineWorker` to refresh its data periodically.

The two previous articles already explained how to use both coroutines and `LiveData` to watch the changes.

The present article explains how to use a simple higher order function, so that you can make your `ViewModel` observe the changes in your repository, using just this simple `init` block:

{% highlight kotlin %}

init { (comms vm this) { comm = comms[comm.name] } }

{% endhighlight %}

## The project

The project I am using in this article as an example is [Victor-Events][victor-events], but because it is still work in progress, and it is already rather large now, you will probably want to just focus on the code presented in the article instead of cloning the whole project.

## The problem

Write a ViewModel that performs the following actions each time the data is updated in the repository:

1. Pulls out one particular item from the repository.
2. Wraps the relevant data concerting that item (two `String`s and a `Boolean`) in a few instances of `LiveData`.

## The ViewModel

This is the `ViewModel` that solves the problem. Each time data is updated in the repository it pulls out of it one item and uses it to update values of three instances of `LiveData`:

{% highlight kotlin %}

class CommViewModel : ViewModel() {

    init { (comms vm this) { comm = comms[comm.name] } }

    var comm = Comm.DUMMY
    set(value) {
        field = value
        reset()
    }
    val name by lazy { MutableLiveData<String>() }
    val desc by lazy { MutableLiveData<String>() }
    val isAdmin by lazy { MutableLiveData<Boolean>() }

    fun reset() {
        name.value = comm.name
        desc.value = comm.desc
        isAdmin.value = comm.isAdmin
    }
}

{% endhighlight %}

I hope I don't have to explain how `LiveData` works. I will just focus on what I did to launch the observation.

This is the function that captures the `ViewModel` into a lambda:

{% highlight kotlin %}

infix fun vm(vm: ViewModel) = { block: () -> Unit ->
    vm.viewModelScope.launch { this@LiveRepository(block) }
}

{% endhighlight %}

The above `infix` function returns a lambda that is launched inside the `ViewModel` with another lambda:

{% highlight kotlin %}

{ comm = comms[comm.name] }

{% endhighlight %}

This is, in turn, used by the `invoke()` operator inside the `LiveRepository`:

{% highlight kotlin %}

suspend operator fun invoke(block: () -> Unit) {
    with (channel.openSubscription()) {
        if (!isEmpty) receive()
        while(true) {
            receive()
            block()
        }
    }
}

{% endhighlight %}

What the above code does it firsts opens a subscription on the `BroadcastChannel`. It checks whether the channel already contains data, and removes it if it does. (It doesn't remove the value from the parrent `BroadcastChannel`). Then, as long as the `CoroutineScope` hasn't been canceled it waits for subsequent values and calls invokes the `block`.

## Recap

The present refactoring allowed me to shorten the original line:

{% highlight kotlin %}

init { viewModelScope.launch { comms { comm = comms[comm.name] } } }

{% endhighlight %}

to this:

{% highlight kotlin %}

init { (comms vm this) { comm = comms[comm.name] } }

{% endhighlight %}

I didn't even have to create a `data` class that could, otherwise, hold references to both the `LiveRepository` and the `ViewModel` (or its `CoroutineScope`). All that I did was I created a function that returns a lambda.

The code snippets presented above are probably not enough for the reader to understand the entire mechanism I used there. The reader is welcome to read about the `LiveRepository` pattern in the two articles referred to in the introduction, or clone the [entire project][victor-events].

The reader might also use their browser to view the changes discussed in the present particle in their corresponding [commit][commit].

## Fixing a memory leak

Don't forget to close the `Channel`s! The above code snippets contain a simplified code that may lead to `Channel` leaks. This may happen because I open a new `Channel` for every `ViewModel` that wants to take advantage of the repository. Notice the code of `ConflatedBroadcastChannel` I use:

{% highlight kotlin %}

@Suppress("UNCHECKED_CAST")
public override fun openSubscription(): ReceiveChannel<E> {
    val subscriber = Subscriber(this)
    _state.loop { state ->
        when (state) {
            is Closed -> {
                subscriber.close(state.closeCause)
                return subscriber
            }
            is State<*> -> {
                if (state.value !== UNDEFINED)
                    subscriber.offerInternal(state.value as E)
                val update = State(state.value, addSubscriber((state as State<E>).subscribers, subscriber))
                if (_state.compareAndSet(state, update))
                    return subscriber
            }
            else -> error("Invalid state $state")
        }
    }
}

{% endhighlight %}

Implementation of the function `openSubscription()` creates a `Subscriber` and stores a reference to it by calling `addSubscriber()`. The reference is only removed when the `Subscriber` cancels.

This is the corrected version of my code, which cancels the `Channel` when the relevant `CoroutineScope` is canceled, which happens always when the correspoding `ViewModel` is cleared:

{% highlight kotlin %}

suspend operator fun invoke(block: () -> Unit) {
    with (channel.openSubscription()) {
        if (!isEmpty) receive()
        try {
            while (true) {
                receive()
                block()
            }
        } finally { cancel() }
    }
}

{% endhighlight %}

When the `ViewModel` is cleared, and its corresponding `CoroutineScope` is canceled, the above function `receive()` will throw a `CancellationException`. It will prevent further execution of any code in the `try` block, so that the `Channel` created during the the invocation of `openSubscription()` may be only canceled in the `finally` block.

The reader might want to refer to the [commit][fix-commit] corresponding to the above fix.

[repository-article]: https://syrop.github.io/jekyll/update/2019/04/14/repository-lifecycle.html
[refreshing-article]: https://syrop.github.io/jekyll/update/2019/04/27/refreshing-your-data.html
[victor-events]: https://github.com/syrop/Victor-Events
[commit]: https://github.com/syrop/Victor-Events/commit/0343b1149a38a928b68bc1396de6d791316d2979
[fix-commit]: https://github.com/syrop/Victor-Events/commit/c3a1da294f6e757e0b03f163f69dd9ff54d36c32
