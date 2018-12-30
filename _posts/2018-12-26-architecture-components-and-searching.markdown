---
layout: post
title:  "Architecture components and searching"
date:   2018-12-26 8:42:00 +0100
categories: jekyll update
---
In this article I will explain how to use [Architecture components][architecture-components], specifically [Navigation][navigation] for creation of a [Fragment][fragment] using a searching UI.

As an example I will use the project [Viktor-Events][events]. Please note that it is not production ready, so you will not find it on Google Play, but you can clone the repository, compile it and run it from Android Studio.

## Adding Navigation

To use Navigation (Architecture component), use the following dependencies:

{% highlight groovy %}

implementation 'android.arch.navigation:navigation-fragment-ktx:1.0.0-alpha09'
implementation 'android.arch.navigation:navigation-ui-ktx:1.0.0-alpha09'

{% endhighlight %}

Unfortunately, the libraries are still in alpha, but I've been able to use it in this project, as well as in the production-ready [Wiktor-Navigator][navigator].

The [navigation graph][navigation-graph], though too big to embed in this article, can be vieved in XML form directly on GitHub.

The code that actually embeds the `Fragment`s, using Navigation Architecture component, into their `Activity` is as follows.

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".main.EventsActivity">

    <com.google.android.material.appbar.AppBarLayout
        android:id="@+id/appbar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:theme="@style/AppTheme.AppBarOverlay"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toTopOf="@+id/nav_host_fragment">

        <androidx.appcompat.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            app:popupTheme="@style/AppTheme.PopupOverlay"/>

    </com.google.android.material.appbar.AppBarLayout>

    <fragment
        android:layout_width="match_parent"
        android:layout_height="0dp"
        app:layout_constraintTop_toBottomOf="@+id/appbar"
        app:layout_constraintBottom_toBottomOf="parent"
        android:id="@+id/nav_host_fragment"
        android:name="androidx.navigation.fragment.NavHostFragment"
        app:navGraph="@navigation/nav_graph"
        app:defaultNavHost="true"
        />

</androidx.constraintlayout.widget.ConstraintLayout>

{% endhighlight %}

## Implementing the Activity and Fragment

The `Activity` and `Fragment` share one instance of `ViewModel`:

{% highlight kotlin %}

class EventsViewModel : ViewModel() {
    val query: MutableLiveData<String> = MutableLiveData()
    val commToCreate: MutableLiveData<String?> = MutableLiveData()
}

{% endhighlight %}

An instance of it is acquired in the `Activity` this way:

{% highlight kotlin %}

private lateinit var eventsModel: EventsViewModel

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    ...
    eventsModel = ViewModelProviders.of(this).get(EventsViewModel::class.java)
    ....
}

eventsModel = ViewModelProviders.of(this).get(EventsViewModel::class.java)

{% endhighlight %}

And in the `Fragment` this way, together with observing the value of `query`:

{% highlight kotlin %}

private lateinit var eventsModel: EventsViewModel

override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    eventsModel = ViewModelProviders.of(activity!!).get(EventsViewModel::class.java)
    eventsModel.query.observe(this) { query ->
        if (!query.isEmpty()) {
            eventsModel.query.value = ""
            search(query)
        }
    }
}

{% endhighlight %}

Please note the shared instance of `ViewModel` needs to be retrieved specifically in the function `onViewCreated()`, and not in `onCreateView()`.

Setting the value of `query` is performed in the `Activity`'s function `onNewIntent()`:

{% highlight kotlin %}

override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    if (Intent.ACTION_SEARCH == intent.action) {
        eventsModel.query.value = intent.getStringExtra(SearchManager.QUERY)
    }
}

{% endhighlight %}

Because both `Activity` and the `Fragment` share the same instance of the `ViewModel`, the `Fragment` may listen to the value changes of the `LiveData` that is set in the `Activity`.

## Configuring the Activity in AndroidManifest.xml

This is the way the `Activity` is configured in `AndroidManifest.xml` so that it handles `android.intent.action.SEARCH` `Intent`s, and that only one instance of it may be run (the parameter `singleTop`):

{% highlight xml %}

<activity
        android:name=".main.EventsActivity"
        android:label="@string/app_name"
        android:launchMode="singleTop"
        android:theme="@style/AppTheme.NoActionBar">
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
            <category android:name="android.intent.category.LAUNCHER"/>
        </intent-filter>
        <intent-filter>
            <action android:name="android.intent.action.SEARCH" />
        </intent-filter>
        <meta-data android:name="android.app.searchable"
                   android:resource="@xml/searchable_community"/>
</activity>

{% endhighlight %}

This is the content of the XML file configuring searchable content in the `Activity`:

{% highlight xml %}

<?xml version="1.0" encoding="utf-8"?>
<searchable xmlns:android="http://schemas.android.com/apk/res/android"
            android:label="@string/searchable_add_community"
            android:hint="@string/searchable_add_community_hint">
</searchable>

{% endhighlight %}

## Adding the Search option menu in the Fragment

This is the code that adds Search option menu to the `Fragment`:

{% highlight kotlin %}

private val searchManager get() =
        activity!!.getSystemService(Context.SEARCH_SERVICE) as SearchManager

override fun onCreateOptionsMenu(menu: Menu, menuInflater: MenuInflater) {
    fun MenuItem.prepareSearchView() = with (actionView as SearchView) {
        setSearchableInfo(searchManager.getSearchableInfo(activity!!.componentName))
        setOnSearchClickListener { onSearchClicked() }
        setOnCloseListener { onSearchViewClosed() }
    }
        
    menuInflater.inflate(R.menu.add_community, menu)
    val searchMenuItem = menu.findItem(R.id.action_search)
    searchMenuItem.collapseActionView()
    searchMenuItem.prepareSearchView()
}

{% endhighlight %}

This is, finally, the XML file that adds the search edit box to the GUI as an option menu:

{% highlight xml %}

<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:app="http://schemas.android.com/apk/res-auto">
    <item
        android:id="@+id/action_search"
        android:orderInCategory="100"
        android:icon="@drawable/ic_search_white_24dp"
        android:title="@string/action_search"
        app:showAsAction="always"
        app:actionViewClass="androidx.appcompat.widget.SearchView"/>
</menu>


{% endhighlight %}

[architecture-components]: https://developer.android.com/topic/libraries/architecture/
[navigation]: https://developer.android.com/topic/libraries/architecture/navigation.html
[fragment]: https://developer.android.com/reference/kotlin/androidx/fragment/app/Fragment
[events]: https://github.com/syrop/Victor-Events
[navigator]: https://github.com/syrop/Wiktor-Navigator
[navigation-graph]: https://github.com/syrop/Victor-Events/blob/master/events/src/main/res/navigation/nav_graph.xml
