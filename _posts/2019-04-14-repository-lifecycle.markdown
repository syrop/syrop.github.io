---
layout: post
title:  "Repository (design pattern) and lifecycle"
date:   2019-04-14 14:00:00 +0200
categories: jekyll update
---

This article explains how to configure a repository so that it notifies its observers whenever its data changes.

## The problem

1. The repository shall return normal objects and their collections, as opposed to only data wrapped in `LiveData`.
2. It shall also contain different properties and operators.
3. Other code can request such data from the repository.
4. The repository doesn't know anything about the view.

I am using a different approach from the one presented in [the article][mvvm-article] in this blog, which explains the [MVVM pattern][mvvm], as my `ViewModel` there held all of the relevant data wrapped in instances of `LiveData`. This time `LiveData` will be only used to prompt the view to ask the repository for the data it wants to display.

## The project

The project I am using it in is [Victor-Events][victor-events], but because it is still work in progress, and it is already rather large now, you will probably want to just focus on the code presented in the present article instead of cloning the whole project.

## The LiveRepository

This is the code of the base class that handles notifying the view:

{% highlight kotlin %}

abstract class LiveRepository {

    private val liveData = MutableLiveData<Unit>()

    protected fun notifyDataSetChanged() =
            if (Looper.getMainLooper().thread === Thread.currentThread()) liveData.value = Unit
            else liveData.postValue(Unit)

    fun observeDataSetChanges(owner: LifecycleOwner, observer: () -> Unit) =
            liveData.observe(owner) { observer() }
}

{% endhighlight %}

I made it `abstract` as opposed to `open`, so that no instances of it may be created. The name of the field `liveData` is arbitrary, because it is `private` anyway. It will not be seen by the child class, so the child class may even have a field or a property of the same name, and it won't matter.

The advantage of this particular construction is such that `notifyDataSetChanged()` is protected, so that the view that is using the data from this repositor, or even updating it, is not responsible for issuing a notification about the changes.

Please note that the condition that checks whether the currently running thread is the main thread uses a tripple equals `===` operator, which checks whether its two arguments are the same instance, as opposed to calling the `equals()` function.

I chose to check what the current thread is, because I wanted to avoid calling `postValue()` unnecessarily. It enters a `synchronized` section, which may slow down the performance of the app, and is not needed at all when the execution is already on the main thread.

The `liveData` in the code above initially has no value set to it. As a result of that, if you start observing the repository before any data is inserted to it, your view will not be updated at all for a while. To fix that, you may want to pass a `Unit` value to the `liveData` field in the constructor of the `LiveRepository`, for example by calling `notifyDataSetChanged()`. It will assure that any view that chooses to observe this `LiveRepository` will be notified immediately, and will have a chance to display some 'empty data' sign. Alternatively, in every view that uses this repository, you may show a progress bar by default, and take it down when the `LiveRepository` notifies about the changes for the first time.

## The child class

This are the select lines of the class that extends the `LiveRepository`:

{% highlight kotlin %}

class Comms : LiveRepository() {

    private val commCache = mutableListOf<Comm>()

    infix fun delete(comm: Comm) = commCache.remove(comm)
            .also { if (it) notifyDataSetChanged() }

    infix fun join(comm: Comm) = (!commCache.contains(comm) && commCache.add(comm))
            .also { if (it) notifyDataSetChanged() }

    fun addAll(comms: Collection<Comm>) {
        commCache.addAll(comms)
        notifyDataSetChanged()
    }
}

{% endhighlight %}

I use the function `also()`, because I want the most important part of each function, adding and removing data, to be the most noticeable in the code. Alternatively, I could write:

{% highlight kotlin %}

fun updateSomething(): Boolean {
    val result: Boolean = ...
    if (result) notifyDataSetChanged()
    return result
}

{% endhighlight %}

I think that the particular syntax I chose over the alternative shown above is more concise, and therefore more legible. Both are correct, though.

## The view

These are the relevant lines of the view that is using the repository being discussed in the present article:

{% highlight kotlin %}

override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)

        comms.observeDataSetChanges(this) {
            if (comms.isEmpty) {
                comms_view.visibility = View.GONE
                prompt.visibility = View.VISIBLE
            }
            else {
                comms_view.visibility = View.VISIBLE
                comms_view.adapter!!.notifyDataSetChanged()
                prompt.visibility = View.GONE
            }
        }

        comms_view.swipeListener { position ->
            val comm = comms[position]
            comm.leave()
            Snackbar.make(
                    comms_view,
                    getString(R.string.comm_list_leave).bold(NAME_PLACEHOLDER, comm.name),
                    Snackbar.LENGTH_LONG)
                    .setAction(R.string.comm_list_undo) { comm.join() }
                    .show()
        }
    }

{% endhighlight %}

`comms_view` in the above snippet is a `RecyclerView`. The above code sets a listener on swiping a particular item - this deletes the item from the repository and immediately shows a `Snackbar` giving the user a chance to undelete it.

The most noticeable piece of code in this `onActivityCreated()` function is the invocation:

{% highlight kotlin %}

comms.observeDataSetChanges(this) { ... }

{% endhighlight %}

The abve incocation creates a listener that will be invoked each time data in the repository is updated. It will be invoked the first time on the entry to the `Fragment` - provided that the repository already contains some data. If you are not sure of it, you may want to configure your layout file to present some dummy view while the repository loads its data for the first time. Alternatively, in you implementation of the repository you may want to initialize it with some dummy data when the app starts, and immediately call the `protected` function `notifyDataSetChanged()`, so that every view using that repository always has some data to display.

The invocations `comm.leave()` and `comm.join()` also call `notifyDataSetChanged()` internally, but you may end up with too many notification if you want to quickly add several items in succession. To remedy that you may want to implement a `commit()` function that commits the changes to the repository, and only then notifies the observers, or you may implement a `vararg` function that adds all the items passed to it and only then issues a notification.

## Conclusion

When you want to present in a view data that is stored using the repository design pattern, you should never have to call a function like `updateViews()` or `refreshScreen()` directly from your `Fragment` each time you are adding or deleting data.

Even if you did want to call such a function, you would never know whether your repository updates synchronously, or uses an asynchronous mechanism like coroutines or RxJava. Depending on the implementation of your repository, you would then have to use an `Observable`, a `suspend` function, or a callback to update your view.

Using the `LiveRepository` base class proposed in the present article assures that you have one lifecycle-aware way of registering an observer that is always triggered on the main thread.


[mvvm-article]: https://syrop.github.io/jekyll/update/2019/04/06/mvvm.html
[mvvm]: https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel
[victor-events]: https://github.com/syrop/Victor-Events


