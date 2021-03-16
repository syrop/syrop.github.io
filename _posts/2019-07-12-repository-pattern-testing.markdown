---
layout: post
title:  "Repository pattern and testing"
date:   2019-07-12 05:01:00 +0200
categories: jekyll update
---

This article shows how to test a repository in a relatively large [project][events].

## Listening to changes

In [another article][liverepository-article] in this blog, in the section 'Observable repository', I described a design pattern I use for different observers to be notified when there has been a change in the repository. I call that pattern `LiveRepository`.

This is the code I was using before I performed a refactoring, which I will discuss shortly. **Please do not use it**:

```kotlin
abstract class LiveRepository {

    private val liveData = MutableLiveData<Unit>()

    private val channel = BroadcastChannel<Unit>(Channel.CONFLATED)

    protected fun notifyDataSetChanged() {
        liveData.postValue(Unit)
        channel.sendBlocking(Unit)
    }

    infix fun vm(vm: ViewModel) = { block: () -> Unit ->
        vm.viewModelScope.launch { this@LiveRepository(block) }
    }

    operator fun plus(owner: LifecycleOwner): HotData<Unit> =
            DefaultHotData(liveData, owner)

    suspend operator fun invoke(block: () -> Unit) {
        with (channel.openSubscription()) {
            if (!isEmpty) receive()
            try {
                while (true) {
                    receive()
                    block()
                }
            } finally { cancel() }
        }
    }
}
```

The code above tries to solve the following problem:

There are two ways in which something may be observed. One is aware of `Lifecycle`, and the other is aware of `CoroutineScope`.

Things observed in `Activity` and `Fragment` should probably use `LiveData`, but beceause `Lifecycle` is not present in `ViewModel`, but `viewModelScope` is, the developer, apart from returning an instance of `LiveData`, should provide a mechanism that is canceled simultaneously with the surrounding `CoroutineScope`.


In the above code, I accommodated for the first case by implementing the operator `plus()` that combines `LiveData` with `LifeCycle`, so that it may be observed by an outside code. The latter case is addressed by the suspend operator `invoke()`, which runs a block of code on each update, as long as the `CoroutineScope` it runs in is not canceled.

The DSL I used in the above code is discussed in a [dedicated article][dsl-article] in this blog. Please note that at present, I **do not recommend** using it.

Instead of the above, I currently suggest the form presented below, and introduced in a specific [commit][commit-two]:

```kotlin
abstract class LiveRepository {

    private val broadcastChannel = BroadcastChannel<Unit>(Channel.CONFLATED)

    protected fun notifyDataSetChanged() {
        broadcastChannel.sendBlocking(Unit)
    }

    fun updatedLiveData(scope: CoroutineScope) =
        updatedLiveData(scope.coroutineContext)

    fun updatedLiveData(context: CoroutineContext = EmptyCoroutineContext) =
            liveData(context) {
                with (broadcastChannel.openSubscription()) {
                    try {
                        while (true) {
                            emit(receive())
                        }
                    }
                    finally {
                        cancel()
                    }
                }
            }
        }

```

The above code takes advantage of my favorite function [`liveData()`][livedata], which wraps `CoroutineScope` handling in `LiveData`.

The function currently is in its alpha state, but I believe it is good to use it. By doing so, the developer takes advantage of code that is probably already tested by thousands of other developers.

By avoiding the use of alpha dependencies, and choosing to use one's own implementations, the developer is forced to use a code that is only tested by themselves, and perhaps a few of the other members of their team, and has to maintain it.

By using the libraries provided by Google, even though they are in alpha stage now, the developer capitalizes of the maintenance performed by the company, and the function itself will probably soon reach a stable release.

The required dependencies for [`liveData()`][livedata] are listed in Google's [documentation][livedata-dependencies].

The only time when I use `updatedLiveData()` with `CoroutineContext` other than the default `EmptyCoroutineContext` is when I don't consume the `LiveData` in `Activity` or `Fragment`, but inside `ViemModel`:

```kotlin
class CommViewModel(private val state: SavedStateHandle) : ViewModel() {

    ...

    val name by lazy { MutableLiveData<String?>() }
    val desc by lazy { MutableLiveData<String?>() }
    val isAdmin by lazy { MutableLiveData<Boolean?>() }

    init {
        comms.updatedLiveData(viewModelScope).observeForever { update() }
        val position = state.get<Int>(COMM_POSITION) ?: -1
        if (position >= 0) {
            comm = comms[position]
        }
    }

    fun withPosition(position: Int) {
        comm = comms[position]
        state.set(COMM_POSITION, position)
    }

    fun update() {
        comm = comms[comm.name]
        reset()
    }

    fun reset() {
        name.value = comm.name
        desc.value = comm.desc
        isAdmin.value = comm.isAdmin
    }

    ...
}

```

In the above code, belonging to a `ViewModel`, I don't expose directly the `LiveData` coming from the `LiveRepository`. Instead, I create a few other instances of LiveData and expose *them*.

The fields `name`, `desc` and `isAdmin` are instances of `LiveData` that are exposed to the `Fragment` that uses them. Because I didn't want the `Fragment` itself be notified about the updates is the repository, I have no access to `Lifecycle`.

What I do have access to is `viewModelScope`, so I use it to create `LiveData` and observe it *forever* in the line:

```kotlin
comms.updatedLiveData(viewModelScope).observeForever { update() }
```

Thanks to that, the `CoroutineScope` wrapped in the `LiveData` will be canceled in due time, which prevents the `ViewModel` from hanging around and being notified when it is no longer alive.


## The test

Testing with coroutines was discussing in another [article][testing-article] in this blog, so I recommend reading it to it in order to understand the present description.

Preparation for tests, like the use of a few of the annotations `@Rule`, `@Before` and `@After`, was explained in the other [article][testing-article], so I won't repeat it here.

This is the testing code:

```kotlin
@ExperimentalCoroutinesApi
@ObsoleteCoroutinesApi
class EventsTest {

    private lateinit var fsReader: FsReader
    private lateinit var fsWriter: FsWriter
    private lateinit var eventsDao: EventsDao

    private val mainThreadSurrogate = newSingleThreadContext("UI thread")

    @get:Rule
    val instantTaskExecutorRule = InstantTaskExecutorRule()

    @Before
    fun setUp() {
        Dispatchers.setMain(mainThreadSurrogate)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
        mainThreadSurrogate.close()
    }

    @Before
    fun mockConstructorParameters() {
        fsReader = mock(FsReader::class.java)
        fsWriter = mock(FsWriter::class.java)
        eventsDao = mock(EventsDao::class.java)
    }

    fun testAdd() = runBlockingTest {
        val events = Events(fsReader, fsWriter, eventsDao)
        val event = Event.creationEvent
        val job = Job()
        @Suppress("UNCHECKED_CAST")
        val observer = mock(Observer::class.java) as Observer<Unit>
        events.updatedLiveData(job).observeForever(observer)
        events.add(event)
        verify(fsWriter).add(event)
        verify(eventsDao).add(event)
        verify(observer).onChanged(Unit)
        job.cancel()
    }
}
```

When I add more tests to the code of the class pasted above, probably all of them are going to inject the same dependencies (`FsReader`, `FsWriter` and `EventDao`) in the constructor of `Events`, so I moved thein initialization to one of the `@Before` functions.

In case specific arrangements need to be done for a particular test, they will be in specific functions annotated as `@Test`, but the creation of the mock itself will probably be common for all tests.

The test verifies that three things happen od addition of an `Event` to the repository:

1. The `Event` is added to Firestore database.
2. The `Event` is added to the DAO.
3. An interested 'Observer' is notified about the change.

As dicsussed above, the test mocks the Firebase cloud and the DAO. Because I am not sure whether the `Observer` is going to be used in other tests, and it is not a required dependency, I decided to mock it in the `@Test` function. Because I do not have a `Lifecycle` available in the test, I start the observation by calling `observeForever()`, which doesn't require it.

## Conclusion

The present article has demonstrated how to add to a project a test using dependency injection and coroutines.

It has also demonstrated how to use one function currently in alpha stage - [`liveData()`][livedata] - to create an instance of `LiveData` that is aware not only of `Lifecycle`, but also of `CoroutineScope`.

It has also shown the progress I made in my understanding of testing with Mockito, since I wrote the first [article][testing-article] about it.

I have used Mockito in two of my projects now: The [first article][testing-article] used the project [Compass] as an example, while the present one is using a more advanced project - [Victor Events][events] - which I still have a great interest in developing, and is probably going to have many more improvements, which may be in time described in this blog.

## Donations

If the reader has enjoyed the article, they may want to donate some bitcoin at the address presented below. Readers may also look at my [donations page][donate].

BTC: bc1qncxh5xs6erq6w4qz3a7xl7f50agrgn3w58dsfp

[events]: https://github.com/syrop/Victor-Events
[liverepository-article]: https://syrop.github.io/jekyll/update/2019/04/27/refreshing-your-data.html
[dsl-article]: https://syrop.github.io/jekyll/update/2019/04/30/repository-and-DSL.html
[commit-two]: https://github.com/syrop/Victor-Events/commit/898a34fffa80131dc6819bb2bd84061c65dc8b07
[livedata]: https://developer.android.com/reference/kotlin/androidx/lifecycle/package-summary#liveData(kotlin.coroutines.CoroutineContext,%20kotlin.Long,%20kotlin.coroutines.SuspendFunction1)
[livedata-dependencies]: https://developer.android.com/topic/libraries/architecture/coroutines
[testing-article]: https://syrop.github.io/jekyll/update/2019/06/21/viewmodel-mockito.html
[compass]: https://github.com/syrop/Compass
[donate]: https://syrop.github.io/donate/

