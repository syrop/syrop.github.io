---
layout: post
title:  "Preventing canceling a coroutine"
date:   2019-06-08 11:45 +0200
categories: jekyll update
---

This article explains how to prevent canceling a coroutine using [`NonCancellable`][noncancellable].

## The problem

Write a coroutine that cannot be canceled. Parts of this coroutine will involve plenty of computation, while other parts will only involve wainitg for a result of an I/O operation.

Once that coroutine has completed, invoke an action on a `Fragment`, but only if the `Fragment` is still alive.

## The project

The project in which I used this technique is [Victor Events][victor-events].

The change is contained within a particular [commit], but the reader is encouraged to follow the present article and the snippets presented herein, instead of trying to decipher the [commit] on their own, as the commit may contain garbage like renaming of a value, or adding a line break. The code might have also changed a few times since I made that commit, whereas in the article I am making effort to present the code's most relevant and-up to date version.

## The Fragment

Since the UI action should be invoked only if the `Fragment` is still alive, I am using the [`lifecycleScope`][lifecyclescope] extension property:

```kotlin
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    if (requestCode == LOGIN_REQUEST && resultCode == Activity.RESULT_OK) {
        prompt.visibility = View.GONE
        ok.visibility = View.GONE
        privacy_policy.visibility = View.GONE
        cancel.visibility = View.GONE
        progress.visibility = View.VISIBLE
        lifecycleScope.launch {
            comms.refreshAdminStatuses()
            yield()
            back()
        }
    }
}
```

The above code is invoked after a log-in attempt has been made. If the user has successfully logged in, most of the GUI elements are hidden, progress bar is brought to the front and an operation is launched, which is meant to check which communities the logged-in user is an admin of.

After refreshing of the admin status of each community, the `Fragment` is closed by calling `findNavController().popBackStack()`. However, the `Fragment` may be already destroyed, so the `back()` function is only called if `lifecycleScope` hasn't been canceled.

The problem is that the refreshing coroutine shall not be canceled while it is already running, as this could result with the repository content not being conistend - some items for instance might be updated only in memory cache, but the might not persisted.

One of the ways of preventing canceling of a coroutine is to use a `CoroutineScope` that is always active - [`GlobalScope`][globalscope]. At first I was experimenting with the following code, and I still believe it's okay:

```kotlin
val refreshJob = GlobalScope.launch { comms.refreshAdminStatuses() }
lifecycleScope.launch {
    refresJob.join()
    back()
}
```

Still, Kotlin allows fluently switching a `CoroutineContext` without launching an isolated coroutine, and this is the solution on which I am going to focus subsequently in the article.

## NonCancellable

[`NonCancellable`][noncancellable] extends [`Job`][job], and this is its `cancel()` implementation:

```kotlin
override fun cancel(cause: CancellationException?) {}
```

The above line of code literally does nothing, so off course children of this [`Job`][job] will not be canceled when this empty implementation of `cancel()` is called.

I wrote some dummy code to demonstrate the behavior of `NonCancellable` in my playground project. One can still see it in a [commit][commit-playground]:

```kotlin
GlobalScope.launch {
        val scope = CoroutineScope(Job() + Dispatchers.Main)
        scope.launch {
            withContext(NonCancellable) {
                println("wiktor inside the block")
                delay(2000)
		yield()
                println("wiktor finished successfully")
            }
            yield()
            println("wiktor not cancelled")
        }
        delay(1000)
        scope.cancel()
        println("wiktor canceled")
    }
```

The code launches a block setting its parent `Job` to `NonCancellable`. It then logs a line to confirm the block is running.

After one second it cancels the `CoroutineScope` the block is running in, and confirms it by logging another line.

Then the line `wiktor finished successfully` is logged to prove that the code called with a `NonCanlellable` was never canceled.

The line `wiktor not canceled`, however, is not logged, as it is called outside of `NonCanlellable`. The `yield()` function throws a `CancellationException` then and prevents calling `println()`.

Neither `delay()` nor `yield()` throw a `CancellationException` when they are called from inside of a `NonCancellable`. Because of that I have to call `yield()` again, outside of the block, to prevent further execution of the code inside of the cancelled `scope`.

Canceling the `CoroutineScope` is discussed in the talk '[Android Suspenders (Android Dev Summit '18'][suspenders])' (beginning at 18:09). Although `NonCancellable` is not discussed there, watching that particular talk allowed me to come up with these solutions.

## Refreshing and transforming data

Explaining the code that refreshes the admin status of each community is outside of the scope of the present article, althought it has already somewhat been explained in another [article][admin-statuses] in this blog.

This is the abbreviation of the code:

```kotlin
suspend fun refreshAdminStatuses() =
        refresh { it.copy(isAdmin = fsReader.isAdmin(it.lcName)) }

private suspend fun refresh(transform: suspend (Comm) -> Comm): List<Comm> =
        withContext(NonCancellable + Dispatchers.Default) {
            val commsCopy = commsCache.toList()
            val transformed = mutableListOf<Comm>()

            commsCopy.launchEach { transformed.add(transform(it)) }
                    .joinAll()
            commsCache.clear()
            commsCache.addAll(transformed.filter { !it.isDummy })
            notifyDataSetChanged()
            commDao.clear()
            commsCache.launchEach { commDao add it }
            transformed
        }
```

First, the reader will noticed that I use the syntax:

```kotlin
withContext(NonCancellable + Dispatchers.Main)
```

As I showed in one of the above section, this code is called from within `lifecycleScope`, which by default uses `Dispatchers.Main`. Here I am replacing the `Job` of the `CoroutineContext` with `NonCancellable`, and furthermore I am setting the `CoroutineDispatcher` to `Dispatchers.Default`, which is more suitable for executing work that requires much computation.

The `transform` operation, however, is using `Dispatchers.IO` to perform reading from Firebase Datastore:

```kotlin
suspend infix fun isAdmin(name: String): Boolean = if (isLoggedIn)
        name.admins.document(login.email).doesExist() else false

private suspend fun DocumentReference.doesExist() = read().exists()

protected suspend fun DocumentReference.read() = withContext(Dispatchers.IO) {
    ...
}
```

The above code demonstrates that each `read()` operation is called with `Dispatchers.IO`, while most of the computation is performed with `Dispatchers.Default`, as discussed previously, and eventually it will trigger an action on the `Fragment`, using `Dispatchers.Main`, but only if the `Fragment` is still alive.

## Conclusion

The present article promotes using coroutines for performing tasks that require frequent switching of contexts, or that should never be canceled if they are already running.

I am not aware how to do the above with RxJava. I am aware that in this library, used frequently as an alternative to coroutines, context switching can be achieved with [subscribeOn()][subscribeon] and [observeOn()][observeon], but I am not sure how to switch the context dozens of times during one operation, or how to prevent the operation from being canceled.

If the reader knows how to solve these problems purely with RxJava, please submit a [ticket], and if I like the solution, I will edit the present article to include it.


[noncancellable]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-non-cancellable.html
[victor-events]: https://github.com/syrop/Victor-Events
[commit]: https://github.com/syrop/Victor-Events/commit/c2470f19b757a3f017ff81206a517edf657c33be
[lifecyclescope]: https://developer.android.com/topic/libraries/architecture/coroutines#lifecyclescope
[globalscope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[commit-playground]: https://github.com/syrop/MyApplication/commit/b6d7c109382794dc4715b6ce4ad6b9a18c48415e
[suspenders]: https://www.youtube.com/watch?v=EOjq4OIWKqM&feature=youtu.be&t=1089
[admin-statuses]: https://syrop.github.io/jekyll/update/2019/04/27/refreshing-your-data.html
[subscribeon]: http://reactivex.io/documentation/operators/subscribeon.html
[observeon]: http://reactivex.io/documentation/operators/observeon.html
[ticket]: https://github.com/syrop/syrop.github.io/issues
