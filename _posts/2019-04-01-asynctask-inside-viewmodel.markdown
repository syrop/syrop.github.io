---
layout: post
title:  "AsyncTask inside a ViewModel (advanced answer)"
date:   2019-04-01 23:22:00 +0200
categories: jekyll update
---

This is to demonstrate how to fit an instance of `AsyncTask` running inside a `ViewModel`. I wouldn't be using it in production, but you can use it during a more advanced job interview.

This time the instance of the AsyncTask is created inside a `ViewModel` and properly canceled.

This may remind you of the refactoring I did in the article '[ViewModel and CompositeDisposable][viewmodel-and-disposable]', in which I created an RxJava `Disposable` - and properly disposed it - inside a `ViewModel`, so that the `Disposable` didn't have to be handled inside an `Activity` or a `Fragment`.

## The previous article

You may not understand the present article unless you have read the [previous one][previous] in which I discuss why I decided to discuss `AsyncTask` in the first place.

## The ViewModel

This is the code of the `ViewModel` containing an instance of the `AsyncTask`:


{% highlight kotlin %}

class CounterModel : ViewModel() {

    val progress by lazy { MutableLiveData<Int>() }
    val result by lazy { MutableLiveData<Int>() }

    private val task = Counter.build {
        progress = this@CounterModel.progress
        result = this@CounterModel.result
    }

    init {
         task.execute(REPETITIONS)
    }

    fun cancel() = task.cancel(false)

    override fun onCleared() {
        cancel()
    }

    companion object {
        const val REPETITIONS = 10
    }
}

{% endhighlight %}

## The Fragment

This is the code creating the observations inside the `Fragment`. It also adds a button canceling the `AsyncTask`.

Please note that even if the canceling button is not pressed at all, the taks will be also canceled when the `ViewModel` is cleared, due to the overridden function `onCleared`.

This is the code that builds an instance of the `AsyncTaks`:

{% highlight kotlin %}

override fun onActivityCreated(savedInstanceState: Bundle?) {
    super.onActivityCreated(savedInstanceState)

    vm.progress.observe(this) {
        progress.text = getString(R.string.counter_fragment_progress)
                .bold(PROGRESS, it.toString() )
    }
    vm.result.observe(this) {
        result.text = getString(R.string.counter_fragment_result)
                .bold(RESULT, it.toString())
    }
    cancel.setOnClickListener { vm.cancel() }
}

{% endhighlight %}

## The AsyncTask

This is the code running in the `AsyncTask`, away from the UI thread:

{% highlight kotlin %}

override fun doInBackground(vararg params: Int?): Int {
    val repetitions = params[0]!!
    runBlocking {
        repeat(repetitions) {
            if (isCancelled) {
                return@repeat
            }
            publishProgress(it)
            delay(1000L)
        }
    }
    return repetitions
}

{% endhighlight %}

Because I use a `repeat()` function that takes a lambda, I have to use the expression `return@repeat` to break it. If I was using a `for` loop, I would write `break` instead.

In the above code I had to use the function `runBlocking()` to obtain a scope in which I could run the `suspend` function `delay()`. In the [previous article][previous] I demonstrated a failed attempt to do the same with with `launch()`. It didn't work, because `launch()` returns immediately, therefore `onPostExecute()` would be run before the counting even ends. Here I use `runBlocking()` which blocks the current thread until the block it contains is completed. The disadvantage of `runBlocking()` and `AsyncTask` in general is such that on Android all `AsyncTaksk` are executed in one thread, and because this particular `AsyncTask` takes about 10 seconds to run, no other `AsyncTask` should be executed simultaneously on the device. In the real case, if this code was meant to go to production, I would probably skip using an `AsyncTask` and use some other coroutine scope or RxJava instead.

## Conclusion

In this article you have learned how to refactor the code from the [previous article][previous] to allow for canceling the task - both from the UI button, and in reaction to lifecycle events.

In the real job interview I usually try to talk about the `AsyncTask` as little as possible, and try to suggest talking about RxJava or coroutines instead, but there you have it. If you want to have a working model of an `AsyncTask` that probably doesn't lead to memory leaks, and properly responds to lifecycle events, you see my proposition.

The code used in the present article is at the time of writing it available in my [repository][asynctask].

[viewmodel-and-disposable]: https://syrop.github.io/jekyll/update/2019/03/23/viewmodel-and-compositedisposable.html
[previous]: https://syrop.github.io/jekyll/update/2019/04/01/asynctask.html
[asynctask]: https://github.com/syrop/AsyncTaks

