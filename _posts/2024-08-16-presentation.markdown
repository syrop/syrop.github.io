---
layout: post
title:  "Separating Presentation layer from Android (refactoring)"
date:   2024-08-16 9:37:00 +0200
categories: jekyll update
---

In this article I will explain how I separated Presentation layer from Android libraries and dependencies.

## Topics covered

* Clean Architecture
* Testing

## The project

The project I use for the demonstation is [Victor Checkers][checkers] This is the [commit] in question.

## The problem

The problem with the previous solution, which treated the Presentation layer like an Android library rather than a Java library, was that I found the JUnit tests placed in the Android library did not recognize the classes set in a Java module.

## The solution

The solution was to remove Android dependencies from the Presentation layer. This is the solution I found in  Eran Boudjnah's project [Clean Architecture for Android Sample Project][clean].

## The steps

**Step 1**

Just remove Android dependencies from your module's level build.gradle:

```groovy
plugins {
    id 'java-library'
    id 'org.jetbrains.kotlin.jvm'
    id 'kotlin-kapt'
}

java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}

dependencies {
    kapt "com.google.dagger:hilt-compiler:2.52"

    implementation project(':checkers-domain')

    testImplementation 'junit:junit:4.13.2'
    testImplementation "org.mockito.kotlin:mockito-kotlin:5.4.0"
    testImplementation "org.mockito:mockito-inline:5.2.0"
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0-RC.2"
}
```

**Step 2**

Rename ```BaseViewModel``` to ```BasePresentation``` and don't inherit it from ```ViewModel```.

```kotlin
package pl.org.seva.checkers.presentation.architecture

import ...

abstract class BasePresentation<VIEW_STATE : Any, NOTIFICATION : Any>(
    useCaseExecutorProvider: UseCaseExecutorProvider
) { ...
```

**Step 3**

Replace ```androidx.compose.runtime.State``` with ```kotlinx.coroutines.flow.MutableStateFlow```.

```kotlin
var whiteWon = MutableStateFlow(false)
var blackWon = MutableStateFlow(false)
```

**Step 3, ViewModel***

The reason I still use ```VIewModel``` is that I want to have access to ```vievModelScope```. If I ever decide to navigate away from the checkerboard screen, I want the minimax algorihm to automatically cancel generation of the new movements, but I do want it to survive mere screen orientation changes.

This is what I put in the UI layer:

```kotlin
package pl.org.seva.checkers.ui.view

import ...
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import pl.org.seva.checkers.presentation.GamePresentation
import javax.inject.Inject

@HiltViewModel
class GameViewModel @Inject constructor(
    private val presentation: GamePresentation
) : ViewModel() { ...
```

The ```ViewModel``` is small, though, and only maps the methods to corresponding methods in the ```Presentation``` class in the Presentation layer.

**Step 4, Jetpack Compose**

Because I use ```StateFlow``` in the Presentation layer, in the UI layer I convert them again to ```State```.

This is what I do in the ```Fragment```:

```kotlin
Pieces(
    piecesPresentationToUiMapper.toUi(vm.viewState.collectAsState().value.pieces),
    vm.viewState.collectAsState().value.movingWhiteMan,
    vm.viewState.collectAsState().value.movingWhiteKing,
    onTouchListener,
)
```

**Step 5, usecases**

Because I no longer use ```ViewModel``` in the Presentation layer, I had to somehow come up with a way to pass ```viewModelScope``` from the UI layer to the Presentation layer.

This is what I do in ```GameViewModel``` to call a usecase:

```kotlin
fun addWhite(x: Int, y: Int, king: Boolean) = presentation.addWhite(x, y, king, viewModelScope)
```

This is the method in ```GamePresentation```:

```kotlin
fun addWhite(x: Int, y: Int, forceKing: Boolean = false, coroutineScope: CoroutineScope) {
    updateViewState(if (forceKing || y == 0) {
        viewState.value.addWhiteKing(x to y)
    }
    else {
        viewState.value.addWhiteMan(x to y)
    })
    lastMove = LastMove.WHITE
    execute(whiteMoveUseCase, coroutineScope, piecesPresentationToDomainMapper.toDomain(viewState.value.pieces), ::presentPieces)
}
```

And this is the ```execute``` method in ```BasePresentation```:

```kotlin
protected fun <INPUT, OUTPUT> execute(
    useCase: UseCase<INPUT, OUTPUT>,
    coroutineScope: CoroutineScope,
    value: INPUT,
    onSuccess: (OUTPUT, CoroutineScope) -> Unit = { _, _ -> },
    onException: (DomainException) -> Unit = {}
) {
        useCaseExecutor.execute(useCase, coroutineScope, value, onSuccess, onException)
}
```

For reference, this is the full code of ```UseCaseExecutor```:

```kotlin
package pl.org.seva.checkers.domain.cleanarchitecture.usecase

import ...

class UseCaseExecutor {

    fun <INPUT, OUTPUT> execute(
        useCase: UseCase<INPUT, OUTPUT>,
        coroutineScope: CoroutineScope,
        value: INPUT,
        onSuccess: (OUTPUT, CoroutineScope) -> Unit = { _, _ ->},
        onException: (DomainException) -> Unit = {}
    ) {
        coroutineScope.launch {
            try {
                useCase.execute(value, coroutineScope, onSuccess)
            }
            catch (ignore: CancellationException) {
            }
            catch (throwable: Throwable) {
                onException(
                    (throwable as? DomainException)
                        ?: UnknownDomainException(throwable)
                )
            }
        }
    }

}
```

## Summary

The following steps have been used for getting rid of Android dependencies in the Presentation layer:

* Remove the dependencies themselves.
* Create a new ```ViewModel``` class in the UI layer, for the sake of ```viewModelScope```.
* Every time you execute a test case, pass the ```viewModelScope``` in a parameter to the Presentation layer.




[checkers]: https://github.com/syrop/Checkers
[commit]: https://github.com/syrop/Checkers/commit/e919a2b9ece2bd87e786d6cc985c00544f0385bc
[clean]: https://github.com/EranBoudjnah/CleanArchitectureForAndroid

