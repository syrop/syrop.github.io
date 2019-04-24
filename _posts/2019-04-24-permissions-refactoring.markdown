---
layout: post
title:  "Permissions handling (refactoring)"
date:   2019-04-24 21:53:00 +0200
categories: jekyll update
---

This article shows how to refactor the code handling permissions.

## Previous articles

Handling permissions was already described on 26th of December, 2018, in a [separate article][permissions-article] in this blog. The present article is a refactoring of than solution.

This solution is used in mapholder pattern, also descibed in a [separate article][mapholder-article].

## The project

The project I am discussing in the present article is [Victor-Events][victor-events], but because it is still work in progress, and it is already rather large now, you will probably want to just focus on the code presented in the present article instead of cloning the whole project.

## The problem

The purpose of the refactoring is to remove dependency on RxJava from the project. I think it is not fair to add dependenry on RxJava to the project, only for the sake of handling permissions.

## The request

This is the code of the permissions request. It contains `String` representation of the requested permission, an action called when the permission is granted, and an action called when the permission is denied. By default, both actions are empty:

{% highlight kotlin %}

class PermissionRequest(
        val permission: String,
        val onGranted: () -> Unit = {},
        val onDenied: () -> Unit = {})

{% endhighlight %}

Screen rotation will deliberately cancel the request, therefore preventing an invocation of either action.

Because the actions may contain references to the actual `Fragment`, they should be canteled as soon as the `Fragment` is destroyed.

You may want to implement requesting the permission again, if such need is indicated by the state retained inside the `ViewModel`.

In this particular [project][victor-events] permission is requested when the `ViewModel` holds information about event's location. If such location has been entered, then the `Fragment` in the first step displays the location on the map, then requests location permission, and if granted - displays current location of the device on the screen as well.

Therefore, even though each particular request is canceled each time the screen is rotated, the same permission is immediately requested again, thanks to state held in the `ViewModel`.

## The result

This is the result of the request discussed in the above section: 

{% highlight kotlin %}

data class PermissionResult(val requestCode: Int, val permission: String)

{% endhighlight %}

For historical reasons, the class `pl.org.seva.events.main.model.Permissions` uses two `Chanenl`s, one for granted, the other for denied permissions. The [original solution][permissions-article] contained two separate RxJava's `Subject`s, so I decided to refactor the original code as little as possible. Alternatively, I could use two separate `sealed` classes for represending the result, and send their instances over one `Channel`, but I decided not to change the [original design][permissions-article] without a good reason.

## Watching the result channels

This is the code watching results of every callback, and depending on the `Channel` over which the result is reported, calls the appropriate action, ether `onGranted()` or `onDenied()`:

{% highlight kotlin %}

class ViewModel : androidx.lifecycle.ViewModel() {
    val granted by lazy { BroadcastChannel<PermissionResult>(DEFAULT_CAPACITY) }
    val denied by lazy { BroadcastChannel<PermissionResult>(DEFAULT_CAPACITY) }

    private fun CoroutineScope.watch(code: Int, request: PermissionRequest) = launch (Dispatchers.IO) {
        suspend fun PermissionResult.ifMatching(block: () -> Unit) {
            if (requestCode == code && permission == request.permission) {
                withContext(Dispatchers.Main) {
                    block()
                }
            }
        }
        val grantedSubscription = granted.openSubscription()
        val deniedSubscription = denied.openSubscription()

        while (true) {
            select<Unit> {
                grantedSubscription.onReceive {
                    it.ifMatching { request.onGranted() }
                }
                deniedSubscription.onReceive {
                    it.ifMatching { request.onDenied() }
                }
            }
        }
    }

    fun watch(code: Int, requests: Array<PermissionRequest>) = viewModelScope.launch {
            for (req in requests) {
                watch(code, req)
            }
        }

    companion object {
        private const val DEFAULT_CAPACITY = 10
    }
}

{% endhighlight %}

I decided to use a `ViewModel`, because it gives me a `viewModelScope` for free. The `public` function `watch()` returns a `Job`, which is not only already contained within the `viewModelScope`, but also can be canceled by external code.

The function launches as many child `Job`s as many requests there are. Each such child `Job`, thanks to `select()`, watches simultaneously the `granted` `BroadcastChannel` and the `denied` one.

Please note that because `BoradcastChannel`s are experimental, I add the following annotation at the beginning of the file:

{% highlight kotlin %}

@file:Suppress("EXPERIMENTAL_API_USAGE")

{% endhighlight %}

## Processing the results

This is the code creating instances of `PermissionResul`, and sending them to appropriate `Channel`:

{% highlight kotlin %}

fun onRequestPermissionsResult(
        fragment: Fragment,
        requestCode: Int,
        permissions: Array<String>,
        grantResults: IntArray) {

    val vm = fragment.getViewModel<ViewModel>()
    val granted = vm.granted
    val denied = vm.denied

    infix fun String.onGranted(requestCode: Int) =
            granted.sendBlocking(PermissionResult(requestCode, this))

    infix fun String.onDenied(requestCode: Int) =
            denied.sendBlocking(PermissionResult(requestCode, this))

    if (grantResults.isEmpty()) {
        permissions.forEach { it onDenied requestCode }
    } else repeat(permissions.size) { id ->
        if (grantResults[id] == PackageManager.PERMISSION_GRANTED) {
            permissions[id] onGranted requestCode
        } else {
            permissions[id] onDenied requestCode
        }
    }
}

{% endhighlight %}

## In the Fragment

This is the function inside the `Fragment` that checks whether location permission has already been granted. If so, it calls `onGranted()`, if it is not - requests the permission and schedules `onGranted()` to be called only when the permission is granted.

{% highlight kotlin %}

private fun checkLocationPermission(onGranted: () -> Unit) {
    if (ContextCompat.checkSelfPermission(
                    context!!,
                    Manifest.permission.ACCESS_FINE_LOCATION) == PackageManager.PERMISSION_GRANTED) {
        onGranted()
    }
    else {
        request(
                Permissions.DEFAULT_PERMISSION_REQUEST_ID,
                arrayOf(Permissions.PermissionRequest(
                        Manifest.permission.ACCESS_FINE_LOCATION,
                        onGranted = onGranted)))
    }
}

{% endhighlight %}

This is the code handling the responses inside the `Fragment`:

{% highlight kotlin %}

override fun onRequestPermissionsResult(
        requestCode: Int,
        requests: Array<String>,
        grantResults: IntArray) =
        permissions.onRequestPermissionsResult(this, requestCode, requests, grantResults)

{% endhighlight %}

These are the extension functions the `Fragment` calls to request the permission and watch the results:

{% highlight kotlin %}

fun Fragment.request(requestCode: Int, requests: Array<Permissions.PermissionRequest>) {
    watch(requestCode, requests)
    requestPermissions(requests.map { it.permission }.toTypedArray(), requestCode)
}

private fun Fragment.watch(requestCode: Int, requests: Array<Permissions.PermissionRequest>) {
    val vm = getViewModel<Permissions.ViewModel>()
    untilDestroy { vm.watch(requestCode, requests) }
}

private fun Fragment.untilDestroy(work: () -> Job) = work().apply {
    lifecycle.addObserver(object : LifecycleEventObserver {
        override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
            if (event == Lifecycle.Event.ON_DESTROY) {
                cancel()
            }
        }
    })
}

{% endhighlight %}

Notice the function `Fragment.untilDestroy()` that creates an insnance of `Job`, and then creates a `LifeCycleObserver` to cancel the `Job` on screen rotation. The `Job` is already canceled when the `viewModelScope` of `Permissions.ViewModel` is canceled.

## Conclusion

Coroutines are a viable alternative to RxJava.

The above solution uses a code that is deliberately similar to the one presented [previously][permissions-article].

There isn't much benefit in refactoring RxJava to coroutines in this simple case. I mostly did it to question whether it makes sense to add RxJava to a project only to perform one simple task, which is permission requesting, which happened to be the only case in my [project][victor-events] where RxJava was still being used.

If RxJava is already being used in your project for other things, and you don't have to add this dependency only for the sake of permissions, then probably both the present solution and the [previous one][permissions-article] are equally fine.

[permissions-article]: https://syrop.github.io/jekyll/update/2018/12/26/permissions-rxjava-and-lifecycle.html
[mapholder-article]: https://syrop.github.io/jekyll/update/2019/01/09/selecting-location-advanced-topic.html
[victor-events]: https://github.com/syrop/Victor-Events

