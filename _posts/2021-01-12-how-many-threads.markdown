---
layout: post
title:  "How many threads run coroutines?"
date:   2021-01-12 20:23:00 +0200
categories: jekyll update
---

I've started to wonder about an efficient way to count how many threads are being used to run my coroutines.

## Dispatchers.IO

I've started with `Dispatchers.IO`, because it should use the most threads, right?

I've designed the following code:

```kotlin
GlobalScope.launch {
    val threads = Array(100_000) {
        async(Dispatchers.IO) { Thread.currentThread() }
    }.map { it.await() }.toSet()
    println("wiktor size ${threads.size}")
}
```

It first creates a hundred thousands of coroutines, each running within `Dispatchers.IO` coroutine context.

Then it waits for them all to finish, thus creating an array of instances of `Thread`.

Lastly, it stores all threads in a set. Because set contains unique elements, each thread is stored only once.

This is the output of running the above code several times:

```
wiktor size 17
wiktor size 32
wiktor size 14
wiktor size 13
wiktor size 22
wiktor size 14
```

Each time the number was higher than a dozen, but lower than three dozens.

## Dispatchers.Default

`Dispatchers.Default` should use about as many threads as the number of my physical code. Let's try.

The code:

```kotlin
GlobalScope.launch {
    val threads = Array(100_000) {
    async(Dispatchers.Default) { Thread.currentThread() }
    }.map { it.await() }.toSet()
    println("wiktor size ${threads.size}")
```

Because this is the default dispatcher for `GlobalScope`, one can omit the first parameter of the function `async()`. The shortened code thus looks this way:

```kotlin
GlobalScope.launch {
    val threads = Array(100_000) {
    async { Thread.currentThread() }
    }.map { it.await() }.toSet()
    println("wiktor size ${threads.size}")
}

```

This is the result of running it a couple of times:

```
wiktor size 3
wiktor size 3
wiktor size 3
```

Each time it uses three threads.

## Dispatchers.Main

`Dispatchers.Main` only uses main thread. To verify this, I use the following code:

```kotlin
GlobalScope.launch {
    val threads = Array(100_000) {
        async(Dispatchers.Main) { Thread.currentThread() }
    }.map { it.await() }.toSet()
    println("wiktor size ${threads.size}")
}
```

The output:

```
wiktor size 1
```

As expected, it runs all coroutines on only one thread.

## Running it all on the main thread.

I modified the above code to run the entire test on main thread:

```kotlin
class MainFragment : Fragment(R.layout.fr_main) {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        lifecycleScope.launch {
            val threads = Array(10_000) {
                async { Thread.currentThread() }
            }.map { it.await() }.toSet()
            println("wiktor size ${threads.size}")
        }
    }
}
```

`lifecyclescope` by defaul uses `Dispatchers.Main.immediate`, so the entire test runs on main thread.

I had to reduce the number of coroutines to ten thousands, otherwise the code would take too long to run.

This time, mapping and waiting also is executed on the main thread. When I tried to run it with a hundred thousands coroutises, I had to manually break it, because I didn't want to wait.

This illustrates that even if a particular operation is expected to run on `Dispatchers.Main.immediate`, the greater context (`lifecyclescope` vs `GlobalScope`) makes a difference.

To remedy this, I can switch coroutine context right in the `launch` command:

```
lifecycleScope.launch(Dispatchers.Default) {
    val threads = Array(100_000) {
        async(Dispatchers.Main) { Thread.currentThread() }
    }.map { it.await() }.toSet()
    println("wiktor size ${threads.size}")
}
```

The advantages of the above code are such, that the coroutines are correctly canceled due to `lifecyclescope`, the most demanding part runs on `Dispatchers.Default`, and the internal coroutines work on `Dispatchers.Main`. Again, it completes within a reasonable time.

The output is in both cases:

```
wiktor size 1
```

## Dispatchers.Unconfined

`Dispatchers.Unconfined` is probably the least used dispatcher, and the one I know least.

Doco says it is using the same thread that started it:

> A coroutine dispatcher that is not confined to any specific thread. **It executes initial continuation of the coroutine in the current call-frame** and lets the coroutine resume in whatever thread that is used by the corresponding suspending function, without mandating any specific threading policy.

I added some more debugs to verify that:

```kotlin
GlobalScope.launch {
    val threads = Array(100_000) {
        async(Dispatchers.Unconfined) { Thread.currentThread() }
    }.map { it.await() }.toSet()
    println("wiktor size ${threads.size}")
    val thread = threads.take(1)[0]
    println("wiktor GlobalScope thread ${thread == Thread.currentThread()}")
    println("wiktor main thread ${thread == Looper.getMainLooper().thread}")
}
}
```

This time, I take an instance of the only thread that runs the coroutines. I compare it to the current thread (the thread that runs the GlobalScope), and verify that it is different from main thread.

The output:

```
wiktor size 1
wiktor GlobalScope thread true
wiktor main thread false
```

The above output shows that all of the coroutines run on one thread, which happens to be the same thread `GlobalScope` runs on, and is different from the main thread.

Can `Dispatchers.Unconfined` run on main thread?

Of course. I wrote the following code:

```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    GlobalScope.launch(Dispatchers.Unconfined) {
        println("wiktor main thread ${Looper.getMainLooper().thread == Thread.currentThread()}")
    }
}
```

This time I made `GlobalScope` use `Dispatchers.Unconfined`, which according to the doco executes the initial continuation in the same thread it was launched from. Here *initial continuation* means everything, since I do not use any suspending functions.

The output:

```
wiktor main thread true
```

## Conclusion

This article, contrary to many other articles in this blog, uses sample code and code snippets, as opposed to real project.

It demonstrates how to write a small handful of lines to help one to choose the right dispatcher.

It also demonstrates, by the example of `Dispatchers.Unconfined`, how to use simple code to verify one's understanding of documentation.
