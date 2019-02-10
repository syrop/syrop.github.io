---
layout: post
title:  "Lazy initialization of ViewModel (simple topic)"
date:   2019-02-10 20:42:00 +0100
categories: jekyll update
---

This article will explain how to initialize a `ViewModel` using the simplest syntax that has occurred to me so far.

This is the follow-up to [another one][TextInputEditText] in which I already wrote about lazy initialization of `ViewModel`.

This time I believe I found a simple solution. I use it in my projects [Victor-Events][victor-events] and [Wiktor-Navigator][navigator].

When you apply this solution, you will only have to write the following code to initialize a `ViewModel` lazily:

{% highlight kotlin %}

private val eventsModel by viewModel<EventsViewModel>()

{% endhighlight %}

Use the following code to create an extention method for `FragmentActivity`:

{% highlight kotlin %}

inline fun <reified R : ViewModel> FragmentActivity.viewModel() = object : LazyDelegate<R> {
    override fun provideDelegate(receiver: Any?, prop: KProperty<Any?>) = lazy {
        ViewModelProviders.of(this@viewModel).get(R::class.java)
    }
}

{% endhighlight %}

Use the this code to create an extension method for `Fragment`:

{% highlight kotlin %}

inline fun <reified R : ViewModel> Fragment.viewModel() = object : LazyDelegate<R> {
    override fun provideDelegate(receiver: Any?, prop: KProperty<Any?>) = lazy {
        ViewModelProviders.of(this@viewModel.activity!!).get(R::class.java)
    }
}

{% endhighlight %}

Even though both extensions functions look similar, one function cannot call the other, which was my chosen solution when I used about initialization of `ViewModels`s [previously][TextInputEditText].

This time the `viewModel` function is called on `Fragment` immediately and returns a lazy delegate that only later is used to create an instance of `ViewModel`. `activity` is `null` at that stage, so you cannot just call `activity!!.viewModel`, or it would throw a `KotlinNullPointerException'.

Please note that you cannot refer to thus created instance of `ViewModel` in your `onCreateView()` function, because `activity` still equals `null` at this stage of the `Fragment`s lifecycle, but you can comfortably refer to it in `onViewCreated()`. 

## In classes other than Activity and Fragment

When you want to gain control over the moment when your `ViewModel` is created, you can use the form I used in my class `InteractiveMapHolder`.

The class itself is described in the advanced topic '[Selecting location][interactive]', although the code I paste below is more up-to-date:

{% highlight kotlin %}

private lateinit var viewModel: CreateEventViewModel

override fun withFragment(fragment: Fragment): MapHolder {
    super.withFragment(fragment)
    with (fragment) {
        val viewModel by viewModel<CreateEventViewModel>()
        this@InteractiveMapHolder.viewModel = viewModel
    }
    return this
}

{% endhighlight %}

You can call the above function at the time you want to initialize your `MapHolder` with an instance of `Fragment`, and your instance of `ViewModel` will be created immediately.

[victor-events]: https://github.com/syrop/Victor-Events
[navigator]: https://github.com/syrop/Wiktor-Navigator
[TextInputEditText]: https://syrop.github.io/jekyll/update/2019/01/17/TextInputEditText-and-LiveData.html
[interactive]: https://syrop.github.io/jekyll/update/2019/01/09/selecting-location-advanced-topic.html

