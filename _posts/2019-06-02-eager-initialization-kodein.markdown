---
layout: post
title:  "Eager initialization for Kodein"
date:   2019-06-02 15:43 +0200
categories: jekyll update
---
This article explains  to use [Kodein] eagerly. By default it works lazily. Until now I haven't been able to use it to retrieve a value eagerly, that is without using the keyword `val` or `var`.

Previously, in order to immediately use a value inside a piece of code instead of having to initialize it lazily, I created a sugar function that first initialized a value eagerly, then immediately returned it:

```kotlin
val Any.log: Logger get() {
    val result by instance<String, Logger>(this::class.java.name)
    return result
}
```

## The project

The project I am using this solution in is [Victor Events][victor-events]. The example I use is creating a per-class instance of `Logger`, which was discussed in detail in another [article][logging] in this blog.

## The binding

This is the Kodein binding I created for `Logger`. Nothing has changed since the [last article][logging], other than adding a `@Suppress` annotation for the sake of suppressing a warning:

```kotlin
bind<Logger>() with multiton { tag: String ->
            Logger.getLogger(tag)!!.apply {
                if (!BuildConfig.DEBUG) {
                    @Suppress("UsePropertyAccessSyntax")
                    setFilter { false }
                }
            }
        }
```

This is the code that retrieves (lazily) an instance of the multiton. Again, nothing has changed in it:

```kotlin
inline fun <reified A, reified T : Any> instance(arg: A) = Kodein.global.instance<A, T>(arg = arg)
```

## Delegated properties

To understand my solution, you may want to see the [documentation][delegated-properties] of delegated properties.

Normally I would just use the `value` property of [`Lazy`][lazy] delegate, because it forces the value to initialize and immediately returns it. However, because Kodein uses the `provideDelegate()` function, it doesn't give me an instance of [`Lazy`][lazy] immediately. Instead, I have to request it. This is the way `provideDelegate()` is defined:

```kotlin
operator fun provideDelegate(receiver: Any?, prop: KProperty<Any?>): Lazy<V>
```

`provideDelegate()` takes in two parameters, only one of which may be `null`.

This is the Kodein-specific implemenation:

```kotlin
override fun provideDelegate(receiver: Any?, prop: KProperty<Any?>): Lazy<V> = lazy {
        @Suppress("UNCHECKED_CAST")
        val context = if (receiver != null && originalContext === AnyKodeinContext) KodeinContext(TTOf(receiver) as TypeToken<in Any>, receiver) else originalContext
        get(context, true) } .also { trigger?.properties?.add(it)
    }
```

When the first parameter is `null`, Kodein just uses the default `KodeinContext`, so let's leave it that way. The second parameter is ignored, so I can pass any value at all, just making sure I do not pass `null`.

I need to pass any value of `KProperty` as the second parameter. [Reflection documentation][reflection] explains how to create one. In order to do it, I only have to use an arbitrary pre-existing value with the `::` operator. This is how I do it:

```kotlin
inline val <T> KodeinProperty<T>.value get() =
        provideDelegate(null, Build::ID).value
```

The `provideDelegate()` function returns an instance of `Lazy`, on which I call `.value` to immediately retrieve the value.

## Usage

This is the code I use to retrieve an instance of `Logger`, using the class name as a parameter passed to Kodein:


```kotlin
val Any.log: Logger get() =
        instance<String, Logger>(this::class.java.name).value
```

The above invocation eagerly retrieves an instance from Kodein by first requesting a Lazy delegate and then calling `value` on it. It works by calling the extension property `KodeinProperty<T>.value` discussed in the preceeding section.

Finally, this is the example use in the actual code of [Victor Events][victor-events]:

```kotlin
log.info("firebaseAuthWithGoogle:" + acct.id!!)
```

The above retrieves an instance of `Logger` using the extention property `Any.log`. Because I configured Kodein to use a a multiton binding for `Logger`, only one instance will be created per class, and the `Logger`'s tag will be initiated with that class name. Also, unless the application is a debug build, no logs will be produced 

## Conclusion

In the present article I explained how I refactored a solution described [previously][logging] in this blog, which can be used to retrieve a per-class instance of `Logger`.

I found this solution more by analyzing the code of `Kodein` than by reading a particular piece of documentation, so the code presented herein might be less than optimal. If you think you know of a better solution, please contact me by [submitting a new issue][issues] on GitHub. Thank you.

[kodein]: https://kodein.org/Kodein-DI/?6.2/core
[victor-events]: https://github.com/syrop/Victor-Events
[logging]: https://syrop.github.io/jekyll/update/2018/12/30/logging.html
[delegated-properties]: https://kotlinlang.org/docs/reference/delegated-properties.html
[lazy]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-lazy/index.html
[reflection]: https://kotlinlang.org/docs/reference/reflection.html
[issues]: https://github.com/syrop/syrop.github.io/issues
