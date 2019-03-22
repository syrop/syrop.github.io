---
layout: post
title:  "Agile or othewise"
date:   2019-03-18 11:17:00 +0100
categories: jekyll update
---

This is the first philosophical entry in this blog.

It may or may not help the reader to decide whether I have soft skills.

## Am I agile?

I do not have a ready plan for my [Victor Events][victor-events] project. Every so many days I register a [ticket][tickets], as they become apparent to me. Most of them are enchancements. I then either fulfill one of them or not. I have closed some of the tickets I submitted, without fulfilling them.

Recently I wrote an article on [combining RxJava and Android Jetpack][observable-to-livedata], then I decided it contained errors that lead to leaks, and I [wrote another one][rxjava-and-livedata]. Both of them refer to my [repository][victor-events], as it was at the time when I wrote the article, although some of the content and design patterns I document in the earlier one is no longer up-to-date.

This is one of the reasons I write a blog, as opposed to a book, or just documenting the code in comments that will be later forgotten and/or deleted. You are meant to see how the designs patterns grow, and how it takes several days to improve a pattern that was already reasonably advanced when I documented it the last time.

(It took me 11 days to progress from the [first][observable-to-livedata] to the [second][rxjava-and-livedata] article, and they are both on the same topic).

As a result of that, you also have a proof that I am sharing with you original content. I've really gone throught the steps of compiling these materials from [Android Jetpack][jetpack] documentation and YouTube presentations.

## On recruitment forms

Most often than not, when I see a job advertisements, the requirements are expert Java and (nice to have) basic Kotlin.

Don't you think that I could take some of the data presented in this blog and translate it to Java if I had the heart to do it? Is it difficult to implement MVP design pattern in plain Java using RxJava and some Observables?

I do not really care, though, as the code would be then way less concise and readable. I am also not really all that skilled in Java 8, as at the time when Java 8 was taking hold in Android I was already running one or two repositories in Kotlin. (Not this blog, though, as I launched this blog much later).

When I see requirements in job advertisemens, seldom do I see Kotlin being even on equal grounds with Java. You are first and foremost required to be an expert in Java, and you may also be given a chance to talk about some most elemental ingredients of Kotlin, if there is time for it.

The most cruel thing you may face during a recruitment process is being required to fill in a skills form.

I recenly had a scheduled job interview in a company that required me to fill in one. It was supposed to be an Android position, yet the form had questions whether I knew Java, Swift and stuff, but Kotlin was not even there. The recruiter told me, though, that I could add an extra position for Kotlin at the end of the form.

The form contained a question whether I knew Dagger (a dependency injection). I could just fill in that I heard of Dagger, and add [Kodein][kodein] at the end of the list, since I already run a blog on how to use [Kodein][kodein] for dependency injection and dependency retrieval, but I just refused to fill in the form, and had the recruiter cancel the interview.

How could I have possibly grown by listing Kodein in Kotlin, lambdas in Kotlin, extension functions in Kotlin, since the company wasn't probably interested in Kotlin at all, and didn't have any interview questions prepared for verifying these particular fields? The recruiter had already told me that the interviewer was not going to study my blog.

## Agile in a company

The handful of times when I was actually involved in an agile process in a company, it wasn't really like I described in the first section of the present article.

The companies I have worked for usually din't document their code in a separate document entitled something like 'Why I chose this particular design pattern over another?', and weren't very open to changes in the code (despite calling the whole process *agile*).

The *agile* part of the agile process meant usually that the developer had to stretch their skills to adapt a particular design pattern or a library to implement a piece of functionality that this programming tool was otherwise unfit for.

Architecture choices were not documented and seldom changed, what created a huge [technical debt][technical-debt].

Eventually the most skilled developers would either choose to leave or be fired by the company. The company didn't care anyway, as the company policy was designed to keep in only the people who fit in the [organizational culture][organizational-culture].

## The purpose of this blog

In this blog I am trying to document good programming practices I've been able to compile from documentation of Kotlin, YouTube presentations, and inspired partly by my modest interest in Scala. I am trying to fill in the lack of good Kotlin-specific documentation of architecture that might come in handy in Android projects, whereas a more broad documentation of Java-specific architecture is already available in the works of [Uncle Bob][uncle-bob], and in the book 'Effective Java' by Joshua Blosh.

I am trying to promote the view that documentation does matter, and share my understanding of why a particular choice leads to code that may be later improved on without introducing much regression.

The techniques I mostly promote in this blog are MVP model, where the presenter is [retrieved with Kodein][kodein], which is triggered by RxJava, but observers are registered to it thru LiveData, and [do not repeat yourself][do-not-repeat-yourself] principle, which is achieved thru practical extension functions.

[victor-events]: https://github.com/syrop/Victor-Events
[tickets]: https://github.com/syrop/Victor-Events/issues
[observable-to-livedata]: https://syrop.github.io/jekyll/update/2019/03/07/rx-observable-to-livedata.html
[rxjava-and-livedata]: https://syrop.github.io/jekyll/update/2019/03/18/combining-rxjava-and-livedata.html
[jetpack]: https://developer.android.com/jetpack/
[kodein]: https://kodein.org/Kodein-DI/
[technical-debt]: https://en.wikipedia.org/wiki/Technical_debt
[organizational-culture]: https://en.wikipedia.org/wiki/Organizational_culture
[uncle-bob]: https://en.wikipedia.org/wiki/Robert_C._Martin
[do-not-repeat-yourself]: https://en.wikipedia.org/wiki/Don't_repeat_yourself
