---
layout: post
title:  "Syncing two ListenableWorkers (decorator pattern)"
date:   2019-05-01 14:41 +0200
categories: jekyll update
---

This article will explain how to add a `ListenableWorker` to a project that already has one, and synchronize the work of the two, so that only one of them can work at a time.

## The solution is not ideal

It the [project][victor-events] I use a `PeriodicWorkRequestBuilder` that cannot be [chained][chaining]. Ideally I should [chain][chaining] two works, so that one of them is executed immediately after the other.

I could still do that by using two chained `OneTimeWorkRequest`s, and schedule them again when their work is over, but I chose against doing that. What matters for me is that two different repositories are synchronized, and that their content will be up-to-date eventually. The exact sequence is not important, although one of the repository does contain data that is relevant to the other.

## Scheduling the work

This is the booting sequence that schedules both works:

{% highlight kotlin %}

private inline fun <reified W : ListenableWorker> scheduleSync(tag: String, frequency: Duration) {
    WorkManager.getInstance().enqueueUniquePeriodicWork(
            tag,
            ExistingPeriodicWorkPolicy.REPLACE,
            PeriodicWorkRequestBuilder<W>(frequency)
                    .setConstraints(Constraints.Builder().setRequiresBatteryNotLow(true).build())
                    .build())
}

fun boot() {
    login.setCurrentUser(FirebaseAuth.getInstance().currentUser)
    io {
        listOf(
            launch { comms.fromDb() },
            launch { messages.fromDb() })
                .joinAll()
        scheduleSync<CommSyncWorker>(CommSyncWorker.TAG, CommSyncWorker.FREQUENCY)
        scheduleSync<EventSyncWorker>(EventSyncWorker.TAG, EventSyncWorker.FREQUENCY)
    }
}

{% endhighlight %}

In the above code `io` is a function that launches a block of code using `Dispatchers.IO`:

{% highlight kotlin %}

fun io(block: suspend CoroutineScope.() -> Unit) = GlobalScope.launch(Dispatchers.IO) { block() }

{% endhighlight %}

What the `boot()` function does is it first launches two `Job`s that read contents of two repositories from a local database, and waits for them to join.

It then schedules enqueues two different works with the same sets of constraints.

## The SyncWorker

This is the entire code of the interface responsible for syncing the work of two different `ListenableWorker`s:

{% highlight kotlin %}

private val syncMutex = Mutex()

interface SyncWorker {
    suspend fun <R> syncCoroutineScope(block: suspend CoroutineScope.() -> R): R = coroutineScope {
        syncMutex.withLock { block() }
    }
}

{% endhighlight %}

It first creates a lazy `Mutex` that is only accessible from within the file that defines the interface.

I briefly modified the above initiation of `syncMutex` to make sure whether it was done lazily, by using the following addition:

{% highlight kotlin %}

private val syncMutex = Mutex().apply { Thread.dumpStack() }

{% endhighlight %}

This was the output:

```
W/System.err: java.lang.Exception: Stack trace
W/System.err:     at java.lang.Thread.dumpStack(Thread.java:1348)
        at pl.org.seva.events.main.model.SyncWorkerKt.<clinit>(SyncWorker.kt:27)
        at pl.org.seva.events.main.model.SyncWorkerKt.access$getSyncMutex$p(SyncWorker.kt:1)
        at pl.org.seva.events.main.model.SyncWorker$syncCoroutineScope$2.invokeSuspend(SyncWorker.kt:31)
        at pl.org.seva.events.main.model.SyncWorker$syncCoroutineScope$2.invoke(Unknown Source:10)
        at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:91)
        at kotlinx.coroutines.CoroutineScopeKt.coroutineScope(CoroutineScope.kt:180)
        at pl.org.seva.events.main.model.SyncWorker$DefaultImpls.syncCoroutineScope(SyncWorker.kt:30)
        at pl.org.seva.events.event.EventSyncWorker.syncCoroutineScope(EventSyncWorker.kt:29)
        at pl.org.seva.events.event.EventSyncWorker.doWork(EventSyncWorker.kt:34)
        at androidx.work.CoroutineWorker$startWork$1.invokeSuspend(CoroutineWorker.kt:66)
        at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
        at kotlinx.coroutines.DispatchedTask.run(Dispatched.kt:238)
W/System.err:     at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:594)
        at kotlinx.coroutines.scheduling.CoroutineScheduler.access$runSafely(CoroutineScheduler.kt:60)
        at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:742)
```

The above dump clearly shows that the constant `syncMutex` is initiated inside of a coroutine. I initially thought that I would gave to wrap `Mutex()` in a `lazy` delegate, I even [committed][lazy-commit] my change, but when I performed the tests documented here I returned to the previous version of my code.

## The CoroutineWorker

This is the entire code of the CoroutineWorker (subclass of ListenableWorker):

```kotlin
class EventSyncWorker(context: Context, params: WorkerParameters) :
        CoroutineWorker(context, params), SyncWorker {

    override val coroutineContext = Dispatchers.IO

    override suspend fun doWork() = syncCoroutineScope {
        events.refresh()
        Result.success()
    }

    companion object {
        val TAG: String = this::class.java.name
        val FREQUENCY: Duration = Duration.ofHours(2)
    }
}
```

The above code contains a `companion object` defining the tag at which the work is enqued, and defines the `CoroutineDispatcher` that runs it.

Because the class implements `SyncWorker` interface discussed already in the previous sections, it gain access to `syncCoroutineScope` which runs the work under the lock of a `Mutex`, by calling:

```kotlin
syncMutex.withLock { block() }
```

Alternatively, instead of implementing `SyncWorker` in order to gain access to `syncCoroutineScope()`, I could just make `synCoroutineScope()` a global function, and it would work just as well. I didn't choose to do that, because I thought that creating a global function that is relevant only when called from a `CoroutineWorker` would litter the global namespace too much.

If you are curious how the refreshing itself is performed, I paste below the actual function. Explaining its operation is outside of the scope of this article:

```kotlin
suspend fun refresh(): List<Event> = coroutineScope {
    mutableListOf<Event>().apply {
        comms.map {
            async { fsReader readEventsFrom it.lcName }
        }.map {
            it.await()
        }.onEach {
            addAll(it)
        }
    }.apply {
        eventsCache.clear()
        eventsCache.addAll(this)
        notifyDataSetChanged()
        eventsDao.clear()
        launchEach { eventsDao.add(it) }
    }
}
```

The above code only uses Kotlin's functional fundamentals to download the list of all events for all communities and add them to a `MutableList`. When the list is already downloaded and stored in the memory cache of this repository, the code invokes `notifyDataSetChanded()` which is my [custom function][refreshing-article] notifying all potential observers about the change. Finall, it clears the persistent database and adds to it all of the newly found events.

## Asynchronous persistence

The `launchEach()` function deserves a special mention. It asynchronously adds every event do the persistent database:

```kotlin
suspend fun <T> Iterable<T>.launchEach(block: suspend (T) -> Unit) = coroutineScope {
    map { launch { block(it) } }
}
```

If I wanted to wait for adding events to the database to finish, I would only have to write:

```kotlin
launchEach { eventsDao.add(it) }.joinAll()
```

Alternatively, I could just tell Room to create a `suspend` function that adds a whole `Collection` of events in one step. I chose the above solution to reuse the code I already have. The [project][victor-events] is design to only handle the handful of events the user is interested in attending over the following couple of weeks. It will probably only be twenty or so, because even if the user is watching three communities, and each is having two events per month, it is still only twelve events in total for the following two months.

I haven't compared the performance of adding either the entire `Collection` of events in one step, or like I showed above adding every event simultaneously. Because the work is only scheduled to run once every two hours, on`Dispatchers.IO`, this shouldn't make a big difference.

## Conclusion

The present article has explained how to use an interface to add an extra piece of functionality to a `CoroutineWorker`.

Thanks to the `syncCoroutineScope()` function shown above, the work is performed under the lock of a `Mutex` by calling:

```kotlin
syncMutex.withLock { block() }
```

The reader has also had a chance to investigate with me how an interface may be used to lazily and outside of the main thread initiate a value the interface is using, provided that it has been accessed for the first time from a `suspend` function.

The code may be further improved by just combining many kinds of work into one instance of `CoroutineWorker`, or using [chaining] to chain two or more instances of `OneTimeWorkRequest`.

In the future version of my [app][victor-events] I may use [paging] to only download chunks of data at a time, or observe Cloud Firestore in real time instead of using a `CoroutineWorker`.

The reader is welcome to see the particular [commit] that introduced the changes discussed in the present article. I might still change the architecture of the [project][victor-events] many times, but by waching the particular [commit] the reader might investigate a curious case of [decorating][decorator] a class, that could escape their attention if they were just to clone the final version of the [code][victor-events].

[victor-events]: https://github.com/syrop/Victor-Events
[lazy-commit]: https://github.com/syrop/Victor-Events/commit/7c095f1ed024b2c65103738e859c27b6911fc67b
[chaining]: https://developer.android.com/topic/libraries/architecture/workmanager/how-to/chain-work
[fix-commit]: https://github.com/syrop/Victor-Events/commit/c3a1da294f6e757e0b03f163f69dd9ff54d36c32
[refreshing-article]: https://syrop.github.io/jekyll/update/2019/04/27/refreshing-your-data.html
[paging]: https://developer.android.com/topic/libraries/architecture/paging/
[commit]: https://github.com/syrop/Victor-Events/commit/0343b1149a38a928b68bc1396de6d791316d2979
[decorator]: https://en.wikipedia.org/wiki/Decorator_pattern
