---
layout: post
title:  "Testing with dependency injection"
date:   2018-12-25 14:03:19 +0100
categories: jekyll update
---
In this article I will explain how to use dependency injection for testing.

In the article I use code snippets from the project [GPS-Texter][texter]. This application sends a text message each time distance from home location changes by 2 kilometres. In the test run I will use a mock location hardcoded in a test class, and instead of really sending text messages I will just count how many times the application was trying to send them and compare this number with the expected value.

For dependency injection I use [Kodein][kodein-di].

To use Kodein in your project, add the relevant dependencies:

{% highlight groovy %}

implementation 'org.kodein.di:kodein-di-generic-jvm:6.0.1'
implementation 'org.kodein.di:kodein-di-conf-jvm:6.0.1'

{% endhighlight %}

Contrary to the recommendation in [kodein-android documentation][kodein-android], I do not use the Android-specific dependency `implementation 'org.kodein.di:kodein-di-framework-android-x:6.0.1'`.

## Creation of the module

The following snippet shows implementation of the module builder, together with the bindings that are relevant for this article:

{% highlight kotlin %}

class KodeinModuleBuilder(private val ctx: Context) {

    fun build() = Kodein.Module("main") {
	bind<Bootstrap>() with singleton { Bootstrap(ctx) }
	...
        bind<LocationSource>() with singleton { LocationSource() }
        bind<SmsSender>() with singleton { SmsSender() }
	...
        bind<ActivityRecognitionSource>() with singleton { ActivityRecognitionSource() }
        ...
    }
}

{% endhighlight %}


This extention property builds the module:

{% highlight kotlin %}

val Context.module get() = KodeinModuleBuilder(this).build()

{% endhighlight %}

The module needs to be imported in the constructor of `Application` class:

{% highlight kotlin %}

init {
    Kodein.global.addImport(module)
}


{% endhighlight %}

## Retrieval

Create one global extention function that creates instances of required types:

{% highlight kotlin %}

inline fun <reified R : Any> instance(): R {
    val result by Kodein.global.instance<R>()
    return result
}

{% endhighlight %}

This will be demonstrated on the example of `Bootstrap` class that is used in [GPS-Texter][texter] to initialize the application once per run:

{% highlight kotlin %}

val bootstrap: Bootstrap get() = instance()

class Bootstrap(private val ctx: Context) {
    ...
}

{% endhighlight %}

Because of the way bindings are created in the application's Kodein module, `bootstrap` is now a global property that returns a single instance of the `Bootstrap` class that contains an instance of application context.

This is the way it is used for initialization of the application:

{% highlight kotlin %}

open class TexterApplication : Application() {

    init {
        Kodein.global.addImport(module)
    }

    override fun onCreate() {
        super.onCreate()
        bootstrap.boot()
    }

    ...
}

{% endhighlight %}

Please note that this class needs to be `open`, because it needs to be overridden for testing.

## Overridding of the module

This is the implementation of the overriden Kodein module:

{% highlight kotlin %}

val mockModule get() = MockModuleBuilder().build()

class MockModuleBuilder {
    fun build() = Kodein.Module("test") {
        bind<LocationSource>(overrides = true) with singleton { MockLocationSource() }
        bind<SmsSender>(overrides = true) with singleton { MockSmsSender() }
        bind<ActivityRecognitionSource>(overrides = true) with singleton { MockActivityRecognitionSource() }
    }
}

{% endhighlight %}

Please note that at the top there is a definition of the property 'mockModule' that returns the instance of this particular class.

## Mock test runner

In `build.gradle` define the test runner you will be using:

{% highlight groovy %}

defaultConfig {
        ...
        testInstrumentationRunner 'pl.org.seva.texter.MockTestRunner'
	...
    }

{% endhighlight %}

Implement the test runner:

{% highlight kotlin %}

@Suppress("unused")  // Declared in build.gradle
class MockTestRunner : AndroidJUnitRunner() {

    override fun newApplication(
        cl: ClassLoader, className: String, context: Context): Application =
        super.newApplication(cl, MockApplication::class.java.name, context)
}

{% endhighlight %}

Because of this, specifically an instance of `MockApplication` will be created and run.

This is the implementation of `MockApplication`:

{% highlight kotlin %}

class MockApplication : TexterApplication() {

    init {
        Kodein.global.addImport(mockModule, allowOverride = true)
    }

    ...
}

{% endhighlight %}

As you see, an instance of the mock Kodein module will be used. Please note the parameter `allowOverride = true`.

## Example mock class

This is the mock class that fails to call Android's code for sending text messages, but just counts how many times it was called, and sets the `PendingIntent` result to `Activity.RESULT_OK`:

{% highlight kotlin %}

class MockSmsSender : SmsSender() {

    var messagesSent: Int = 0
        private set

    ...

    override fun sendTextMessage(
            text: String,
            sentIntent: PendingIntent,
            deliveredIntent: PendingIntent) {
        messagesSent++
        sentIntent.send(Activity.RESULT_OK)
    }
}

{% endhighlight %}

This way it can be called even on an emulator or a devide that doesn't support texting.

## The JUnit test

The test method just waits several seconds, and asserts that the number of sent messages has been at least equal to the expected value:

{% highlight kotlin %}

@RunWith(AndroidJUnit4::class)
class LocationTest {

    // https://stackoverflow.com/questions/29945087/kotlin-and-new-activitytestrule-the-rule-must-be-public
    @Suppress("unused")
    @get:Rule
    val activityRule = ActivityTestRule(MainActivity::class.java, true, true)

    @Test
    fun testLocation() {
        onView(isRoot()).perform(DelayAction.delay(TENTH_SECOND_MS))
        repeat (DURATION_SEC) {
            onView(isRoot()).perform(DelayAction.delay(SECOND_MS))
        }
        val sender = smsSender as MockSmsSender
        assertTrue(sender.messagesSent >= TestConstants.EXPECTED_MESSAGES_SENT)
    }

    companion object {
        private const val DURATION_SEC = 50
        private const val SECOND_MS = 1000L
        private const val TENTH_SECOND_MS = 100L
    }
}


{% endhighlight %}

Because thus configured application uses the injected mock location simulator and mock text messages sender, the sent messages counter will be incremented so many times by the app itself, and no further steps need to be performed in the JUnit test.

Android Espresso is used herein only for waiting a specified amount of seconds. All other actions, like simulating location changes, and counting the sent messages, are performed by the injected objects inside of the tested application.

[texter]: https://github.com/syrop/GPS-Texter
[kodein-di]: https://kodein.org/di/
[kodein-android]: http://kodein.org/Kodein-DI/?6.0/android

