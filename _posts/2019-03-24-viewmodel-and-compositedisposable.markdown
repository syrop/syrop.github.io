---
layout: post
title:  "ViewModel and CompositeDisposable"
date:   2019-03-23 21:23:00 +0100
categories: jekyll update
---

In this article I further describe my work on the demonstration project [Timer][timer].

## Changes since last episodes

I got rid of some of the elements of the DSL introduced in the [last episode][last-episode], as they were too confusing for me.

The main reason I write the present article is that I discovered a way to handle RxJava's `Disposable`s directly inside the `ViewModel` that scopes the entire `Activity`. Thanks to that change I do not have to write any code in the `Activity` or `Fragment` that would initiate updating the `LiveData`.

## What hasn't changed

I still demonstrate the use of two `ViewModel`s that are `Fragment`-specific. Changes described in the present article do not apply to them, as I want to retain their form close to the [original][first-episode], for the sake of comparison. 

## The new ViewModel

This is the base class of the `ViewModel` that is being discussed in the present article:

{% highlight kotlin %}

open class RxViewModel : ViewModel() {
    private val cd = CompositeDisposable()

    protected fun <T> disposableLiveData(observable: Observable<T>): Lazy<LiveData<T>> = lazy {
        MutableLiveData<T>().apply {
            cd.add(observable.subscribe { this.postValue(it) })
        }
    }

    override fun onCleared() {
        cd.dispose()
    }
}

{% endhighlight %}

It creates one private instance of `CompositeDisposable` that is disposed when the `ViewModel` is cleared. It also has one `protected` function that lazily creates an instance of `MutableLiveData`. It requires passing an instance of `Observable` and subscribes to it.

Please remember to define the return type of `disposableLiveData()` as `Lazy<LiveData<T>>`. By doing this you hide the real type of the `LiveData()` created here, and therefore it will be seen as immutable by the code that is using it.

This is the way in which the above class is extended:

{% highlight kotlin %}

class Timer {

    private val timer = Observable.interval(1, TimeUnit.SECONDS, Schedulers.io())

    private val seconds = BehaviorSubject.createDefault(0)
    private val minutes = BehaviorSubject.createDefault(0)

    val secondsObservable: Observable<Int> = seconds.delay(DELAY_MS, TimeUnit.MILLISECONDS)
    val minutesObservable: Observable<Int> = minutes.delay(DELAY_MS, TimeUnit.MILLISECONDS)

    ...

    class ViewModel : RxViewModel() {
        val seconds by disposableLiveData(timer.secondsObservable)
        val minutes by disposableLiveData(timer.minutesObservable)
    }
}

{% endhighlight %}

The `ViewModel` initiates lazily two instances of `LiveData` and creates a `CompositeDisposable`. The rest of the code of the `Timer` is described in the [first episode][first-episode] of this series.

## Observing

This is the way in which the new `ViewModel` is observed. Please note that the below snippet also, for comparison, shows the line of the code that do not use the design pattern proposed by the present article:

{% highlight kotlin %}

class FirstFragment : Fragment() {

    private val vm by viewModel<ViewModel>()
    private val tvm by viewModel<Timer.ViewModel>()

    ...

    override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)
        next_fab.setOnClickListener { nav(R.id.action_firstFragment_to_secondFragment) }

        (timer seconds vm.seconds).observe(this) { seconds.text = it.toString() }
        (timer minutes vm.minutes).observe(this) { minutes.text = it.toString() }

        tvm.seconds.observe(this) { stable_seconds.text = it.toString() }
        tvm.minutes.observe(this) { stable_minutes.text = it.toString() }
    }

    class ViewModel : androidx.lifecycle.ViewModel() {
        val seconds by lazy { MutableLiveData<Int>() }
        val minutes by lazy { MutableLiveData<Int>() }
    }
}

{% endhighlight %}

## Donations

If you've enjoyed this article, consider donating some bitcoin at the address below. You can also look at my [donations page][donations].

BTC: bc1qncxh5xs6erq6w4qz3a7xl7f50agrgn3w58dsfp 

[last-episode]: https://syrop.github.io/jekyll/update/2019/03/23/rxjava-lifecycle-dsl.html
[first-episode]: https://syrop.github.io/jekyll/update/2019/03/22/timers-rxjava-viewmodel.html
[timer]: https://github.com/syrop/Timer
[donations]: https://syrop.github.io/donate/

