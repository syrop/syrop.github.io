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
    override fun provideDelegate(receiver: Any?, prop: KProperty<Any?>) = lazy { provideViewModel<R>() }
}

inline fun <reified R : ViewModel> FragmentActivity.provideViewModel() =
        ViewModelProviders.of(this).get(R::class.java)

{% endhighlight %}

Use the this code to create an extension method for `Fragment`:

{% highlight kotlin %}

inline fun <reified R : ViewModel> Fragment.viewModel() = object : LazyDelegate<R> {
    override fun provideDelegate(receiver: Any?, prop: KProperty<Any?>) = lazy { provideViewModel<R>() }
}

inline fun <reified R : ViewModel> Fragment.provideViewModel() = activity!!.provideViewModel<R>()

{% endhighlight %}

[victor-events]: https://github.com/syrop/Victor-Events
[navigator]: https://github.com/syrop/Wiktor-Navigator
[TextInputEditText]: https://syrop.github.io/jekyll/update/2019/01/17/TextInputEditText-and-LiveData.html

