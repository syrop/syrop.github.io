---
layout: post
title:  "Repository and DSL (refactoring)"
date:   2019-04-29 20:22:00 +0200
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

I hope I don't have to explain how `LiveData` works. I will just focus on what I did to launch the observation. This is the function that combines the repository with the `CoroutineScope` and invokes the blok each time there is a change, as long as the `CoroutineScope` hasn't been canceled:

{% highlight kotlin %}

infix fun vm(vm: ViewModel) = { block: () -> Unit ->
    vm.viewModelScope.launch { this@LiveRepository(block) }
}

{% endhighlight %}

The above `infix` function retrieves the `CoroutineScope` from the `ViewModel`, and captures it into a lambda it returns. The lambda is then launched insine the `ViewModel` with another lambda:

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

## Conclusion

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



[repository-article]: https://syrop.github.io/jekyll/update/2019/04/14/repository-lifecycle.html
[refreshing-article]: https://syrop.github.io/jekyll/update/2019/04/27/refreshing-your-data.html
[victor-events]: https://github.com/syrop/Victor-Events
[commit]: https://github.com/syrop/Victor-Events/commit/0343b1149a38a928b68bc1396de6d791316d2979

