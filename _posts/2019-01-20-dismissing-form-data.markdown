---
layout: post
title:  "Dismissing form data"
date:   2019-01-20 16:04:00 +0100
categories: jekyll update
---

This article will explain how to dismiss data entered in a form when you leave the current fragment.

Assumption is made that you already know how to back in a `ViewModel` the data you enter in a form.

This article is a continuation of the [previous one][TextInputEditText] in which I explain how to store in an instance of `ViewModel` the data you enter in a form.

I used this information in my project [Victor-Events][victor-events], more specifically in the commit [8cdef88][commit] fixing issue [#6][issue].

## ViewModel

This is the `ViewModel` used in this article:

{% highlight kotlin %}

class CreateEventViewModel : ViewModel() {

    val comm by lazy {
        MutableLiveData<String>()
    }

    val name by lazy {
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

    ...

    val isFilledIn get() = eventData.any { it.value != null }

    fun clear() = eventData.onEach {
        it.value = null
    }

    private val eventData by lazy {
        listOf(name, time, date, location, description)
    }
}

{% endhighlight %}

Please note the last three function and properties.

`eventDada` is the immutable list of all elements that will be check to determine whether the form has been edited at all, and also the list of elements that need to be cleared when the form data is dismissed. One of the fields in this `ViewModel`, the one called `comm`, has been deliberately omitted.

There is a property `isFilledIn` that uses that list to see whether any of the elements has been filled in, and `clear()`, which clears them all.

## Leaving the fragment

I use [Navigation Architecture Component][navigation] to switch between `Fragment`s.

Because the buttons [Back] and [Home] are handled by the `Activity` as opposed to `Fragment`, the code handling them will be placed in the `Activity`. Still, to determine which `Fragment` is currently displayed on the screen, I use `NavController`:

{% highlight kotlin %}

private val nav by lazy { findNavController(R.id.nav_host_fragment) }

{% endhighlight %}

This is the code inside the `Activity` that handless pressing [Home] and [Back]. I assume that in your particular application they are to work identically:

{% highlight kotlin %}

private fun showDismissEventDialog(): Boolean {
    return if (nav.currentDestination?.id == R.id.createEventFragment && createEventsViewModel.isFilledIn) {
        question(
                message = getString(R.string.events_activity_dismiss_event),
                yes = {
                    createEventsViewModel.clear()
                    nav.popBackStack()
                })
        true
    }
    else false
}

override fun onBackPressed() {
    if (!showDismissEventDialog()) {
        super.onBackPressed()
    }
}

override fun onOptionsItemSelected(item: MenuItem): Boolean {
    return if (item.itemId == android.R.id.home) showDismissEventDialog() else false
}

{% endhighlight %}

The code checks what `Fragment` is displayed currently. If it is a predifined `Fragment` containing the form, it displays a confirmation dialog and prevents switching the screen to anoter `Fragment` before the dialog is closed.

## Displaying yes/no questions

For convenience, I createn an extension function that creates and shows a very simple instance of `AlertDialog`:

{% highlight kotlin %}

fun Context.question(message: String, yes: () -> Unit = {}, no: () -> Unit = {}) {
    YesNoListener(yes, no).apply {
        AlertDialog.Builder(this@question)
                .setMessage(message)
                .setPositiveButton(android.R.string.yes, this)
                .setNegativeButton(android.R.string.no, this).show()
    }
}

private class YesNoListener(
        private val yes: () -> Unit = {},
        private val no: () -> Unit = {}): DialogInterface.OnClickListener {
    override fun onClick(dialog: DialogInterface, which: Int) = when (which) {
        DialogInterface.BUTTON_POSITIVE -> {
            dialog.dismiss()
            yes()
        }
        DialogInterface.BUTTON_NEGATIVE -> {
            dialog.dismiss()
            no()
        }
        else -> Unit
    }
}

{% endhighlight %}

I will first describe the code used to create the `OnClickListener`.

Constructor of the class `YesNoListener` takes in two lambdas, one to call in case of confirmation, the other in case of cancellation. They are both empty by default, so if you do not explicitly provide them, the dialog will be just dismissed, without performing any other action.

I used in the above code a `when` expression, which must be exhaustive, so I provided the most simple `else` branch I could think of.

It simply retunrs `Unit` vaule, which in Kotlin means that it will do nothing at all. You could just as well write `else -> dialog.dismiss()` to close the dialog, or even throw an `IllegalArgumentException`. In this particular example this branch will be never used anyway, because I create a dialog with only two buttons, but you do need to think about it when you use this `OnClickListener` in a dialog that has extra buttons. You may just as well create a `default` lambda expression for this case, just as I created `yes` and `no`.

The `question()` function defined above creates and shows an `AlertDialog` with a custom message and two buttons which call the `yes` and `no` actions passed to the function as parameters, which are empty by default.

In the `question()` function I wrote `YesNoListener(yes, no).apply { ... }`, because I want to use the same instance of `YesNoListener` twice.

Alternatively, I could write `val yesNo = YesNoListener(yes, no)`, and then refer to the listener by name, but then the data flow would have been less obvious to me. I would have to remember which name I used to store the instance, and then try to find out what the scope of the constant is. Whenever I see the function `apply()` I know that I only have to find the corresponding `}` bracket to know when the value is going to be dismissed, unless I have used `val` at the same time.

I do use `val`, however, when I want to store more than one constant value, or when I know I am going to store the value longer that just in the most immediate block of code that follows the creation of the instance.


## Clearing the fields in the GUI

When you pass `null` values to `LiveData`, you have to be particularly careful how the listeners react to them. For each field of the form you need to implement displaying whatever you understand as its empty value.

Please note that for `LiveData` it doesn't really matter whether you define it as `MutableLiveData<String>()` or `MutableLiveData<String?>()`, as you can pass `null` values to it either way.

This is the code I used to display the formatted time and date when appropriate, and an empty `String` for `null` values:

{% highlight kotlin %}

fun onTimeChanged(t: LocalTime?) = time.setText(if (t == null) "" else "${t.hour}:${t.minute}")
fun onDateChanged(d: LocalDate?) = date.setText(if (d == null) "" else "${d.year}-${d.monthValue}-${d.dayOfMonth}")

model.time.observe(this, Observer { onTimeChanged(it) })
model.date.observe(this, Observer { onDateChanged(it) })

{% endhighlight %}

Please note that if you use `ViewModel` in `Fragments`, you should not do so in the function `onCreateView()`, but you can do it it `onViewCreated()` as the model is going to be available by then.

## Donations

If you've enjoyed this article, consider donating some bitcoin at the address below. You can also look at my [donations page][donations].

BTC: bc1qncxh5xs6erq6w4qz3a7xl7f50agrgn3w58dsfp 

[victor-events]: https://github.com/syrop/Victor-Events
[TextInputEditText]: https://syrop.github.io/jekyll/update/2019/01/17/TextInputEditText-and-LiveData.html
[commit]: https://github.com/syrop/Victor-Events/commit/8cdef8885fe46c0da4516590fc397e1dcceb5fb0
[issue]: https://github.com/syrop/Victor-Events/issues/6
[navigation]: https://developer.android.com/topic/libraries/architecture/navigation.html
[donations]: https://syrop.github.io/donate/
