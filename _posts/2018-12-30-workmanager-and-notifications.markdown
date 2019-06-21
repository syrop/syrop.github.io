---
layout: post
title:  "WorkManager and notifications"
date:   2018-12-30 21:12:00 +0100
categories: jekyll update
---

This is another article in the series of articles about [Android Jetpack][jetpack].

I created this design pattern for a professional project I was involved in. I am not using it in my [GitHub profile][github].

## Definitions

In this article I will use the terms 'push notification' and 'notification' interchangeably, as this is really the same things.

When [Firebase Cloud Messaging][fcm] is meant, I will make it explicit in the article.

## WorkManager

To add [WokrManager][workmanager] to your project, please refer to [Jetpack documentation][jetpack]. In this article a basic understanding of [Jetpack][jetpack] is assumed.

## Creating notification channels

I use one extention function to create all notification channels:

{% highlight kotlin %}

fun Context.createNotificationChannels() {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) return
    createSignUpIncomplete()
    createSubscriptionIncomplete()
    createFcm()
    createPlayedFirstTrack()
    createNoInteraction()
}

{% endhighlight %}

You can call it from `onCreate` function inside your `Application` class:

{% highlight kotlin %}

override fun onCreate() {
    super.onCreate()
    createNotificationChannels()
}

{% endhighlight %}

Because it is an extention function, an instance of your application `Context` will be automatically passed to it.

If you want to use a more advanced design pattern to initialize your application, together with dependency retrieval, you can create an instance of the class `Bootstrap` I explain in the article '[Testing with dependency retrieval][testing]' in this blog.

This is the part of your `Bootstrap` class that calls the function creating the channels:

{% highlight kotlin %}

class Bootstrap(private val ctx: Context) {

    fun boot() {
        ctx.createNotificationChannels()
    }
}

{% endhighlight %}

The function creating each individual channel is **private** within the same file that defines the function `createNotificationChannels`, so you can't create each channel individually, you can only create them simultaneously. This is one of such functions:

{% highlight kotlin %}

@RequiresApi(Build.VERSION_CODES.O)
private fun Context.createSignUpIncomplete() = createNotificationChannel(
        id = Notifications.Channels.SIGN_UP_INCOMPLETE,
        name = getString(R.string.app_name),
        description = getString(R.string.sign_up_not_complete))

{% endhighlight %}

This is the function that creates every channel:

{% highlight kotlin %}

@RequiresApi(Build.VERSION_CODES.O)
private fun Context.createNotificationChannel(id: String, name: String, description: String) {
    val importance = NotificationManager.IMPORTANCE_HIGH
    val channel = NotificationChannel(id, name, importance)
    channel.description = description
    channel.setShowBadge(true)
    val nm = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
    nm.createNotificationChannel(channel)
}

{% endhighlight %}

## Worker

This is the `Worker` class that shows the notification:

{% highlight kotlin %}

abstract class AbstractNotificationWorker(private val context: Context, params: WorkerParameters) : Worker(context, params) {

    protected abstract val channel: String

    protected abstract val title: String

    protected abstract val text: String

    protected abstract val targetActivity: Class<out Activity>?

    protected abstract val tag: String

    @SuppressLint("InlinedApi")
    override fun doWork(): Result {
        context.notify(channel, title, text, targetActivity, tag)
        return Result.SUCCESS
    }
}

{% endhighlight %}

`Context` will be supplied by WorkManager itself, and `params`, which are also supplied my WorkManager, can be ignored in case of hardcoded messages, and are used in this article only for Firebase Cloud Messaging.

The extension function `notify()`, used above, needs to be written by yourself. You can use your favourite implementation of the code that creates notifications, but it will probably similar to the extenion functions that create the channels, that are explained above.

For exact description of `Worker` as well as `WorkManager`, please see [Jetpack documentation][jetpack]. This article just describes how to use `WorkManager` **specifically for notifications**, assuming that you already know the basics of [architecture components][architecture-components] of [Android Jetpack][jetpack].

To implement specific `Worker` for every particular notification, just extend the abstract class defined above, like this:

{% highlight kotlin %}

class SignUpReminderWorker(private val context: Context, params: WorkerParameters) :
        AbstractNotificationWorker(context, params) {

    override val channel = Notifications.Channels.SIGN_UP_INCOMPLETE

    ...
	
    override val tag = TAG

    companion object {
        const val TAG = "sign_up_reminder"
        const val DELAY_H = 24L
    }
}

{% endhighlight %}

## Scheduling

This section will describe how to schedule a notification for some time in the future (here: 24 hors), how to test it manually, by reducing the time to just a couple of minutes in debug builds, and how to prevent the notification from showing during the "do not disturb" hours.

To schedule one notification, you call this extention function on your `Fragment`:

{% highlight kotlin %}

@MainThread
fun Fragment.scheduleSignUpReminder() =
        context!!.scheduleReminderOnce<SignUpReminderWorker>(SignUpReminderWorker.TAG, SignUpReminderWorker.DELAY_H)

{% endhighlight %}

This in turn calls this extension function on the `Context`. It expent a type parameter that extents the abstract class `AbstractNotificationWorker` explained above:

{% highlight kotlin %}

inline fun <reified T : AbstractNotificationWorker>Context.scheduleReminderOnce(tag: String, delayInHours: Long) {
    val subscriptionRemindWork = OneTimeWorkRequestBuilder<T>()
            .delay(delayInHours)
	    .addTag(tag)
	    .build()
    WorkManager.getInstance().enqueue(subscriptionRemindWork)
}

{% endhighlight %}

The extention function `delay` used above requires an extra explanation, at it contains the code that decides if the code is run after the required time measured in hours (in production builds), or in minutes (in debug builds), and prevent displaying notifications at nighttime.

{% highlight kotlin %}

fun OneTimeWorkRequest.Builder.delay(delayInHours: Long): OneTimeWorkRequest.Builder {
    val hour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY).toLong()
    val doNotDisturbShift = when {
        hour < DO_NOT_DISTURB_MORNING -> DO_NOT_DISTURB_MORNING - hour
        hour > DO_NOT_DISTURB_EVENING -> DO_NOT_DISTURB_MORNING + HOURS_IN_DAY - hour
        else -> 0
    }
    return if (BuildConfig.DEBUG || BuildConfig.FLAVOR == App.FLAVOR_STAGING) {
        val testDelayInMinutes = delayInHours
        setInitialDelay(
                testDelayInMinutes + doNotDisturbShift * MINUTES_IN_HOUR,
                TimeUnit.MINUTES)
    } else setInitialDelay(delayInHours + doNotDisturbShift, TimeUnit.HOURS)
}

private const val DO_NOT_DISTURB_EVENING = 23L
private const val DO_NOT_DISTURB_MORNING = 7L
private const val HOURS_IN_DAY = 24L
private const val MINUTES_IN_HOUR = 24L

{% endhighlight %}

The above function calculates by how many hours the notification needs to be shifted so that it does not appear between 23:00 and 7:00. The effect of the shift will be that the notification will be then shifted to some time after 7:00, but before 8:00.

It then checks whether this is a debug build, or one of the debug flavors. If so, then instead of waiting several hours for the notification to show up, it will wait the same amoutd of minutes. (But either way it will not show up at nighttime).

## Canceling the scheduled notifications

These couple of functions cancel the scheduled notifications:

{% highlight kotlin %}

fun Fragment.cancelSignUpReminder() = context!!.cancelLoginReminder()

fun Context.cancelSignUpReminder() = cancelReminder(SignUpReminderWorker.TAG)

fun Context.cancelReminder(tag: String) {
    WorkManager.getInstance().cancelAllWorkByTag(tag)
    with(NotificationManagerCompat.from(this)) {
        cancel(tag, Notifications.DEFAULT_NOTIFICATION_ID)
    }
}

{% endhighlight %}

## Firebase Cloud Messaging

This section will explain how to use design patterns already described above in conjunction with [Firebase Cloud Messaging][fcm].

If you want to create a separate notification channel for this, creation of notification channels is also desrbibed in the preceding sections.

To handle [Firebase Cloud Messaging][fcm] you need to set up a service in your `AndroidManifest.xml`:

{% highlight xml %}

<service android:name="io.github.syrop.fcm.SyropFcmService"
         tools:ignore="ExportedService">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>

{% endhighlight %}

Please note that this is the only time you have to set up a `Service` in order to use notifications. All other times notificiations have been handled by [WorkManager][workmanager], and `WorkManager` will be used for handling cloud messaging as well, but this time the notification will be scheduled from a `Service`.

This is the body of the `Service`:

{% highlight kotlin %}

class SyropFcmService : FirebaseMessagingService() {

    override fun onMessageReceived(rm: RemoteMessage) {
        rm.notification?.body?.let { body ->
            scheduleFcmNotification(body)
        }
    }

    override fun onNewToken(token: String) {
        log.info("Refreshed FCM token: $token")
    }
}

{% endhighlight %}

You do not need to use the `Service`'s `Context` to display the notification, as `WorkManager` provides its own `Context`.

This is the function scheduling the notification. It works similar to the way described above:

{% highlight kotlin %}

fun scheduleFcmNotification(text: String) {
    val data = Data.Builder().putAll(mapOf(FcmNotificationWorker.MESSAGE to text)).build()

    val loginRemindWork = OneTimeWorkRequestBuilder<FcmNotificationWorker>()
            .setInputData(data)
	    .setInitialDelay(10L, TimeUnit.SECONDS)
            .build()
    WorkManager.getInstance().enqueue(loginRemindWork)
}

{% endhighlight %}

The difference is that now it needs to set up `InputData`, as now it doesn't use hardcoded notification text. All other notifications described previously in this article used hardcoded text:

{% highlight kotlin %}

class FcmNotificationWorker(private val context: Context, val params: WorkerParameters) :
        AbstractNotificationWorker(context, params) {

    override val channel: String get() = context.getString(R.string.fcm_notification_channel)

    override val title: String = context.getString(R.string.app_name)

    override val text get() = params.inputData.getString(MESSAGE)!!

    override val targetActivity: Class<out Activity>? = null

    override val tag = TAG

    companion object {
        const val MESSAGE = "message"
        const val TAG = "fcm"
    }
}

{% endhighlight %}

The message is retrieved using the code `params.inputData.getString(MESSAGE)!!`. Setting `targetActivity` to null will inform the `AbstractNotificationWorker` that it doesn't have to set any target activity in the message activity it is going to display, but if you set here any oter value, you will be driven to that activity (by a `PendingIntent`) when you tap on the notification.

Because of the way I wrote the code in this section, cloud messages will be displayed with but a 10 seconds delay, regardless of whether it is day or night, but you can just as well use the code explained previously to create an extra delay when cloud messages are received at nighttime.


## Donations

If you've enjoyed this article, consider donating some bitcoin: bc1qncxh5xs6erq6w4qz3a7xl7f50agrgn3w58dsfp (you can also look at [donations page][donations]).

[jetpack]: https://developer.android.com/jetpack/
[github]: https://github.com/syrop/
[fcm]: https://firebase.google.com/docs/cloud-messaging/
[workmanager]: https://developer.android.com/topic/libraries/architecture/workmanager/
[testing]: https://syrop.github.io/jekyll/update/2018/12/25/testing-with-dependency-retrieval.html
[architecture-components]: https://developer.android.com/topic/libraries/architecture/
[donations]: https://syrop.github.io/donate/
