---
layout: post
title:  "Java or Kotlin?"
date:   2019-03-27 22:14:00 +0100
categories: jekyll update
---

This is more of a 'current affairs' as opposed to 'architecture' post, as I am describing the present shape of Kotlin as I know it, as opposed to a particular design pattern.

Still, much of the content of this article is focused on architecture of Android programs in Kotlin, mostly on coroutines.

Firstly I will try to discuss why use Kotlin at all, as opposed to Java.

## NullPointerExcetion

I have never really understood what the big deal was about type safety in Kotlin.

Just don't assign wrong types to wrong variables, and don't use null values where you are not supposed to. Problem solved.

I hardly had any `NullPointerException`s in Java at the time when I decided to refactor all programs in my [profile][github] to Kotlin. In fact I started getting plenty of `KotlinNullPointerException`s, which is the same thing, when I moved on to Kotlin. Kotlin has a more complex syntax around `null`s, and achieves the same results.

Sure, when I do receive a `KotlinNullPointerException`, they are trivial to fix, as they usually occur right after the `Activity` is started. Fixing Java's `NullPointerExpeption`s is way more difficult, yet this is also 100% eliminated when you do not assign `null` values when you do not mean to.

`NullPointerException` in Java is a nonissue for me, so Kotlin's approach with the `!!` operator that throws a `KotlinNullPointerException` at you to tell you that you are stupid makes no difference.

## Extension fuctions

Extension functions are a huge thing for me, as in this blog I use many, for example in [this article][rx-to-livedata] I write about an extension function that subcribes to RxJava's `Observable` and returns an instance of `LiveData`. Fancy and handy, but you can do the same with Java's static methods, which in fact Kotlin's extension functions are based on.

## Inversion of control

In Kotlin I do not really use dependency injection, as I never explicitly inject all dependencies into an instance of a class by calling a single function why the instance is being initiated.

Dependency injection is only one of the ways inversion of control, inversion of controll meaning that the code of the class does not control the way in which the dependency is resolved, but dependency resolution is handled by external code.

[Dagger][dagger] is a Java-specific library that implements dependency *injenction*, which means that you are meant to *inject* the values at one point of another.

I use Kotlin-specific [Kodein][kodein], which is also a dependency injection library, but apart from *injecting* all dependencies by a method call, you can *retrieve* them, that is, explicitly request them, only one dependency at a time.

## Coroutines and channels

Coroutines went stable in Kotlin 1.3, and I do use them in the project [Viktor Events][events] when I access a Room database. Coroutines in their present form are good for running an action that is meant to run once, complete and maybe result in updating the UI.

Coroutines are not so good, though, at least not in Kotlin 1.3, to respond to events contituously, like to changing states of your phone (whether the phone is stationary or moving).

Kotlin 1.4 should introduce [channels][channels], which serve exactly this purpose. You can subscribe to them by calling `consumeEach()`, and your action is going to continue to be inoked on every subsequent event.

Channels still have an experimental status. They probably cannot be used without appropriately annotating the class and using a particular Kotlin compiler options. I am tempted to write an article on how to trigger something via a coroutine channel, and then respond to it via a `LiveData`, but I think I will refuse to do it until channels become stable.

## Channels vs Subjects

I think that channels do not store and repeat a value, so the observer only receives the values that are passed to a channel after the observer has been connected.

I could probably fix that shortage by creating a design pattern that stores the values passed to it in a `List` or something, and then creates a separate channel for every observer, and then replays the values to it.

Still, RxJava already contains the `ReplaySubject` and `BehaviorSubject` that do just that. I do not know whether rewriting them in Kotlin and using excusively coroutines is worth it. (Probably I will do it anyway when I install Kotlin 1.4).

You can see a demonstration of the `BehaviorSubject` in my project [Timer][timer], in which I use this `Observable` to create two on-screen counters.

## Conclusion

I am so much used to the `!!` operator and extension functions in Kotlin that I take them for granted.

Still, apart from Kodein, they do not really make a big difference in Android development.

I am not sure whether during job interviews I should insist on using only Kotlin at work, or should I wait for Kotlin 1.4, when stable coroutine channels (without having tou se a special compiler flag) are probably going to be available.

I used to walk out of job interviews when I found out that the company was going to run their projects only in Java. Now, as I summed up in the present article the major differences between Kotlin and Java, I may be more able to endure the Java-specific interview, and maybe even convey some important thoughts.

[github]: https://github.com/syrop
[rx-to-livedata]: https://syrop.github.io/jekyll/update/2019/03/07/rx-observable-to-livedata.html
[events]: https://github.com/syrop/Victor-Events
[channels]: https://kotlinlang.org/docs/reference/coroutines/channels.html
[timer]: https://github.com/syrop/Timer
[dagger]: https://google.github.io/dagger/
[kodein]: http://kodein.org/Kodein-DI/
