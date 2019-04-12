---
layout: post
title:  "Backing TextInputEditText with LiveData (DSL)"
date:   2019-04-12 19:18:00 +0200
categories: jekyll update
---

This article demonstrates how to back a TextInputEditText with `MutableLiveData` using a DSL.

It is a follow-up to the [previous article][previous-article] in whith I was solving a problem stated in the next section. This time I am solving it with a DSL, although the underlying mechanism is identical.

## The problem

The problem is how to save all changes made in a form (here: TextInputEditText) immediately into a `MutableLiveData<String>`, so that it survives screen orientation changes, switching `Fragment`s and so on.

Once your `TextInputEditText` form fields are backed by instances of `MutableLiveData<String>` you may refer to them in your `AppCompatActivity` to react to [Back] and [Home] button in the way presented in a [dedicated article][back-home] in this blog.

## The project

The project I am using it in is [Victor-Events][victor-events], but because it is still work in progress, and it is already rather large now, you will probably want to just focus on the code presented in the present article instead of cloning the whole project.

## Introducing a new type

I am introducing a new data type, which I called `HotData`, that connects `MutableLiveData` with `LifecycleOwner`:

{% highlight kotlin %}

data class HotData<T>(val liveData: MutableLiveData<T>, val owner: LifecycleOwner)

operator fun <T> MutableLiveData<T>.plus(owner: LifecycleOwner) = HotData(this, owner)

operator fun <T> LifecycleOwner.plus(liveData: MutableLiveData<T>) = HotData(liveData, this)

{% endhighlight %}

The above `HotData` is created with the `+` operator by using it either way; `MutableLiveData` on the left and `LifecycleOwner` on the right, or the other way round.

## += operator

This is the operator that performs the actual backing:

{% highlight kotlin %}

operator fun TextInputEditText.plusAssign(hotData: HotData<String>) = withLiveData(hotData.owner, hotData.liveData)

{% endhighlight %}

It is invoked this way:

{% highlight kotlin %}

name += model.name + this

{% endhighlight %}

(`model.name` in the above code snippet is an instance of `MutableLiveData<String>`, and `this` is a `LifecycleOwner`).

It works identical to the following invocation, already discussed in details in the [previous article][previous-article]:

{% highlight kotlin %}

name.withLiveData(this, model.name)

{% endhighlight %}

## Conclusion

I occasionally enjoy playing with DSLs, although they are nothing more that clever operators, lambdas and so on.

I took the particular idea of combining objects with a `+` operator from the [`plus` operator][plus] in `kotlinx.coroutines`.

I am not entirely convinced to using operators and new types instead of regular function calls, but I have already performed the refactoring in all of my forms in the [project][victor-events], and I shall see whether I still like it after a couple of weeks.

[previous-article]: https://syrop.github.io/jekyll/update/2019/01/17/TextInputEditText-and-LiveData.html
[back-home]: https://syrop.github.io/jekyll/update/2019/04/11/backhome.html
[victor-events]: https://github.com/syrop/Victor-Events
[plus]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/plus.html

