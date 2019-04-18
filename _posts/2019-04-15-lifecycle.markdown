---
layout: post
title:  "lifecycle"
date:   2019-04-15 10:17:00 +0200
categories: jekyll update
---

This is a very generic article about several different ways of approaching the lifecycle.

This particular article doesn't propose any design patterns, although it does summarize my recent research into the lifecycle in gereral.

During job interviews I am very often asked about `Activity` lifecycle. It is always `Activity` - never `Fragment`.

I do not really know the answer to this question. During a job interview I could always look up the [documentation][lifecycle-doco], which would only prove to the interviewer that I am able to refer to documentation even during a stressful situation or - better still - have memorized some of it.

Still, I am not really sure whether just being able to enumerate the canonical sequence of cycle events makes a good programmer, or a good software architect.

## Different lifecycle functions

There is a function called `onStart()`, which I have never used in production, although I was told recently that it was being called afrer the views have beed displayed on the screen. The [doco][lifecycle-doco] doesn't say how to use it, but it says:

> The `onStart()` method completes very quickly and, as with the Created state, the activity does not stay resident in the Started state.

This is about as much as I am able to say the next time somebody asks me about this function.

There are two prominnent functions `onResume()` and `onPause()` that may be used to, respectively, start updating the views once the app has been resumed, and stop updating them once the app is off the screen again. Failure to use these functions won't hurt, as updating a view that is not currently visible doesn't crash the app, although pausing the updates may free up some resources. If you are concerned about using you resources while the app is not visible, consider using `LiveData`. All but the last value emitted while the app was paused will be ignored, and the views will be only updated when the app is resumed - without you having to implement the body of `onResume()`.

Lastly, there is `onDestroy()`, that probably is meant to be used to get rid of RxJava's `Disposable`s, but you can probably do it as well in `ViewModel`'s `onCleared()`.

Even while I am writing this article I do not know without looking in the [doco][lifecycle-doco] whether I have enumerated all the functions. I am definitely not able to list them all from memory during a job interview.

I usually mention the function `onSaveInstanceState()` when I am asked about `Activity` lifecycle, even thought it is not drawn in the chart that first comes to my mind (fig. 1 in the [documentation][lifecycle-doco]). Still, this particular function may decline in relevance as Android Jetpack has already released an alpha version of [Saved State module][savedstate] for ViewModel.

## RxJava

I think that understanding of the lifecycle is the most important when you use RxJava. You do not want to have subscriptions lying around when the relevant lifecycle ends, although you can clean up after them in the `Activity`'s `onDestroy()` function, the equivalent function in the `Fragment`, so I am not sure whether I should talk about RxJava when I am asked about `Activity` lifecycle specifically, or should I leave discussing RxJava for a more open question, like: 'How to avoid memory leaks?'.

## Coroutines

If a coroutine needs to be cancelled when its `Activity` or `Fragment` is destroyed, you can invoke it in `ViewModel`'s extension property [`viewModelScope`][viewmodelscope]. It will cancel the coroutine for you, so you don't have to, and probably shouldn't, handle `Activity` lifecycle events yourself when you are dealing with coroutines.

## Conclusion

Throughout this article I still haven't been able to discuss `Activity` lifecycle properly.

I still don't know which lifecycle functions are relevant - and which are not - even thought obviously I can look up their list in the [doco][lifecycle-doco] any time of the day.

I do not understand why the [documentation][lifecycle-doco] mentions some of them in a graphical chart, while other seemingly more relevant functions are mentioned only towards the end of the documentation entry.

I think, though, that I am competent in solving problems such as avoiding memory leaks and updating the views only when the app is on the screen.

I think that simply being able to enumerate from your memory a succession of several `Activity` lifecycle events doesn't - by itself - prove that the candidate is able to solve such problems as saving UI state or running concurrent tasks, and this question should be eliminated from job interviews.

[lifecycle-doco]: https://developer.android.com/guide/components/activities/activity-lifecycle
[savedstate]: https://developer.android.com/topic/libraries/architecture/viewmodel-savedstate
[subject]: http://reactivex.io/documentation/subject.html
[viewmodelscope]: https://developer.android.com/reference/kotlin/androidx/lifecycle/package-summary#(androidx.lifecycle.ViewModel).viewModelScope:kotlinx.coroutines.CoroutineScope
