---
layout: post
title:  "AsyncTask (a job interview question)"
date:   2019-04-01 12:50:00 +0200
categories: jekyll update
---

I am very often asked during job interviews about `AsyncTask`.

I am not very excited about it, as I know techniques that I think are more suitable for Android development than AsyncTask, but still, it must be possible to use an AsyncTask without causing an activity leak and stuff, so I've tried to share with you my present understanding of how use an `AsyncTaks` - and `AsyncTaks` specifically - to run an asynchronous process on Android.

## The project

The project I use in this example is called - appropriately - [AsyncTask][asynctaksk]. It is a copy of my project [MyApplication][myapplication], with all its history wiped, and with some extension functions copied from [Victor-Events][events].

## The ViewModel

This is the `ViewModel` I use in this example. It contains one `LiveData` to display the progress, one to display the result, and a flag that indicates whether an AsyncTask is alerady running:

{% highlight kotlin %}

class CounterModel : ViewModel() {
    val progress by lazy { MutableLiveData<Int>() }
    val result by lazy { MutableLiveData<Int>() }

    var isAsyncTaskRunning = false
}

{% endhighlight %}

## The Builder

This is the code that builds an instance of the `AsyncTaks`:

{% highlight kotlin %}

class Counter() : AsyncTask<Int, Int, Int>() {

    private var result: MutableLiveData<Int>? = null
    private var progress: MutableLiveData<Int>? = null

    private constructor(result: MutableLiveData<Int>?, progress: MutableLiveData<Int>?): this() {
        this.result = result
        this.progress = progress
    }
    
    class Builder {
        var result: MutableLiveData<Int>? = null
        var progress: MutableLiveData<Int>? = null

        fun build() = Counter(result, progress)
    }

    companion object {
        fun build(block: Builder.() -> Unit) = Builder().apply(block).build()
    }
}

{% endhighlight %}

## Creating an instance

This is how a `Fragment` creates an instance of `Counter` with its `Builder`:

{% highlight kotlin %}

val counter = Counter.build {
    progress = vm.progress
    result = vm.result
}

{% endhighlight %}

## First attempt: coroutines

This is how I was trying to implement the code of the `AsyncTaks`s method `doInBackground()` using coroutines:

{% highlight kotlin %}

override fun doInBackground(vararg params: Int?): Int {
    val repetitions = params[0]!!
    GlobalScope.launch(Dispatchers.IO) {
        repeat(repetitions) {
            publishProgress(it)
            delay(1000L)
        }
    }
    return repetitions
}

{% endhighlight %}

The problem with this approach was such that the method would immediately return the result, while the progress would be still updated for the next couple of seconds.

I was briefly considering returning `Unit` in the above example and just interact with my `Fragment` using just the function `publishProgress()` to tell the parent when the task is already completed, but then, it would be just too tempting to just do away with `AsyncTaks` completely, and put the entire process inside a coroutine.

The job interview question is specifically about `AsyncTask`, though, and I really wanted to prepare an answer for it that would take advantage of the `onPostExecute()` function, so I decided to simplify the above code and use threads instead of coroutines.

## Second attempt: threads

This is the implementation of the function `doInBackground()` that is really present in my [repository][asynctask] at the time of writing:

{% highlight kotlin %}

override fun doInBackground(vararg params: Int?): Int {
    val repetitions = params[0]!!
    repeat(repetitions) {
        publishProgress(it)
        Thread.sleep(1000L)
    }
    return repetitions
}

{% endhighlight %}

The disadvantage of this approach is that I do not have control on where the `Thread.sleep()` is executed. If I was using RxJava, I would use `Schedulers.io()`, and if I was usirg coroutines, I would use `Dispatchers.IO`, as I showed previously.

Still, the question was about using `AsyncTask`, and this is the simplest solutions I could come up with.

I should probably also register an observer on the `Activity`'s lifecycle, so that when the entire screen (as opposed to a single `Fragment`) gets destroyed, the `AsyncTaks` gets canceled, but this example is tailored towards a short telephone interview, so I can only put in it as much information as I hope to easily memorize.

## UI thread

These are the standard functions that are called in the IO thread:

{% highlight kotlin %}

override fun onProgressUpdate(vararg values: Int?) {
    val progress = values[0]!!
    this.progress?.value = progress
}

override fun onPostExecute(result: Int?) {
    this.result?.value = result
}

{% endhighlight %}

Because `AsyncTask` guarantees that these two functions always run on the UI thread, I can use `this.result?.value = result`.

If I wasn't sure that this line was going to be called from the UI thread, I would instead have to write: `this.result?.postValue(result)`, as only then would I be assured that the `Observer` is called on the correct thread (the UI thread).

## Launching the AsyncTask

This is the code inside the `Fragment`'s function `onActivityCreated()`:

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

    if (!vm.isAsyncTaskRunning) {
        val counter = Counter.build {
            progress = vm.progress
            result = vm.result
        }
        vm.isAsyncTaskRunning = true
        counter.execute(REPETITIONS)
    }
}

{% endhighlight %}

## Bold section (just for fun)

Just for fun I made the views prettier by adding some bold letters. This is the extension function I used:

{% highlight kotlin %}

fun String.bold(placeholder: String, replacement: String): CharSequence {
    val idName = indexOf(placeholder)
    val idEndName = idName + replacement.length
    val boldSpan = StyleSpan(Typeface.BOLD)
    return SpannableStringBuilder(replace(placeholder, replacement)).apply {
        setSpan(boldSpan, idName, idEndName, Spanned.SPAN_INCLUSIVE_EXCLUSIVE)
    }
}


{% endhighlight %}

## Conclusion

I frankly admire the interviewers for still asking the candidates about the `AsyncTak`.

I remember reading in an article somewhere that `AsyncTask`s were prone to activity leaks or something, but as I've hopefully demonstraded with the above example, Android Jetpack helps to mitigate such dangers, even in APIs as ancient as `AsyncTask`.

I think that the purpose of asking this question during an interview is verifying whether the candidate is aware of traditional Android APIs (AsyncTask has been present in Android since API 3, HoneyComb), and whether they can use more modern solutions to mitigate the risks they involve.

I wouldn't be using the above code in production, though. I've used above the `Thread.sleep()` construction, which I would much rather replace with something from coroutines or RxJava. I've tested my solutions by rotating or turning off the screen while the `AsyncTask` was running, and I noticed that the screen was updating correctly, with no crashes. I haven't properly run memory profiling, though, and I didn't implement canceling the operation.

Still, by writing the above code I satisfied my curiosity on whether Jetpack's `ViewModel` and `LiveData` mitigates old dangers of `AsyncTaks`, and you too should be prepared to answer questions about combining several different APIs.

[asynctask]: https://github.com/syrop/AsyncTaks
[myapplication]: https://github.com/syrop/MyApplication
[events]: https://github.com/syrop/Victor-Events

