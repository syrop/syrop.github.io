---
layout: post
title:  "Timers, RxJava and ViewModel"
date:   2019-03-22 11:27:00 +0100
categories: jekyll update
---

This is the next article from the cycle on how to combine RxJava and `LiveData`.

This time I am not creating a new instance of `LiveData` each time a subscription is made, but using a `ViewModel` to provide that.

I created a [separate repository][timer] for the sake of this article. It shouldn't change much after I publish it, so you will probably always see in it the code as it is discussed here.

In the article I take for granted the reader's knowledge of Android Jetpack and Kodein, as these have been discussed thoroughly in other articles in this blog.

## The project

For the sake of the article I copied the project [MyApplication][my-application]. Because it is my own project, I am not allowed to fork it, so I just created its copy locally on my computer and changed its origin to point to the newly created repository. You can check for yourself that both projects have the same history.

I did that because I have already plenty of libraries imported in my prototype [MyApplication][my-application] project, and I didn't want to copy them line by line. I also already had a Kodein module implemented there, as well as some of my favourite extension functions.

If the [Timer][timer] project was meant to go to production, I would probably have to delete the history and only retain the dependencies and some of the code from my dummy project.

## The Observables

This is the part of the class `Timer` that creares the `Observable`s:

{% highlight kotlin %}

private val timer = Observable.interval(1, TimeUnit.SECONDS, Schedulers.io())

private val seconds = BehaviorSubject.createDefault(0)
private val minutes = BehaviorSubject.createDefault(0)

init {
    timer.subscribe { seconds.onNext(it.toInt()) }
    seconds.filter { it % 60 == 0 }
            .map { it / 60 }
            .subscribe { minutes.onNext(it) }
}

{% endhighlight %}

I've decided to use a `BehaviorSubject` both for seconds an minutes, as it immediately emits the last value once somebody subscribes to it. If I used `PublishSubject` instead, the subscriber would have to wait one second (or one minute) after the subscription in created before it can receive its first value.

This class is not meant to be used as a clock, as the seconds counter is not reset when the minute counter is incremented. Both counters just increase their values forever.

## Combining with LiveData

This is the code that combines the seconds counter with `LiveData`:

{% highlight kotlin %}

fun secondsWithLiveData(liveData: MutableLiveData<Int>) = seconds
        .delay(DELAY_MS, TimeUnit.MILLISECONDS)
        .withLiveData(liveData)

{% endhighlight %}

I've decided to introduce a delay of 200 milliseconds for the sake of the demonstration. Thanks to that you will exactly see the moment when the old value, kept in the `ViewModel` is replaced by the new value, emitted by `BehaviorSubject` when the user navigates to a new fragment. The delay is defined like this:

{% highlight kotlin %}

companion object {
    const val DELAY_MS = 200L
}

{% endhighlight %}

In the above solution the `Subject` is never shared with the outside world. The function `secondsWithLiveData()` also enforces creating an instance of `LiveData` in another class. (In this article the `LiveData` will be provided by a `ViewModel`).

The function `toLiveData()` creates an instance of a data class called `LiveObservable`:

{% highlight kotlin %}

fun <T> Observable<T>.withLiveData(liveData: MutableLiveData<T>) =
    LiveObservable(this, liveData)

{% endhighlight %}

The code of the class `LiveObservable` itself:

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

You may recognize some code already discussed in the [previous episode][rx-and-livedata], namely, adding an instance of `RxLifecycleObserver` to the lifecycle. Here is the `Observer` that gets triggered when the `Activity` or `Fragment` is destroyed:

{% highlight kotlin %}

class RxLifecycleObserver(private val disposable: Disposable) : LifecycleObserver {

    @Suppress("unused")
    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    private fun onEvent() {
        disposable.dispose()
    }
}

{% endhighlight %}

## ViewModel

Below is the code of the `ViewModel` I use. In the project the code is repeated for every fragment, because I want to show a situation in which no data is shared between the fragments, all of is retrieved from the `timer`:

{% highlight kotlin %}

class ViewModel : androidx.lifecycle.ViewModel() {
    val seconds by lazy { MutableLiveData<Int>() }
    val minutes by lazy { MutableLiveData<Int>() }
}

{% endhighlight %}

Here is the way an instance of the `ViewModel` is retrieved in the `Fragment`:

{% highlight kotlin %}

private val vm by viewModel<ViewModel>()

{% endhighlight %}

I discussed retrieving `ViewModel`s lazily in a [separate article][viewmodels].

This is the whole code of the `Fragment`:

{% highlight kotlin %}

class FirstFragment : Fragment() {

    private val vm by viewModel<ViewModel>()

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?) =
            inflate(R.layout.first_fragment, container)

    override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)
        next_fab.setOnClickListener { nav(R.id.action_firstFragment_to_secondFragment) }

        timer.secondsWithLiveData(vm.seconds).observe(this) { seconds.text = it.toString() }
        timer.minutesWithLiveData(vm.minutes).observe(this) { minutes.text = it.toString() }
    }

    class ViewModel : androidx.lifecycle.ViewModel() {
        val seconds by lazy { MutableLiveData<Int>() }
        val minutes by lazy { MutableLiveData<Int>() }
    }
}

{% endhighlight %}

First it sets a navigation action on a floating action button, using Android Jetpack.

Then it combines `BehaviorSubject`s used internally in `Timer` with appropriate instances of `LiveData` retrieved from the ViewModel, to create instances of the data class `LiveObservable` discussed above.

It calls then `observe()` on the `LiveObservable` to create an RxJava's disposable to subsequently dispose it when the fragment gets destroyed. The action passed in the lambda will be called as soon as the RxJava's Observable emits its first value, and because `Timer` uses a `BehaviorSubject` internally, this value will be emitted almost immediately, that is afted the delay of 200 milliseconds I introduced for the sake of the demonstration.

## Running

The project contains two `Fragment`s, `FirstFragment` and `SecondFragment`.

No data is shared between them. Furthermore, the first `Fragment` is never destroyed while the application is on the screen, but the second one is destroy as soon as the user leaves it.

Thanks to that, you will never observe any glitches on the `FirstFragment`. It always displays the most up to date data.

Because the `SecondFragment` stores its data in a `ViewModel`, which is kept in memory during the duration of the `Activity`, even after the `Fragment` is destroyed, you will at some times see stalled data. This is deliberate. Because I introduced the delay of 200 milliseconds in the `Timer`, and because the RxJava's `Disposable` is disposed as soon as the `Fragment` that created it is destroyed, during the first 200 milliseconds after you navigato to `SecondFragment` you will see the old data retrieved from the ViewModel while you wait the data coming from the `Timer`.

## Why use ViewModel at all

In the [previous article][rx-and-livedata] on the topic of `LiveData` I discussed a solution that could also create a reasonably good effect.

Originally I was planning to write the present article only as a demonstration of using RxJava's `BehaviorSubject`. I noticed, though, that even without the deliberately introduced delay of 200 ms, when I was navigating back and forth between both screens, an empty value would be briefly shown it the seconds and minutes views before the UI would react to the first value emitted by the `BehaviorSubject`. It introduced a noticeable flickering. Because of that I decided to keep the value inside of the `ViewModel`, so that at least the old value would be displayed immediately.

## Avoiding stalled data

In the above examples I deliberately created two separate `ViewModel`s, because I wanted to show you a situation in which two `Fragment`s pas two different sets of `LiveData` to the `Timer` independently, and how in this situation the lifecycle of each `Fragment` affects the data that is retrieved.

In production, if you want to share the same data between `Fragment`s, you will probably want to create one `ViewModel` for the whole parent `Activity`, and update the `LiveData` from there, while every `Fragment` will observe the changes individually.

These are the relevant lines that create the and update the `LiveData` from the Activity. This is the version that is currently in my [repository][timer] on GitHub:

{% highlight kotlin %}

class MainActivity: AppCompatActivity() {

    private val vm by viewModel<ActivityViewModel>()

    @SuppressLint("CheckResult")
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        timer.secondsWithLiveData(vm.seconds).observe(this) {}
        timer.minutesWithLiveData(vm.minutes).observe(this) {}
    }

    class ActivityViewModel : androidx.lifecycle.ViewModel() {
        val seconds by lazy { MutableLiveData<Int>() }
        val minutes by lazy { MutableLiveData<Int>() }
    }
}

{% endhighlight %}

Unless you want to perform some action inside the `Activity`, you can just pass an empty lambda into `observe()`, as it is shown above. This is just to update the contents of the `LiveData`. Please note that it violates the [single responsibility principle][srp], as here `observe()` not only passes the `Observer()`, but also initiates updating the `LiveData`. This is not much diffenent, though, from the technique already used in RxJava, where `subscribe()` not only passes the lambda that is triggered on every emission, but often also launches the process being subscribed to.

This is how the `LiveData` retrieved from the `ViewModel` created above is observed in the `Fragment`:

{% highlight kotlin %}

class SecondFragment : Fragment() {

    private val avm by viewModel<MainActivity.ActivityViewModel>()

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?) =
            inflate(R.layout.second_fragment, container)

    override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)

        avm.seconds.observe(this) { stable_seconds.text = it.toString() }
        avm.minutes.observe(this) { stable_minutes.text = it.toString() }
    }
}

{% endhighlight %}

For clarity, from the above snippets referring to the fragment-specific `ViewModel` have been omitted, but when you clone the [repository][timer], you will see that `SecondFragment` uses simultaneously the `ViewModel` inherited from its parent `Activity`, as well as its own, to demonstrate the difference in the data they are showing.

CAUTION: I used above my custom extension function `observe()`. It is not present by default in Kotlin. This is the way in which I implemented it:

{% highlight kotlin %}

fun <T> LiveData<T>.observe(owner: LifecycleOwner, observer: (T) -> Unit) =
        observe(owner, Observer<T> { observer(it) })

{% endhighlight %}

[timer]: https://github.com/syrop/Timer
[my-application]: https://github.com/syrop/MyApplication
[rx-and-livedata]: https://syrop.github.io/jekyll/update/2019/03/18/combining-rxjava-and-livedata.html
[viewmodels]: https://syrop.github.io/jekyll/update/2019/02/10/lazy-initialization-of-viewmodel.html
[srp]: https://en.wikipedia.org/wiki/Single_responsibility_principle
