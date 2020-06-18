---
layout: post
title:  "Refactoring the code to functional style"
date:   2019-09-14 20:14:00 +0200
categories: jekyll update
---

This is an article about functional programming.

## The problem

Given a handful of functions that refresh a repository held in cache, refactor them to only mutate the state of the repository at the very end.

The only function that mutates the repository is to be called `commit()` and accept no parameters.

The entire thing should take advantages of coroutines, switching the `CoroutineContext` as appropriate.

## The commits

The reared may find the solution to the problem in a particular [commit] in the project [Victor Events][events].

Some of the later changes, discussed in the present article, but not included in this particular [commic], involve adding a return value to `commit()`.

After the changes, the `commit()` function launches asynchronously an operation that commits the values to a persistent storage, and immediately returns `this`.

## Before the refactoring

These are fragments of the code as it was before the solution was applied:

```kotlin
suspend fun refreshAdminStatuses() =
    refresh { it.copy(isAdmin = fsReader.isAdmin(it.lcName)) }

suspend fun refresh() =
    refresh { fsReader.findCommunity(it.name).copy(color = it.color) }

private suspend fun refresh(transform: suspend (Comm) -> Comm): List<Comm> =
        withContext(Dispatchers.Default) {
            val transformed = commsCache
                    .toList()
                    .async { transform(it) }
                    .awaitAll()
            withContext(NonCancellable) {
                commsCache.clear()
                commsCache.addAll(transformed.filter { !it.isDummy })
                commsDao.clear()
                commsDao add commsCache
                notifyDataSetChanged()
            }
            transformed
        }
```

In the above code, the most of work is performed by the `refresh` function. It takes as a parameter the lambda that will be called od every item, runs it asynchronously and saves the changes before returning the transformed list.

Above it, there are two functions that take advantafe of `refresh()` by calling it in order to replace every item either by one with just an updated admin status, or with one whose all values save for color have been replaced with that from Firebase Firestore.

## After the refactoring

This is the code after the changes. Most of the changes the reader may find in the [commit] mentioned in one of the preceeding sections.

```kotlin
suspend fun refreshAdminStatuses() =
    mapCache { it.copy(isAdmin = fsReader.isAdmin(it.lcName)) }.commit()

suspend fun refresh() =
    mapCache { fsReader.findCommunity(it.name).copy(color = it.color) }.commit()

private suspend fun mapCache(transform: suspend (Comm) -> Comm) =
        withContext(Dispatchers.Default) {
            commsCache
                    .toList()
                    .async { transform(it) }
                    .awaitAll()
        }

private suspend fun List<Comm>.commit() = coroutineScope {
    launch(NonCancellable) {
        commsCache.clear()
        commsCache.addAll(filter { !it.isDummy })
        commsDao.clear()
        commsDao add commsCache
        notifyDataSetChanged()
    }

    this@commit
}
```

In the above code, the function `mapCache()` maps the entire cache asynchronously, which means that the transform may be applied to many items at the same time. It returns the list of the transformed items, but does not save the change.

In `mapCache()` I decided to use `Dispatchers.Default`, because the function performs mostly computational work, like creation of new lists, or data manipulation. The `transform` itself should chose its own `CoroutineScope`. In the [project][events] I use `Dispatchers.IO` for the `transform`, because it downloads data from the internet, but discussing the intricate work of each `transform` is outside of the scope of the present article.

The functions `refresh()` and `refreshAdminState()` call `mapCache()`, passing an appropriate `transform` to it. It is these two functions that are responsible to call `commit()` if they want to save changes.

I chose the name `commit()` as opposed to `apply()`, because I want the function to work asynchronously.

The function immediately returns the list of values. The values are not being changed in this step.

The reader is advised to note that parts of the operations of `commit()` are performed in `NonCancellable`, which is the `CoroutineContext` I chose because I want these operations to be run as an atomic block of code. Canceling the `CoroutineScope` it works in will not affect the operations once the control is inside that block.

## Conclusion

The reader has had an opportunity to see how I refactored my own code to make it more readable even for myself.

Before that change I was already starting to wonder what the code was doing. I was considering adding javadocs, but eventually I saw that there were unnecessarily two functions that were both called `refresh()`, one of which were doing two different things - both mapping the values and saving the changes - and that they were being saved not at the end, but somewhere in the middle of the code.

Eventually, I dediced to write the code in the order that I want it to work:

* Map the values by giving them a lambda transforming them.
* Save them in an atomic ansynchronous operation when they are ready.
* Return the transformed values to allow further processing: for example logging.

## Donations

This blog is a part of my programming passion. I document in it the changes I make in the projects I run in my free time.

I am trying to put in it stuff I haven't found in other blogs or pieces of documentation.

If the reader thinks they have learned anything from this, and they are in the mood for rewarding me for it, please send me some bitcoin using my [donations page][donate].


[commit]: https://github.com/syrop/Victor-Events/commit/fdd65690d9ddfb0ee30971a336a266643de26791
[events]: https://github.com/syrop/Victor-Events
[donate]: https://syrop.github.io/donate/

