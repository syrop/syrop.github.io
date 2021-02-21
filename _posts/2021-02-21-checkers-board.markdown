---
layout: post
title:  "Drawing a checkers board (data binding and StateFlow)"
date:   2021-02-21 18:53:00 +0200
categories: jekyll update
---

In this article I explain how to draw men on a checkers board.

## Project

The project I use for the demonstation is [Checkers].

## Credit

In this article I rely heavily on the examples given in the article [StateFlow with One- and TwoWay-DataBinding on Android][article] by [Anastasia Finogenova][anastasia].

## New versions

According to [doco], ```StateFlow``` can be only combined with data binding starting from Gradle plugin **7.0.0-alpha04**.

I will describe now my particular setup of Android Studio Arctic Fox \| 2020.3.1 Canary 7.

Here is the relevant section of project-level **build.gradle**:

```groovy
dependencies {
        classpath 'com.android.tools.build:gradle:7.0.0-alpha07'
        classpath 'org.jetbrains.kotlin:kotlin-gradle-plugin:1.4.21'
}
```

It is combined with a new Gradle version in **gradle-wrapper.properties**:

```
distributionUrl=https\://services.gradle.org/distributions/gradle-6.8.1-all.zip
```

In Project Structure I use the following JDK path:

```
/usr/lib/jvm/java-11-openjdk-amd64
```

## Layout file

This is the layout file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:custom="http://schemas.android.com/apk/res-auto"
    >

<data>
    <variable
        name="viewModel"
        type="pl.org.seva.checkers.game.GameVM"
        />
</data>

<RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

    <pl.org.seva.checkers.game.view.Board
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

    <pl.org.seva.checkers.game.view.Pieces
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        data="@{viewModel.dataFlow}"/>

</RelativeLayout>
</layout>
```

It draws the board and the pieces separately.

Please note the line ```data="@{viewModel.dataFlow}"```. This is what combines ```StateFlow``` with the view.

## The fragment

 ```LifecycleOwner``` must be set to the binding. Notice the line ```binding.lifecycleOwner = this``` in the code below:

```kotlin
class GameFragment : Fragment(R.layout.fr_game) {

    private lateinit var binding: FrGameBinding
    private val viewModel by viewModels<GameVM>()

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        binding = FrGameBinding.inflate(inflater, container, false)
        binding.lifecycleOwner = this
        binding.viewModel = viewModel
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    }
}
```

## The board

Drawing the board is straightforward and will not be explained here:

```kotlin
class Board(context: Context, attrs: AttributeSet) : View(context, attrs) {

    private val fill = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.BLACK
        style = Paint.Style.FILL
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        with(canvas) {
            val dx = right / 8f
            val dy = bottom / 8f
            repeat(8) { x ->
                repeat(8) { y ->
                    if (x % 2 != y % 2) {
                        drawRect(x * dx, y * dy, (x + 1) * dx, (y + 1) * dy, fill)
                    }
                }
            }
        }
    }
}
```

## The ViewModel


This is the ViewModel:

```kotlin
class GameVM : ViewModel() {

    private val _dataFlow = MutableStateFlow(GameData(whiteStartPosition, blackStartPosition))
    val dataFlow: StateFlow<GameData> = _dataFlow

    companion object {
        val whiteStartPosition = arrayListOf(0 to 7, 1 to 6, 2 to 7, 3 to 6, 4 to 7, 5 to 6, 6 to 7, 7 to 6, 0 to 5, 2 to 5, 4 to 5, 6 to 5)
        val blackStartPosition = arrayListOf(0 to 1, 1 to 0, 2 to 1, 3 to 0, 4 to 1, 5 to 0, 6 to 1, 7 to 0, 1 to 2, 3 to 2, 5 to 2, 7 to 2)
    }
}
```

This is the data class:

```kotlin
data class GameData(
        val whiteMen: ArrayList<Pair<Int, Int>>,
        val blackMen: ArrayList<Pair<Int, Int>>,
)
```

The ViewModel contains coordinates of white and black men, wrapped in a ```StateFlow```.

## The BindingAdapter

This is the ```BindingAdapter```. It can be placed in any file, but it must be a global function:

```kotlin
@BindingAdapter("data")
fun setData(pieces: Pieces, data: GameData) {
    pieces.setGameData(data)
}
```

It works with ```StateFlow``` out of the box.

## The pieces

This is the view of the pieces. The current version doesn't support kings:

```kotlin
class Pieces(context: Context, attrs: AttributeSet) : View(context, attrs) {

    private var whites = arrayListOf<Pair<Int, Int>>()

    private var blacks = arrayListOf<Pair<Int, Int>>()

    private val whiteFill = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.GREEN
        style = Paint.Style.FILL
    }

    private val blackFill = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.RED
        style = Paint.Style.FILL
    }

    fun setGameData(data: GameData) {
        whites = data.whiteMen
        blacks = data.blackMen
        invalidate()
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)


        with (canvas) {
            val dx = right / 8f
            val dy = bottom / 8f
            val radius = min(dx, dy) / 2 * 0.85f
            for (piece in whites) {
                drawCircle(piece.first * dx + dx / 2, piece.second * dy + dy / 2, radius, whiteFill)
            }
            for (piece in blacks) {
                drawCircle(piece.first * dx + dx / 2, piece.second * dy + dy / 2, radius, blackFill)
            }
        }
    }
}
```

The function ```setGameData(data: GameData)``` is important, as this is what is called when the value of ```StateFlow``` changes.

Please don't forget to call ```invalidate()```.

## Example coroutine

This is the example coroutine you might use to test the behavior of the checkers board. (It is not on GitHub):

```kotlin
class GameVM : ViewModel() {

    private val _dataFlow = MutableStateFlow(GameData(whiteStartPosition, blackStartPosition))
    val dataFlow: StateFlow<GameData> = _dataFlow

    init {
        viewModelScope.launch {
            delay(1000L)
            _dataFlow.value = GameData(
                    arrayListOf(3 to 3, 3 to 4, 4 to 3, 4 to 4),
                    ArrayList(),
            )
        }
    }

    companion object {
        val whiteStartPosition = arrayListOf(0 to 7, 1 to 6, 2 to 7, 3 to 6, 4 to 7, 5 to 6, 6 to 7, 7 to 6, 0 to 5, 2 to 5, 4 to 5, 6 to 5)
        val blackStartPosition = arrayListOf(0 to 1, 1 to 0, 2 to 1, 3 to 0, 4 to 1, 5 to 0, 6 to 1, 7 to 0, 1 to 2, 3 to 2, 5 to 2, 7 to 2)
    }
}
```

The above code first draws a checkers board with men in their usual positions, and after one second just four white men in the middle.

## Conclusion

The article demonstrates how to draw a checkers board with men using data binding and ```StateFlow``` in the new version of Android Studio.

Step by step it introduces the layout file, ViewModel and two custom views, only one of which reacts to changes in the ```StateFlow```.

At the end it demonstrates how to write a coroutine that, after a delay, moves the men to a new (though invalid) position.

[checkers]: https://github.com/syrop/Checkers
[article]: https://proandroiddev.com/stateflow-with-one-and-twoway-databinding-on-android-cf4e6c847988
[anastasia]: https://medium.com/@itcreativetechnology
[doco]: https://developer.android.com/topic/libraries/data-binding/observability#lifecycle-objects
