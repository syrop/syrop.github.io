---
layout: post
title:  "Retrofit asynchronously"
date:   2019-06-18 20:48 +0200
categories: jekyll update
---

This article explains how to use [Retrofit] asynchronously.

## The problem

Download data about platforms that are said to be used by SpaceX for space travel. Use [SpaceX API][spacex].

Once you have done that, download a thumbnail from Wikipedia and display it.

When the user taps on one of such platforms, they are driven to a `Fragment` displaying locationof the chosen platform.

## The project

The project is [Launch Pads][launchpads]. It is also available on [Google Play].

I use Retrofit 2.6.0, because it allows running asynchronous requests by adding the `suspend` keyword to each function. This the dependency:

```groovy
implementation 'com.squareup.retrofit2:retrofit:2.6.0'
```

## Credits

I originally learned how to use Retrofit in this particular manner from [another blog][blog].

The rocket icon, which I use for the launcher, Goople Play and a placeholder inside app comes from [Clipart Library][cliparts].

I found information on how to use [Wikipedia API][wikipedia] in a particular [thread][stackoverflow] on Stack Overflow.

## Services

This is the code for downloading data from [SpaceX API][spacex] and [Wikipedia API][wikipedia]:

```kotlin
interface SpaceXService {
    @GET("launchpads")
    suspend fun all(): Response<List<LaunchPadJson>>
}

class SpaceXServiceFactory {

    fun getLaunchPadService(): SpaceXService = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .addConverterFactory(ScalarsConverterFactory.create())
        .addConverterFactory(MoshiConverterFactory.create())
        .build()
        .create(SpaceXService::class.java)

    companion object {
        const val BASE_URL = "https://api.spacexdata.com/v3/"
    }
}

interface WikipediaService {
    @GET("summary/{article}")
    suspend fun getSummary(@Path("article") article: String): Response<WikipediaArticle>
}

class WikipediaServiceFactory {

    fun getWikipediaService(): WikipediaService = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .addConverterFactory(ScalarsConverterFactory.create())
        .addConverterFactory(MoshiConverterFactory.create())
        .build()
        .create(WikipediaService::class.java)

    companion object {
        const val BASE_URL = "https://en.wikipedia.org/api/rest_v1/page/"
    }
}
```

These are the Kodein bindings:

```kotlin
bind<SpaceXServiceFactory>() with singleton { SpaceXServiceFactory() }
bind<SpaceXService>() with singleton { spaceXServiceFactory.getLaunchPadService() }
bind<WikipediaServiceFactory>() with singleton { WikipediaServiceFactory() }
bind<WikipediaService>() with singleton { wikipediaServiceFactory.getWikipediaService() }
```

## Downloading and parsing

This is the type of the data downloaded from [SpaceX API][spacex]:

```kotlin
@ExperimentalCoroutinesApi
data class LaunchPadJson(
    val status: String,
    val location: Location,
    val wikipedia: String,
    val site_id: String) {

    fun toLaunchPad(scope: CoroutineScope) = LaunchPad(
        status,
        location,
        scope.async { getThumbnail() },
        site_id
    )

    private suspend fun getThumbnail(): String {
        val response = wikipediaService.getSummary(wikipedia.replace(PREFIX, ""))
        return if (response.isSuccessful) response.body()!!.thumbnail.source else ""
    }

    companion object {
        const val PREFIX = "https://en.wikipedia.org/wiki/"
    }
}
```

It contains a function converting it to another format. Please note that the resulting class contains one `Deferred` field:

```kotlin
@ExperimentalCoroutinesApi
data class LaunchPad(val status: String, val location: Location, val thumbnail: Deferred<String>, val site_id: String)
```

This is the way it is used. Please note that it can be run in two modes. At first, when the `Deferred` hasn't completed, it simply calls `await()` on it. The second times it calls `getCompleted()`, that can be called withouth suspension, but throws `IllegalStateException` if the data isn't yet available:

```kotlin
override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    val lp = list[position]
    holder.name.text = lp.location.name
    holder.status.text = lp.status
    try {
        Picasso.get()
            .load(lp.thumbnail.getCompleted())
            .into(holder.thumbnail)
    }
    catch (e: IllegalStateException) {
        scope.launch {
            Picasso.get()
                .load(lp.thumbnail.await())
                .into(holder.thumbnail)
        }
    }
}

```

Initially I was trying not to use `getCompleted()` at all, which forced me to each time start a coroutine and call `await()` in it. However, this resulted in the placeholder being displayed ever-so-briefry each time when I returned to this `Fragment` from another one; I concluded therefore that I had to call `launch()` inside `onBindViewHolder()` very sparingly.

## The LiveData

This is the way I create the `LiveData` that is going to be observed:

```kotlin
val ld = liveData(
    context = viewModelScope.coroutineContext,
    timeoutInMs = Long.MAX_VALUE) {
    coroutineScope {
        val response = spaceXService.all()
        if (response.isSuccessful) {
            emit(Status.Success(response.body()!!.map { it.toLaunchPad(this) }))
        }
        else {
            emit(Status.Error)
        }
    }
}

sealed class Status {
    object Error : Status()
    data class Success(val list: List<LaunchPad>) : Status()
}
```

Because of the above syntax I get the following:

* The download only starts when the data starts being observed. It is therefore the equivalent of RxJava's `doOnSubscribe()`, which only starts the execution when there is a subscriber.
* The code, however, does not end when the last observer unsubscribes, so it will not delete the downloaded data when the `Fragment` enters paused stade. The developer may control the behavior by modifying the value set to `timeoutInMs`. The value set to it indicates the timeout that has to pass afer the last observer is removed (for instance when screen is rotated or application is paused), without adding new observes, for the data contained therein do be discarded.
* The download is canceled and the already downloaded data, if any, is discarded, when `viewModelCoroutineContext` is canceled.

## The GUI

This is the way the above `LiveData` is used with the GUI:

```kotlin
(list.ld to this) { response ->
    if (response is ListVM.Status.Success) {
        progress.visibility = View.GONE
        recycler.visibility = View.VISIBLE
        recycler.verticalDivider()
        recycler.adapter = LaunchPadAdapter(response.list, lifecycleScope) { position ->
            single.launchPad = response.list[position]
            nav(R.id.action_mainFragment_to_mapFragment)
        }
        recycler.layoutManager = LinearLayoutManager(context)
    }
    else {
        getString(R.string.launch_pads_network_error).toast()
    }
}
```

The code does use some DSL, but I hope it is fine with the reader, as the DSL parts have been described previously in this blog. In either case, the reader is welcome to investigate the code further by cloning my [project][launchpads] to their local machine.

## The map

Clicking on each item displayed in the `RecyclerView` does lead the user to a `Fragment` with Google Maps presenting its location, but discussing the code that does it outside of the scope of the present article. It doesn't really matter how I do it. I only used navigation to another `Fragment` as a way of demonstrating what happens to the contents of the original `Fragment` when it is hidden.

Displaying Google Maps has already been discussed thoroughly in [another article][maps] in this blog.

## Conclusion

The article has shown how to solve common problems in asynchronous data loading:

* How to download a JSON, and immediately start downloading individual JSONs per every item contained in the original one.
* Where to store such data while it is still being downloaded. For this I used `Deferred`.
* How to use [Picasso] with such `Deferred`. How to prioritize using it without suspending on `await()` to avoid briefly showing the placeholder unnecessarily.

I tried to demonstrate thru this article how to simultaneously use three levels of asynchronous data downlading: Two levels with Retrofit and one with Picasso.

## Donations

If the reader has enjoyed the present article, they may want to donate some bitcoin at the address presented below. Readers may also look at my [donations page][donations].

BTC: bc1qncxh5xs6erq6w4qz3a7xl7f50agrgn3w58dsfp

[retrofit]: https://github.com/square/retrofit
[spacex]: https://docs.spacexdata.com/
[launchpads]: https://github.com/syrop/Launch-Pads
[play]: https://play.google.com/store/apps/details?id=pl.org.seva.launchpads
[blog]: https://android.jlelse.eu/kotlin-coroutines-and-retrofit-e0702d0b8e8f
[cliparts]: http://clipart-library.com/outer-space-cliparts.html
[wikipedia]: https://www.mediawiki.org/wiki/API:Main_page
[maps]: https://syrop.github.io/jekyll/update/2018/12/28/displaying-google-maps.html
[picasso]: https://github.com/square/picasso
[donations]: https://syrop.github.io/donate/
