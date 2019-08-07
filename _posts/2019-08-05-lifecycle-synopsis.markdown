---
layout: post
title:  "Lifecycle (synopsis)"
date:   2019-08-05 12:00:00 +0200
categories: jekyll update
---

This is an article about different way of hadlling lifecycle in GUI.

Frequently I am asked at job interviews about `Activity` lifecycle.

I am not sure what does listing these six bulletpoints matter:

* onCreate()
* onStart()
* onResume()
* onPause()
* onStop()
* onDestroy()

I've been trying to ask interviewers about the meaning of `onStart()` vs `onResume()` and `onStop()` vs `onPause()`, as I've never seen all of them being used simultaneously in a project. Specifically, I wanted to find out what code to put in each of them.

The answers I received from the interviewers, however, usually only addressed what happens in Android during each particular stage - not what problems I can solve by overriding each particular function and whether I should override all of them.

In the present article I am trying to answer how to use lifecycle to solve different categories of problems.

## Reflecting the state in the UI

In my projects I do not use data binding, so discussig it is beyond the scope of the present article.

The contents of the present section may become obsolete when I introduce [Jetpack Compose][jetpack] to my projects. Jetpack Compose should solve many problems connected to displaying the state, and responding to user input.

The present implementation of `android.view.View` is 27753 lines long, so using `View` already precludes use of [single responsibility][single-responsibility-principle] principle, and perhaps other [SOLID] principles.

Still, I am trying to make the code as it is. The present article contains only a synopsis of handling lifecycle. The reader may find a more complete description of the patterns MVVM and MVP [elsewhere][mv] in the blog.

In the blog I used to write a couple of articles on combining `LiveData` with `RxJava`. I do not use any of these solutions presently, as I am trying to shy away from RxJava. There is notthing inherently wrong with RxJava, but I fail to see advantages of using RxJava over coroutines, which are present natively in Kotlin.

Still, if I was combining RxJava with `LiveData`, I would probably use a code similar to this:

```kotlin
class RxLiveData<T>(private val observable: Observable<T>) : LiveData<T>() {
        
    private lateinit var disposable: Disposable
        
    override fun onActive() {
        super.onActive()
        disposable = observable.subscribe { postValue(it) }
    }

    override fun onInactive() {
        super.onInactive()
        disposable.dispose()
    }
}

```

The above code is written using RxJava 2.

Depending what happens in `Obsevrable`'s `doOnSubscribe()` and `doOnDispose()`, it may be wise to refrain from using the above `onActive()` and `onInactive()` as these are called each time the app is paused. More accurately speaking, `onActive()` happens in `onStart()`, and `onInactive()` happens in `onStop()`.

If the developer wants to have a greater control over the `Observable`, they may want to override lifecycle functions manually, and instead of `onStart()` and `onStop()` use `onCreate()` and `onDestroy()`.

Because I do not use RxJava very often, I am not sure whether the above code should use `lateinit var disposable: Disposable` or rather `var disposable: Disposable? = null`. If the reader thinks they know a more suitable version of the code, please [contact me][ticket], and I will consider changing the article, with due credit to the author of the ticket.

## Performing asynchronous work off main thread

Another category of problems that are solved by correctly handling lifecycle is performing background work (off main thread) that needs to be canceled when either `Activity`, `Fragment` or `ViewModel`.

The preceeding section already discussed some of the ways lifecycle may be handled in RxJava, so I am not going to discuss it presently. I am trying to move away from using RxJava in favor of coroutines, which are native to Kotlin, and in my opinion allow writing a more concise code.

I am not keeping up with changes in RxJava introduced in [RxJava 3.0.0-RC1][rxjava3], so I am not really sure what the presently recommended way of handling lifecycle in RxJava is.

The code discussed in the present section may require the following dependencies:

```groovy
implementation 'androidx.lifecycle:lifecycle-extensions:2.2.0-alpha02'
implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.2.0-alpha02'
implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.2.0-alpha02'
```

Launching work that is meant to be canceled when `Activity` or `Fragment` is gone may be performed with the following code:

```kotlin
lifecycleScope.launch { ... }
```

or:

```kotlin
lifecycleScope.async { ... }
```

Writing the actual coroutine is trivial, so it is beyond the scope of the present article. The reader may, however, find many examples of using coroutines in production-ready open source projects in many articles in this blog.

For the sake of the present article it is sufficient to say that coroutines launched this way by default run on main thread, but they can be easily switched to another thread by using a relevant `CoroutineDispatcher`, which may be swithed at will, many times, even in a single block:


```kotlin
lifecycleScope.launch {
    ...
    withContext(Dispatchers.IO) {
        ...
    }
    withContext(Dispatchers.Default) {
        ...
    }
    withContext(Dispatchers.IO) {
        ....
    }
    ...
}
```

The above code may be used, for instance, to display a *please wait* message on the main thread, read data from disk, process it, write results to disk, and display a *well done* message again on the main thread.

To launch a piece of work inside a `ViewModel`, as opposed to inside a `Fragment`, very similar code may be used, but instead of `lifecycleScope` the developer will write the name `viewModelScope`.

Performing work inside `ViewModel` is further discussed in [another article][work] in this blog.

## Deferring work until future lifecycle events

Coroutine may de delayed until a particular lifecycle by using the following code:

```kotlin
lifecycleScope.launch {
    whenCreated { ... } // defer until lifecycle.State.CREATED
    whenStarted { ... } // defer until lifecycle.State.STARTED
    whenResumed { ... } // defer until lifecycle.State.RESUMED
}
```

## Combining asynchronous wokr with displaying state

The reader may refer to [previously mentioned article][work] if they want to read about performing complex tasks inside `ViewModel`.

The present section talks about performing coroutines inside `LiveData`, but the work that is being discussed here is less complex than the kind of work discussed in the [previous article][work].

This is the function that creates `LiveData` performing a coroutine internally:

```kotlin
liveData {
    try {
       ...
    }
    finally {
        ...
    }
}
```

The code inside the `try` block is performed as soon as the `LiveData` has at least one observer. When the `LiveData` looses the last observer, and timeout elapses, the `finally` block is executed.

If the `LiveData` doesn't require clearing up, the `try` and `finally` keywords may be omitted, so that the entire lambda passed to `liveData` constitutes the code that is run as lond as there is at least one observer, and there has been no timeout.

`liveData` is actually one of my very favorite functions in Kotlin, and in all of Android, so I have already dedicated a few articles to it. The reader is recommended to read the articles about [combining `LiveData` with `Channel`][channel], and [testing `LiveData`][testing] in this blog.

## Conclusion

Discussing lifecycle is not trivial.

I am personally responsible for spreading some confusion when I wrote a couple of articles about combining `LiveData` with RxJava, promoting design patterns that I soon discarded when I learned about coroutines.

I am worried that when I am asked about lifecycle next time in a job interview, I will not have a ready answer. Writing just this article required me to spend a considerable amount of time, doing some research and compiling some code. I might not be able to repeat all of this information during a phone interview, but having written it all once in a concise article may significantly help me.

If you are an interviewer, and you are planning to ask this question, please note that a discussion of lifecycle handling may span a couple of programming and testing techniques. It definitely took me writing more than one sophisticated article. When you are designing your interview question, please make it as succinct as possible, such as:

> How to display 'please wait' and after a while 'well done'?

## Donations

If the reader has enjoyed the article, they might want to donate some bitcoin at the address presented below.

BTC: bc1qncxh5xs6erq6w4qz3a7xl7f50agrgn3w58dsfp

Readers may also look at my [donations page][donate].

[compose]: https://developer.android.com/jetpack/compose/
[single-responsibility-principle]: https://en.wikipedia.org/wiki/Single_responsibility_principle
[solid]: https://en.wikipedia.org/wiki/SOLID
[mv]: https://syrop.github.io/jekyll/update/2019/06/30/mv-disambiguation.html
[ticket]: https://github.com/syrop/syrop.github.io/issues
[rxjava3]: https://github.com/ReactiveX/RxJava
[work]: https://syrop.github.io/jekyll/update/2019/04/20/coroutine-inide-mvvm.html
[channel]: https://syrop.github.io/jekyll/update/2019/06/04/wrapping-channel-in-livedata.html
[testing]: https://syrop.github.io/jekyll/update/2019/06/21/viewmodel-mockito.html
[donate]: https://syrop.github.io/donate/

