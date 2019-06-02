---
layout: post
title:  "Eager initialization for Kodein"
date:   2019-06-02 09:59 +0200
categories: jekyll update
---
I think I might have figured out how to use Kodein eagerly.

## The problem

[Kodein][kodein] works lazily by design. Until now I haven't been able to use it to retrieve a value eagerly, that is without using the keyword `val` or `var`.

In order to use a value inside a piece of code, I created a sugar function that first initialized a value eagerly, then immediately returned it:

```kotlin
val Any.log: Logger get() {
    val result by instance<String, Logger>(this::class.java.name)
    return result
}
```

The following sections present a solution that allows shortening the above code to one line.

## The project

The project I am using this solution in is [Victor Events][victor-events]. You can also see the particular [commit] in which I introduced it.

I use the code to retrieve a per-class instance of `Logger`, which was discussed in detail in another [article][logging] in this blog.

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

When the first parameter is `null`, Kodein just uses the default `KodeinContext`, so let's leave it that way. The second parameter is ignored, so I can pass any value at all, just not `null`.

I need to pass any value of `KProperty` as the second parameter. [Reflection documentation][reflection] explains how to create one.

One of the simplest solutions that comes to my mind is to just create some dummy Int value:

```kotlin
val a = 0
```

However, I can avoid allocating any memory at all by simply using a value that is already present in Kotlin at all times. This is what I wrote:

```kotlin
private val unit = Unit
val <T> KodeinProperty<T>.value get() = provideDelegate(null, ::unit).value
```

The [reflection documentation][reflection] does't say that I can create a reference to an object (here: `Unit`) directly. However, I can create any property that is initiated with that object, and create a reference to it using the `::` operator (which automatically creates an instance of `KProperty`).

Because I am not going to use the `unit` property anywhere else in the code, I marked it as `private`.

You can see this particular solution in the [commit] referred to at the beginning of this article.

## Usage

This is the code that eagerly retrieves an instance from Kodein by first requesting a `Lazy` delegate and then calling `value` on it.

```kotlin
val Any.log: Logger get() = instance<String, Logger>(this::class.java.name).value
```

The above line of code works by calling the extension property `KodeinProperty.value` discussed in the preceeding section.

Finally, this is the example use in the actual code of [Victor Events][victor-events]:

```kotlin
log.info("firebaseAuthWithGoogle:" + acct.id!!)
```

The above will retrieve an instance of `Logger` using the extention property `Any.log`. Because I configured Kodein to use a a multiton binding for `Logger`, only one instance will be created per class, and the `Logger`'s tag will be initiated with that class name. Also, unless the application is a debug build, no logs will be produced 

## Conclusion

In the present article I explained how I refactored a solution described [previously][logging] in this blog, which can be used to retrieve a per-class instance of `Logger`.

I found this solution more by analyzing the code of `Kodein` than by reading a particular piece of documentation, so the code presented herein might be less than optimal. If you think you know of a better solution, please contact me by [submitting a new issue][issues] on GitHub. Thank you.




[kodein]: https://kodein.org/Kodein-DI/?6.2/core
[victor-events]: https://github.com/syrop/Victor-Events
[commit]: https://github.com/syrop/Victor-Events/commit/a3e8fdc1d1a797bdaa1fd549a2c9d812cd23c7f8
[logging]: https://syrop.github.io/jekyll/update/2018/12/30/logging.html
[delegated-properties]: https://kotlinlang.org/docs/reference/delegated-properties.html
[lazy]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-lazy/index.html
[reflection]: https://kotlinlang.org/docs/reference/reflection.html
[issues]: https://github.com/syrop/syrop.github.io/issues
