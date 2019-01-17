---
layout: post
title:  "TextInputEditText and ViewModel"
date:   2019-01-17 21:34:00 +0100
categories: jekyll update
---

This article will explain how to easily create a connection between an instance of `TextInputEditText` and `LiveData`, which is a part of a `ViewModel`.

Thanks to this connection any data typed manually in the `TextInputEditText` will be automatically saved inside of `LiveData`, so that it will be automatically reinserted in the GUI when the `Fragment` is recreated, or when the same `ViewModel` is being used in another `Fragment`.

The design pattern used herein has been implemented in my project [Victor-Events][victor-events].

## ViewModel

The following code snippet demonstrates the `ViewModel` that will be used in this article.

An earlier version of this `ViewModel` has been already described in my earlier article '[Selecting location][selecting-location]' in this blog. The difference is that in the present article I added `location`.

{% highlight kotlin %}

class CreateEventViewModel : ViewModel() {

    val comm by lazy {
        MutableLiveData<String>()
    }

    val time by lazy {
        MutableLiveData<LocalTime>()
    }

    val date by lazy {
        MutableLiveData<LocalDate>()
    }

    val location by lazy {
        MutableLiveData<EventLocation?>()
    }

    val description by lazy {
        MutableLiveData<String?>()
    }

    fun observeLocation(owner: LifecycleOwner, mapHolder: MapHolder) {
        location.observe(owner, Observer { mapHolder.putMarker(it?.location) })
    }
}

{% endhighlight %}

## Lazily providing the ViewModel

An instance of the ViewModel can be procured using the following code:

{% highlight kotlin %}

private val model by lazy { viewModel<CreateEventViewModel>() }

{% endhighlight %}

If you are using this code a `Fragment`, you should not try to access the value of `model` from the function `onCreateView()`, because then you will not be able to get an instance of the `ViewModel`. I access the value of `model` for the first time in the function `onViewCreated()`, so that by that time an instacne of `ViewModel` may be correctly provided.

Below is the code providing the actual instance of `ViewModel`. I put this code in files that I designate specifically for extension functions:

{% highlight kotlin %}

inline fun <reified R : ViewModel> Fragment.viewModel() =
        activity!!.viewModel<R>()

inline fun <reified R : ViewModel> FragmentActivity.viewModel() =
        ViewModelProviders.of(this).get(R::class.java)

{% endhighlight %}

## XML file containing the TextInputLayout

This is the XML code defining the `TextInputLayout` that will be programmatically attached to an instance of `LiveData`:

{% highlight xml %}

<com.google.android.material.textfield.TextInputLayout
        android:id="@+id/description_layout"
        android:layout_height="wrap_content"
        android:layout_width="match_parent"
        android:layout_below="@+id/address_layout">

        <com.google.android.material.textfield.TextInputEditText
            android:id="@+id/description"
            android:layout_height="wrap_content"
            android:layout_width="match_parent"
            android:hint="@string/event_description"/>
    </com.google.android.material.textfield.TextInputLayout>

{% endhighlight %}

## Connecting to LiveData

The above `TextInputEditText` can be connected to an instance of `LiveData` by this simple line, which I call from the function `onViewCreated()`:

{% highlight kotlin %}

description.withLiveData(this, model.description)

{% endhighlight %}

Before `model` is initiated lazily, an instance of `ViewModel` is provided only when the above line is called.

Below is the code that creates the actual connection.

{% highlight kotlin %}

fun TextInputEditText.withLiveData(owner: LifecycleOwner, liveData: MutableLiveData<String?>) {
    watch { liveData.value = text.toString() }
    liveliveData.observe(owner, Observer { value ->
        if (value == text.toString()) {
            return@Observer
        }
        setText(value)
    })
}

private fun TextInputEditText.watch(afterTextChanged: Editable.() -> Unit) {
    addTextChangedListener(object : TextWatcher {
        override fun afterTextChanged(s: Editable) {
            s.afterTextChanged()
        }

        override fun beforeTextChanged(s: CharSequence, start: Int, count: Int, after: Int) = Unit

        override fun onTextChanged(s: CharSequence, start: Int, before: Int, count: Int) = Unit
    })
}

{% endhighlight %}

I will first explain the function `TextInputEditText`. Because this function does not belong to any class (it is considered global) and is at the same time `private`, it is visible only to other global functions defined in the same file.

I create an instance of [TextWatcher][textwatcher], and because `Unit` is in Kotlin just a shorthand for 'no relevant value', I just write `= Unit` for every function defined in the `interface` I do not care about the execution of.

This leaves me with a need to implement only one function, `afterTextChanged()`, which just calls the lambda with receiver that has been passed as a parameter to `watch()`. (Here the receiver of this lambda is an instance of `Editable`).

Notice in the above code the line:

{% highlight kotlin %}

watch { liveData.value = text.toString() }

{% endhighlight %}

It ensures that whenever the text of the `TextItputEditText` changes, the code `liveData.value = text.toString()`, which will save the new text value in the `LiveDada`.

In the next line, `observe()` is called on `LiveData`, so that observer is created setting text to the `TextInputEditText`.

To avoid an infinite loop, a check needs to be made that the text is not being set to the value that is already there. This condition will be `true` if the observer is called after setting the text automatically by `setText(value)` herein, as opposed to manually from Android's virtual keyboard.

## Test it

You can test the design pattern presented in this article by commenting out the code `description.withLiveData(this, model.description)`. After you comment this out, the content you just entered will be retained when you rotate the screen or when you pause the application, but **not** when you destroy the fragment, or when you try to pass the entered data to another fragment that is supposed to use the same `ViewModel`.

I tested it in the project [Victor-Events][victor-events] in this way that I entered the event creation `Fragment`, manually entered a random value in the event description and left the `Fragment` by pressing [Back].

When I did't call the function `withLiveData()` on the `description` `View`, data was not retained.

When I did call `withLiveData(this, model.description)` on the `View`, data was correctly restored being restored on recreation of the `Fragment`.

## What you still need to implement

When you have implemented saving the entered data into an instance of `ViewModel`, so that it can be passed between `Fragment`s, or restored when a `Fragment` is being recreated, you probably should also implement persisting the data locally, or uploading it to the cloud.

Doing this is outside of the scope of this article, as you can just use your favourite method of persistence.

However, you may actually want to delete the data being stored within the `ViewModel` when the user leaves the `Fragment` without confirming the changes.

In case of `description`, to add a possibility to clear the data, you can add the following member function to the code of `CreateEventViewModel`:

{% highlight kotlin %}

class CreateEventViewModel : ViewModel() {

    ...

    val description by lazy {
        MutableLiveData<String?>()
    }

    fun clear() {
        description.value = ""
        ...
    }

    ...
}

{% endhighlight %}

Because `description` is of type `MutableLiveData<String?>`, instead of clearing its value by setting it to an empty string, you may just as well set it to `null`, depending on the effect you want to achieve.

[victor-events]: https://github.com/syrop/Victor-Events
[selecting-location]: https://syrop.github.io/jekyll/update/2019/01/09/selecting-location-advanced-topic.html
[textwatcher]: https://developer.android.com/reference/android/text/TextWatcher

