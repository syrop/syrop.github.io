---
layout: post
title:  "Moving checkers men (functional programming)"
date:   2021-02-23 18:53:00 +0200
categories: jekyll update
---

In this article I explain how to implement moving checkers pieces using only immutable lists.

## Project

The project I use for the demonstation is [Checkers].

## Pending

Only movements of white men are implemented. In future versions, black men are going to be moved by the computer.

Kings are not supported. In memory, position of kings is represented by empty lists.

## Layout

This is the layout file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

<data>
    <variable
        name="viewModel"
        type="pl.org.seva.checkers.game.GameVM"/>
</data>

<RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

    <pl.org.seva.checkers.game.view.Board
        android:id="@+id/board"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

    <pl.org.seva.checkers.game.view.Pieces
        android:id="@+id/pieces"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        gameState="@{viewModel.gameStateFlow}"/>

</RelativeLayout>
</layout>

```

I use two views in my game, separately for the board and for pieces.

Notice the line `gameState="@{viewModel.gameStateFlow}`. It binds the view with the `StateFlow` containing information about positions of the pieces.

## Data class

This is the data class containing positions of men and kings:

```kotlin
data class GameState(
        val whiteMen: List<Pair<Int, Int>>,
        val blackMen: List<Pair<Int, Int>>,
        val whiteKings: List<Pair<Int, Int>>,
        val blackKings: List<Pair<Int, Int>>,
        val moving: Pair<Int, Int> = -1 to -1,
) {
    fun containsWhite(pair: Pair<Int, Int>) = whiteMen.contains(pair) || whiteKings.contains(pair)

    fun containsBlack(pair: Pair<Int, Int>) = blackMen.contains(pair) || whiteKings.contains(pair)

    fun removeWhite(pair: Pair<Int, Int>) = GameState(
            whiteMen.filter { it != pair },
            blackMen,
            whiteKings.filter { it != pair },
            blackKings,
    )

    fun removeBlack(pair: Pair<Int, Int>) = GameState(
            whiteMen,
            blackMen.filter { it != pair },
            whiteKings,
            blackKings.filter { it != pair },
    )

    fun addWhiteMan(pair: Pair<Int, Int>) = GameState(
            whiteMen + pair,
            blackMen,
            whiteKings,
            blackKings,
    )
}
```

It contains a few functions creating a new state of the game. Please analyze the above code to see how to create an immutable list containing one element less, or one element more, than the previous version.

Future versions of this class will contain code generating and analysing possible movements made by the computer.

## The fragment

This is the code of the fragment:

```kotlin
class GameFragment : Fragment(R.layout.fr_game) {

    private lateinit var binding: FrGameBinding
    private val vm by viewModels<GameVM>()

    var isInMovement = false
    var pickedFrom = -1 to -1

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?,
    ): View {
        binding = FrGameBinding.inflate(inflater, container, false)
        binding.lifecycleOwner = this
        binding.viewModel = vm
        return binding.root
    }

    @SuppressLint("ClickableViewAccessibility")
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        fun predecessor(x1: Int, y1: Int, x2: Int, y2: Int): Pair<Int, Int> {
            if (abs(x2 - x1) != abs(y2 - y1) ||
                    abs(x2 - x1) < 2 || abs(y2 - y1) < 2) return -1 to -1
            val dirx = if (x2 - x1 > 0) 1 else -1
            val diry = if (y2 - y1 > 0) 1 else -1
            return x2 - dirx to y2 - diry
        }

        binding.pieces.setOnTouchListener { _, event ->
            when (event.actionMasked) {
                MotionEvent.ACTION_DOWN -> if (vm.isWhiteMoving) {
                    val x = binding.pieces.getX(event.rawX)
                    val y = binding.pieces.getY(event.rawY)
                    pickedFrom = x to y
                    isInMovement = vm.removeWhite(x, y)
                }
                MotionEvent.ACTION_MOVE -> if (isInMovement) {
                    vm.moveTo(event.rawX.toInt(), event.rawY.toInt())
                }
                MotionEvent.ACTION_UP -> if (isInMovement) {
                    val x = binding.pieces.getX(event.rawX)
                    val y = binding.pieces.getY(event.rawY)
                    if (x in 0..7 && y in 0..7 && vm.isEmpty(x, y) &&
                            abs(x - pickedFrom.first) == 1 &&
                            (y == pickedFrom.second - 1 ||
                                    y == pickedFrom.second - 2 &&
                                    vm.removeBlack(predecessor(pickedFrom.first, pickedFrom.second, x, y)))) {
                        vm.addWhite(x, y)
                    }
                    else {
                        vm.addWhite(pickedFrom.first, pickedFrom.second)
                    }
                }
            }
            true
        }
    }
}
```

Please analyze the touch listeners. It performs some validation and then it manipulates the state of the game.

On `MotionEvent.ACTION_DOWN` it first calculates the coordinates on the board from the coordinates of the event. This is the code in the `View` that performs the conversion:

```kotlin
fun getX(x: Float) = x.toInt() * 8 / measuredWidth

fun getY(y: Float) = y.toInt() * 8 / measuredHeight - 1
```

The listener then remembers the original position of the man and removes it from the board.

In the data class the man currently in movement is remembered in the field:

```kotlin
val moving: Pair<Int, Int> = -1 to -1
```

This is the field that is used for animating the movement of the man currently held by the user.

On `MotionEvent.ACTION_MOVE` the listener moves (animates) one man.

On `MotionEvent.ACTION_UP' the listener performs validation of the movement and does some of the following:

- Captures (removes from memory) a black piece
- Stores in memory the new position of the white man
- Puts the white man back in its original position if the movement is invalid

## The ViewModel

This is the ViewModel:

```kotlin
class GameVM : ViewModel() {

    var isWhiteMoving = true

    private var gameState = GameState(WHITE_START_POSITION, BLACK_START_POSITION, emptyList(), emptyList())

    private val _gameStateFlow = MutableStateFlow(gameState)
    val gameStateFlow: StateFlow<GameState> = _gameStateFlow

    fun removeWhite(x: Int, y: Int): Boolean {
        val removed = gameState.removeWhite(x to y)
        val result = removed != gameState
        _gameStateFlow.value = removed
        gameState = removed
        return result
    }

    fun addWhite(x: Int, y: Int) {
        gameState = gameState.addWhiteMan(x to y)
        _gameStateFlow.value = gameState
    }

    fun moveTo(x: Int, y: Int) {
        gameState = gameState.copy(moving = x to y)
        _gameStateFlow.value = gameState
    }

    private fun containsWhite(x: Int, y: Int) = gameState.containsWhite(x to y)

    private fun containsBlack(x: Int, y: Int) = gameState.containsBlack(x to y)

    fun removeBlack(pair: Pair<Int, Int>): Boolean {
        val removed = gameState.removeBlack(pair)
        val result = gameState == removed
        gameState = removed
        return result
    }

    fun isEmpty(x: Int, y: Int) = !containsWhite(x, y) && !containsBlack(x, y)

    companion object {
        val WHITE_START_POSITION = listOf(0 to 7, 1 to 6, 2 to 7, 3 to 6, 4 to 7, 5 to 6, 6 to 7, 7 to 6, 0 to 5, 2 to 5, 4 to 5, 6 to 5)
        val BLACK_START_POSITION = listOf(0 to 1, 1 to 0, 2 to 1, 3 to 0, 4 to 1, 5 to 0, 6 to 1, 7 to 0, 1 to 2, 3 to 2, 5 to 2, 7 to 2)
    }
}
```

Removing a white piece (man or king) in the function `removeWhite()` first creates a version of the game state with one of whie pieces removed. Then it compares the two states to determin whether there has been a change.

If the state is not changed, it means that there was not a white piece in the field in the first place, so movement will not be initiated.

Either way, it sets the new state as a new value of the `StateFlow`. Nothing happens if the value is the same. If the value has been changed, the new state is then passed to the view and reflected on the GUI.

Removing a black piece in the function `removeBlack()` works in a similar way.

The difference is that in the current version of the game a black piece is removed from memory (captured) only while a white man is moving. There is therefore no need to flush to the screen the intermediate state. The final state is flushed to the screen only when the white man finishes its movement.

Storing a new position of the white man in the function `addWhiteMan()` is performed when the white man is finished moving. The new state of the game is then flushed to the screen using the line `_gameStateFlow.value = gameState`.

Finishing the movement of the white man means also that the movement is no longer animated. Under the hood, adding one man to the board in its new position creates a new game state by calling the code:

```kotlin
fun addWhiteMan(pair: Pair<Int, Int>) = GameState(
        whiteMen + pair,
        blackMen,
        whiteKings,
        blackKings,
)
```

It calls the constructor of `GameState` with the default value (`-1 to -1`) of the field `moving`:

```kotlin
data class GameState(
        val whiteMen: List<Pair<Int, Int>>,
        val blackMen: List<Pair<Int, Int>>,
        val whiteKings: List<Pair<Int, Int>>,
        val blackKings: List<Pair<Int, Int>>,
        val moving: Pair<Int, Int> = -1 to -1,
)
```

Negative values mean that no piece is currently moving.

## Conclusion

The article explained how to move checkers pieces in two dimension using a handful of immutable lists.

Creating new versions of the lists have been demonstrated using filtering and concatenation.

Animation of the piece currently being moved has been briefly discussed.

Ground has been prepared for implementation of computer's movement by creating a tree of possible moves represented by immutable game states.


[checkers]: https://github.com/syrop/Checkers

