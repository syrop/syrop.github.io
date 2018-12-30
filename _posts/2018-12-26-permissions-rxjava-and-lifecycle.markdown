---
layout: post
title:  "Permissions, RxJava and lifecycle"
date:   2018-12-26 8:43:00 +0100
categories: jekyll update
---
This article will demonstrate how to delegate permission handling to a designated class whose instance in retrieved using dependency injection with [Kodein][kodein-di].

I will show how to implement and then call a fuction that takes in a list of permissions to request. Each such permission will be accompanied with one action that will be called when the permission has been granted, and one that will be called whet it has been denied.

Configuration of dependency injection with [Kodein][kodein-di] has been already described in this blog in the article called '[Testing with dependency injection][testing]'.

Presented herein permission-handling code has been used in the project [Wiktor-Navigator][navigator].

## Structure of the request

The class containing the request contains the name of the request, the action called when it has been granted, the action called when it has beed denied, and doesnt't contain any fuction. By default both of these actions are empty:

{% highlight kotlin %}

class PermissionRequest(
            val permission: String,
            val onGranted: () -> Unit = {},
            val onDenied: () -> Unit = {})

{% endhighlight %}

## Requesting the permissions

Permissions are requested by calling a global extention function called on the `Fragment`. Because I use [Navigation][navigation] Architecture Component almost all of GUI is implementet in `Fragment`s as opposed to `Activity`s, but one could write a similar extention function for `Activity` as well:

{% highlight kotlin %}

fun Fragment.requestPermissions(
        requestCode: Int,
        permissions: Array<Permissions.PermissionRequest>) =
    permissions().request(
            activity!! as AppCompatActivity,
            requestCode,
            permissions)

{% endhighlight %}

The call `permissions()` automatically retrieves an instance of the permission-handling class, whose Kodein binding has been created in the module by this line:

{% highlight kotlin %}

bind<Permissions>() with singleton { Permissions() }

{% endhighlight %}

It is then retrieved by this global function:

{% highlight kotlin %}

fun permissions() = instance<Permissions>()

{% endhighlight %}

This is the code that actually requests the permissions and subscribes the responses using RxJava:

{% highlight kotlin %}

class Permissions {

    private val grantedSubject = PublishSubject.create<PermissionResult>()
    private val deniedSubject = PublishSubject.create<PermissionResult>()

    fun request(
            activity: AppCompatActivity,
            requestCode: Int,
            permissions: Array<PermissionRequest>) {
        val lifecycle = activity.lifecycle
        val permissionsToRequest = ArrayList<String>()
        permissions.forEach { permission ->
            permissionsToRequest.add(permission.permission)
                grantedSubject
                        .filter { it.requestCode == requestCode && it.permission == permission.permission }
                        .subscribe(lifecycle) { permission.onGranted() }
                deniedSubject
                        .filter { it.requestCode == requestCode && it.permission == permission.permission }
                        .subscribe(lifecycle) { permission.onDenied() }
        }
        ActivityCompat.requestPermissions(activity, permissionsToRequest.toTypedArray(), requestCode)
    }
    ...
}

{% endhighlight %}

It uses `lifecycle` to automatically dispose the subscriptions when activity ends. Making lifecycle-aware subscribtions has been described in this blog in the article called '[Lifecycle-aware Rx subscriptions][lifecycle]'.

## In the Fragment

This is the code in the `Fragment` that requests Location permission, passing to the request the actions that will be called when it is granted or denied:

{% highlight kotlin %}

private fun requestLocationPermission() {
        fun showLocationPermissionSnackbar() {
            snackbar = Snackbar.make(
                    coordinator,
                    R.string.snackbar_permission_request_denied,
                    Snackbar.LENGTH_INDEFINITE)
                    .setAction(R.string.snackbar_retry)  { requestLocationPermission() }
                    .apply { show() }
        }

        requestPermissions(
                Permissions.LOCATION_PERMISSION_REQUEST_ID,
                arrayOf(Permissions.PermissionRequest(
                        Manifest.permission.ACCESS_FINE_LOCATION,
                        onGranted = ::onLocationPermissionGranted,
                        onDenied = ::showLocationPermissionSnackbar)))
}

{% endhighlight %}

If the permission is denied, a snackbar will be shown, prompting the user again to grant the permission. When the user chooses to automatically deny the permission each time, this snackbar will just continue to show up, suggesting to the user that this permission is mandatory for the application to function.

## Handling responses

Handling the response begins in the `Fragment`:

{% highlight kotlin %}

override fun onRequestPermissionsResult(
            requestCode: Int,
            permissions: Array<String>,
            grantResults: IntArray) =
            permissions().onRequestPermissionsResult(requestCode, permissions, grantResults)

{% endhighlight %}

It retrieves an instance of the permissions-handling class and passes the results to it. The actual permission-handling code:

{% highlight kotlin %}

fun onRequestPermissionsResult(requestCode: Int, permissions: Array<String>, grantResults: IntArray) {
    infix fun String.onGranted(requestCode: Int) =
            grantedSubject.onNext(PermissionResult(requestCode, this))

    infix fun String.onDenied(requestCode: Int) =
            deniedSubject.onNext(PermissionResult(requestCode, this))

    if (grantResults.isEmpty()) {
        permissions.forEach { it onDenied requestCode }
    } else repeat(permissions.size) {
        if (grantResults[it] == PackageManager.PERMISSION_GRANTED) {
            permissions[it] onGranted requestCode
        } else {
            permissions[it] onDenied requestCode
        }
    }
}

{% endhighlight %}

It just calls `onNext` on the appropriate RxJava's `Subject`s, and the previously described code handles the subscriptions, disposing them if they are already irrelevant due to the `Activity`'s finished `lifeficle`.

[kodein-di]: https://kodein.org/di/
[testing]: https://syrop.github.io/jekyll/update/2018/12/25/testing-with-dependency-injection.html
[navigator]: https://github.com/syrop/Wiktor-Navigator
[navigation]: https://developer.android.com/topic/libraries/architecture/navigation.html
[lifecycle]: http://localhost:4000/jekyll/update/2018/12/25/lifecycle-aware-rx-subscriptions.html
