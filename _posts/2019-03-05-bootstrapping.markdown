---
layout: post
title:  "Bootstrapping"
date:   2019-03-05 18:19:00 +0100
categories: jekyll update
---

This is an article about how to initialize (bootstrap) an Android app written in Kotlin.

It will also give hints on how to configure a [Kodein][kodein] module for dependency injection.

The code used in this article is presently used in my project [Victor Events][victor-events].

## Application class

This is the entire code of the `Application` class used in the project:

{% highlight kotlin %}

class EventsApplication : Application() {

    init {
        Kodein.global.addImport(module)
    }

    override fun onCreate() {
        super.onCreate()
        bootstrap.boot()
    }

    fun login(user: FirebaseUser) = bootstrap.login(user)

    fun logout() = bootstrap.logout()
}

{% endhighlight %}

## Kodein module

The first few lines of the `Application` class create and import a [Kodein][kodein] module implemented in this way:

{% highlight kotlin %}

val Context.module get() = KodeinModuleBuilder(this).build()

inline fun <reified R : Any> instance() = Kodein.global.instance<R>()

...

class KodeinModuleBuilder(private val ctx: Context) {

    fun build() = Kodein.Module("main") {
        bind<Bootstrap>() with singleton { Bootstrap() }
	...
        bind<Geocoder>() with singleton { Geocoder(ctx, Locale.getDefault()) }
        ...
    }
}

{% endhighlight %}

As you see, there is an extention property defined for `Context`, so that when I use the property `module` inside `Application` class, an instance of the module is created, and the application context is passed to it.

Thus passed application `Context` us then used to create, for example, binding for [`Geocoder`][geocoder], because [`Geocoder`][geocoder] needs to have `Context` passed in its constructor.

## Bootstrap

This is the code of the `Bootstrap` class:

{% highlight kotlin %}

val bootstrap by instance<Bootstrap>()

class Bootstrap {

    fun boot() {
        login.setCurrentUser(FirebaseAuth.getInstance().currentUser)
        db.commDao.getAllAsync { comms.addAll(it) }
        db.messageDao.getAllAsync { messages.addAll(it) }
    }

    fun login(user: FirebaseUser) {
        login.setCurrentUser(user)
    }

    fun logout() {
    }
}

{% endhighlight %}

The file first defines a global constant `bootstrap` that is retrieved lazily when the function `onStart()` inside `Application` class calls:

{% highlight kotlin %}

bootstrap.boot()

{% endhighlight %}

It also defines two other functions (`login()` and `logout()`) that can be called at any moment from the `Application` class. The functions may be removed in future versions, because they are not clearly related to bootstrapping. I put them there, because I did not want them to be global, and this particular class seemed to be appropriate for containing all the code that should be called from `Application` class, but I may choose a different solution later.

## Conclusion

This article showed where to put code that should be run when the application starts, but you do not want to put it directly in the code of `Application`'s `onCreate()` function.

It also showed how to capture application `Context` using [Kodein][kodein] and lazy dependency retrieval.

Lastly, it also showed that although the `Bootstrap` class may be also used to hold other functions that you do not want to be global, and for example you may want them to have access to some values captured by `Bootstrap`, but it may be more appropriate to put them elsewhere. For example, you may want to create a class called `Architecture`, and retrieve its instance depending on your product's flavor or hardware.

[kodein]: http://kodein.org/Kodein-DI/
[victor-events]: https://github.com/syrop/Victor-Events
[geocoder]: https://developer.android.com/reference/android/location/Geocoder

