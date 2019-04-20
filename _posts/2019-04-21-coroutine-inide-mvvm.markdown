---
layout: post
title:  "Coroutine inside a ViewModel"
date:   2019-04-21 00:52:00 +0200
categories: jekyll update
---

This is an article on how to launch a coroutine so that it is aware of the parent's lifecycle.

The parent of this particular coroutine will be a `ViewModel` - as opposed to an `Activity` or `Fragment`. The advantage is such that the coroutine will survive screen orientation, although it will we canceled when the `ViewModel` is cleared.

## Difficulty level of this article

In this article I am discussing a piece of code that is meant to go into production.

I tie together several design patterns I described in a number of my recent articles. Some of the other propositions I made then might even be obsolete by now, which I shall point out when I address them here.

The article summarizes my current understanding of Kotlin, and its main advantage over Java - coroutines. I believe that coroutines break Kotlin's inoperability with Java, and once you start applying them, you might not want to, or not be able to return to Java.

At the time of writing this I am actively looking for a job, and I see in many advertisements requirements that require being able to use Java as the primary language, or require knowledge of libraries that bridge Java and Kotlin - RxJava and Dagger.

If you want, you may implement in Java a factory that supports lazy initialization, you may manually check whether everything is `null` to ascertain `null` safety, or you can manually implement the `equals()` function in every class that holds your data, so that you don't have to use Kotlin's `data` classes.

At one point, though, one wonders whether it makes sense to implement manually the things that are available in another language for free - or use bridging dependencies, such as Dagger, RxJava or Guava.

By the means of this article I am trying to promote the view that Kotlin is alright on its own, it's mature enough, and can contribute enough value to your project. Please consider requiring from your candidates to be able to use Kotlin as the primary language of a project, or add competency in coroutines to your core requirements.

## The problem

This article tries to answer how to run a concurrent task so that it survives screen rotation.

When the screen is rotated, the `Fragment` shall check whether there is a task running, and display a progress bar if it is. If a result has already been found, the `Fragment` shall not launch the task again, but display the result immediately.

## Previous relevant articles

In the current article I am presenting a refactoring of a searching solution already described [previously][searching-article] in this blog. The previous solution had a bug - when the screen was rotated, results of the search were lost, and the user had to run the search again. Thanks to running a coroutine in the container of a `ViewModel` this is no longer the case.

I described an extremely similar solution in [an article][asynctask-article] about running an `AsyncTask` inside a `ViewModel`. The other article was using a demo project, though, because I did not intend to use an `AsyncTask` in production. This time you have a chance to see a more mature solution based on coroutines, combined with other useful techniques that you might use in a commercial project.

## Source materials

I learned how to use coroutines from a talk given at last year's conference [Android Dev Summit][android-suspenders]. It took me a couple of weeks to thoroughly study that particular YouTube video. This and some of my other recent articles in this blog have been inspired by this talk.

## The project

The project I am discussing in the present article is [Victor-Events][victor-events], but because it is still work in progress, and it is already rather large now, you will probably want to just focus on the code presented in the present article instead of cloning the whole project.

I recommend using **Android Studio 3.5 Canary 12**, because it comes with preinstalled Kotlin 1.3.30.

If you want to open the project in **Android Studio 3.4**, which is currently the latest stable version, you need to replace this line in your project-level `build.gradle`:

{% highlight groovy %}

classpath 'com.android.tools.build:gradle:3.5.0-alpha12'

{% endhighlight %}

with:

{% highlight groovy %}

classpath 'com.android.tools.build:gradle:3.4.0'

{% endhighlight %}

## The suspend function

This is the `suspend` function that reads data from Firebase Cloud Firestore:

{% highlight kotlin %}

private suspend fun DocumentReference.read(): DocumentSnapshot = suspendCancellableCoroutine { continuation ->
    get().addOnCompleteListener { result ->
        if (result.isSuccessful) {
            continuation.resume(result.result!!)
        }
        else {
            continuation.resumeWithException(result.exception!!)
        }
    }
}

{% endhighlight %}

`get()` is a function that returns `com.google.android.gms.tasks.Task`. The above code then adds a listener on it that resumes the coroutine when there is a result. It is crucial that it you put it inside of the block passed to `suspendCancellableCoroutine()` function, as opposed to `suspendCoroutine()`. If you use the latter, it will still return the correct result, but you won't be able to cantel the coroutine, so it may return a result of a search that is no longer relevant, and try to display that result on the screen.

This is, subsequently, a `suspend` function that tries to find a community, given its name:

{% highlight kotlin %}

suspend fun findCommunity(name: String) = coroutineScope {
    val lcName = name.toLowerCase()
    val deferredComm = async {
        communities.document(lcName).read().toCommunity()
    }
    val isAdmin = async {
        if (isLoggedIn) isAdmin(lcName) else false
    }
    val comm = deferredComm.await()

    if (comm.isDummy) comm else comm.copy(isAdmin = isAdmin.await())
}

{% endhighlight %}

The above code calls two `suspend` functions: `read()`, which has already been discussed, and `isAdmin()`, which uses the same function `read()` internally. Because both of them should run concurrently, they are called inside of their respective `async` operation.

Notice the `coroutineScope` invocation at the top of the function body. It allows the code to inherit the `CoroutineScope` the whole function is run in.

Thanks to that, and because I used `suspendCancellableCoroutine` when I defined the `read()` function, every corouitine inside of `findCommunity()` will be canceled when the `CoroutineScope` itself is canteled.

The following section will show how to run a `suspend` function in a `CoroutineContext` of its container `ViewModel`. Thank to that every coroutine will be canceled when the `ViewModel` itself is cleared.

## The viewmodel

These are the relevant lines of the `ViewModel` that is calling this coroutine:

{% highlight kotlin %}

class MainViewModel : ViewModel() {
    val queryState by lazy { MutableLiveData<QueryState>().apply { value = QueryState.None }}
    private var queryJob: Job? = null

    var pastDestination = 0

    fun query(name: String) {
        queryState.value = QueryState.WorkInProgress
        queryJob = viewModelScope.launch(Dispatchers.IO) {
            fsReader.findCommunity(name).let {
                queryState.postValue(QueryState.Comm(it))
            }
        }
    }

    fun resetQuery() {
        queryJob?.cancel()
        queryState.value = QueryState.None
    }

    sealed class QueryState {
        object None : QueryState()
        data class Completed(val comm: Comm) : QueryState()
        object InProgress : QueryState()
    }
}

{% endhighlight %}

In has a function to start the coroutine and to cancel it. It has an `Int` property to remember the id of the `View` that is currently displayed. It defines a `sealed` class for representing the state.

By default the state is `None`, to inform the `Fragment` that no work has been yet done, and that there is no result. It also can be `WorkInProgress`, or contain a representation of the found community.

## The Activity

The following code is the function which initiates the search whenever it receives an `ACTION_SEARCH` `Intent`. The exact searching mechanism has been described in a [separate article][searching-article] in this blog.

{% highlight kotlin %}

override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    if (Intent.ACTION_SEARCH == intent.action) {
        mainModel.query(intent.getStringExtra(SearchManager.QUERY))
    }
}

{% endhighlight %}

Because `query()` uses `viewModelScope` internally, all associated coroutines will be canceled only when the `ViewModel` is cleared. It will survive screen orientation changes, when the `Activity` that initiated the search is itself destroyed.

This is the listener resposible for canceling the search when the appropriate `Fragment` disappears from the screen:

{% highlight kotlin %}

navController.addOnDestinationChangedListener { _, destination, _ ->
    when (mainModel.pastDestination) {
        R.id.addCommFragment -> mainModel.resetQuery()
    }
    mainModel.pastDestination = destination.id
}

{% endhighlight %}

I couldn't just cancel the search on pressing [Back] or [Home]. When the found community is joined (or when it is not found, but a new one is created), the `Fragment` is automatically removed from the screen, and then also the state of the query needs to be set to the default `None`.

(Handling [Back] and [Home] keys, which is used in the same [project][victor-events] for dismissing data manually entered in forms, is described in a [dedicated article][back-article] in this blog).

## DSL

I did a little refactoring of the [project][victor-events] to support `+` operator both for `LiveData` and `MutableLiveData`. These is the relevant function:

{% highlight kotlin %}

operator fun <T> LiveData<T>.plus(owner: LifecycleOwner) =
        if (this is MutableLiveData<T>) MutableHotData(this, owner)
        else DefaultHotData(this, owner)

{% endhighlight %}

I have to support both the mutable and immutable variant because I use `MutableLiveData` to support filling in data in `TextInputEditText`, as described in a [separate article][textinputedittext-article] in this blog.

These are the classes and interfaces that get created when you write `liveData + this`:

{% highlight kotlin %}

interface HotData<T> {
    val liveData: LiveData<T>
    val owner: LifecycleOwner

    operator fun invoke(observer: (T) -> Unit) = liveData(owner, observer)
}

data class DefaultHotData<T>(override val liveData: LiveData<T>, override val owner: LifecycleOwner): HotData<T>

data class MutableHotData<T>(override val liveData: MutableLiveData<T>, override val owner: LifecycleOwner): HotData<T>

{% endhighlight %}

In the case of just observing the data it doesn't matter which instance is created, as the interface's default implementation of `invoke()` operator is going to be used either way, but if you need to seamlessly set values of your `MutableLiveData` after you've combined it with LifeCycleOwner, you probably do need to use a special case od the `data` class, here: `MutableHotData`.

Again, combinig `MutableLifeData` with `LifecycleOwner` using the `+` operator, and then using it to back data entered in form, is described in a [separate article][textinputedittext-article].

Using `invoke()` operator to observe the `LiveData` is going to be described in the following section.

## The Fragment

This is the observer that observes the progress of the coroutine and its result:

{% highlight kotlin %}


(eventsModel.queryState + this) { result ->
    when (result) {
        is MainViewModel.QueryState.InProgress -> {
            recycler.visibility = View.GONE
            prompt.visibility = View.GONE
            progress.visibility = View.VISIBLE
        }
        is MainViewModel.QueryState.Completed -> searchResult(result.comm)
    }
}

{% endhighlight %}

The above code invokes `eventsModel.querystate + this` to create an instance of `HotData` and then calls `invoke()` operator on it to start observing the values. It would have worked the same way whether `eventsModel.queryState` was mutable or not.

The class `MainViewModel.QueryState` doesn't have to `sealed`. Because `when` in the above invocation doesn't return a value, it doesn't need to be exhaustive. Still, I decided to use the `sealed` keyword in the definition of the class, because the code might be more easy to maintain when it is already designed the possibility to used it exhautive expressions.

In the case when the `result` is `QueryState.InProgress`, the `when` statement just brings the progress bar to the front by appropriately modifying the visibility property of this and other views. When it is `QueryState.Completed`, it just calls a function that displays the found community, or prompts the user to create it if it hasn't been found.

The design of the `searchResult()` function is arbitrary, though. It checks whether the community has been found or not. If it has, displays it on the screen and allows joining it, if it has not been found in the database, prompts the user to create it. If is not aware of `MainViewModel.QueryState`, and I designed it before I even had coroutines in mind. I designed that function before I knew about coroutines, and haven't refactored it much since. I was alredy using a version of it at the time when I was writing the [article][searching-article] about implementing Search functionality.

## Summary

In the present article I`ve summarised over a month worth of research into the MVVM design pattern.

This and several other recent articles in this blog have been inspired by the [talk][android-suspenders] given at Android Dev Summit 2018.

In this article you have had a chance to learn how to run a coroutine inside of a `ViewModel` so that it survives screen orientation change, but is canceled when another `Fragment` is brought to the screen.

You have also seen how to use some DSL to observe `LiveData`. Not that I particularly like DSL, though. DSL is nothing more than a glorified aggregation of lambdas. I like creating my own ones from time to time, still, just to see whether I am able to match the level of DSL design presented at professional confernces I watch on YouTube.

The dominant theme of this article has been how to refactor the code that was already presented in some of the other articles here to compel your code to become less error-prone and more concise. By employing these techniques you may or may not reduce the cost of Android softwar development in your company.


The point of the present article shall be to aid the reader do gauge whether Kotlin is a mature language that can be used on its own to promote concurrent programming, lifecycle handling, data retention and many more.



[victor-events]: https://github.com/syrop/Victor-Events
[searching-article]: https://syrop.github.io/jekyll/update/2018/12/26/architecture-components-and-searching.html
[asynctask-article]: https://syrop.github.io/jekyll/update/2019/04/01/asynctask-inside-viewmodel.html
[android-suspenders]: https://www.youtube.com/watch?v=EOjq4OIWKqM
[back-article]: https://syrop.github.io/jekyll/update/2019/04/11/backhome.html
[textinputedittext-article]: https://syrop.github.io/jekyll/update/2019/04/12/backing-textinputedittext-with-livedata-dsl.html
