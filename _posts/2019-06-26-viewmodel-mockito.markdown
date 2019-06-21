---
layout: post
title:  "Testing ViewModel with Mockito"
date:   2019-06-20 13-38 +0200
categories: jekyll update
---

This article explains how to refactor the code so that is uses dependency injection instead of dependency retrieval.

## The problem

I had my [project][compass] recently reviewed by a professional who pointed out to me that I use dependency retrieval (as opposed to dependency injection) too much. They were specifically referring to the `ViewModel` I use.

I've decided to address the problem in a [commit] I document in the present article.

The new implementation of the `ViewModel` shouldn't be aware of the depencency injection/retrieval framework at all, and instead have its dependencies injected thru a constructor.

In future episodes I may introduce testing of the `ViewModel` with Mockito. However, in the present article I only focus on refactoring the `ViewModel` itself, without testing it yet.

## Source materials

There are two pieces of documentation I used to learn how to perform this refactoring. One is a more generic [explanation of the differences between injection and retrieval][injection-retrieval]. The other is an [android-specific documentation][viewmodel] on how to create a suitable `ViewModel`.

The [doco][injection-retrieval] warns againts using injection (as opposed to retrieval) too often:


> When dependencies are **injected**, the class is provided its dependencies at construction.  
When dependencies are **retrieved**, the class is responsible for getting its own dependencies.

> Using dependency **injection** is a bit more cumbersome, but your classes are "pure": they are unaware of the dependency container. Using dependency **retrieval** is easier (and allows more tooling), but it does binds your classes to the Kodein API.

> (...)

>If you are developing a library, then you probably should use dependency **injection**, to avoid forcing the users of your library to use Kodein as well.  
If you are developing an application, then you should consider using dependency **retrieval**, as it is easier to use and provides more tooling. 

I've decided to use injection anyway, at least in the parts of the code I am considering testing in the furure. I do retain dependency retrvieval in these parts of the code that have a lower priority in testing.

## The bindings

This is the `ViewModel` binding I use:

```kotlin
bind<CompassViewModel>() with provider { CompassViewModel(instance(), instance()) }
```

Please note that I didn't use any binding for `CompassViewModel` previously, as I was just using the parameterless constructor, which is invoked by default anyway when the default `ViewModelProvider.Factory` is used. This time I had to create a binding with the `provider` shown above, so that each time a new instance is created, with this particular constructor, whenever the `ViewModelProvider` decides it is needed.

## ViewModelProvider.Factory

This is the universal inmplementation of `ViewModelProvider.Factory` I use:

```
object InjectedVMFactory : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T =
            Kodein.global.direct.Instance(TT(modelClass))
}
```

I haven't decided yet whether I want to use an `object`, shown above, and therefore refer to it without having to use `brackets()`, or a `class`, and create its instance by calling `InjectedVMFactory()`. The reader is welcome to send me their suggestions to the article by [submitting a ticket][ticket] to this blog.

This is the way I create the `ViewModel` with the above `Factory`:

```kotlin
private val viewModel
        by navGraphViewModels<CompassViewModel>(R.id.nav_graph) { KodeinVMFactory }
```

## Conclusion

The present article has shown how one can refactor a class to use dependency injection instead of dependency retrieval.

I performed this change in order to use the `ViewModel` without having to reconfigure Kodein when the desired dependencies change. In future episodes I may may use the currently introduced change to implement testing.

## Donations

If the reader has enjoyed the present article, they may want to donate some bitcoin at the address presented below. Readers may also look at my [donations page][donations].

BTC: bc1qncxh5xs6erq6w4qz3a7xl7f50agrgn3w58dsfp

[compass]: https://github.com/syrop/Compass
[commit]: https://github.com/syrop/Compass/commit/395058b90d2c3582739d2cfa12f7238b038f6540
[injection-retrieval]: https://kodein.org/Kodein-DI/?6.2/core#_injection_retrieval
[viewmodel]: https://kodein.org/Kodein-DI/?6.2/android#view-model-factory
[ticket]: https://github.com/syrop/syrop.github.io/issues
