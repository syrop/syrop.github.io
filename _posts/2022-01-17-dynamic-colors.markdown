---
layout: post
title:  "Checkers and dynamic colors (refactoring)"
date:   2022-01-17 19:28:00 +0200
categories: jekyll update
---

In this article I explain how to refactor a product to add dynamic colors introduced in Android 12.

## The project

The project I use for the demonstation is [Checkers].

It is available on [Google Play][checkers_play].

## The commit

The [commit] discussed in this article.

## build.gradle

Change `targetSdkVersion` and `compileSdkVersion` to 31:

```groovy
android {
    compileSdkVersion 31
    buildToolsVersion '30.0.3'

    defaultConfig {
        applicationId 'pl.org.seva.checkers'
        minSdkVersion 26
        targetSdkVersion 31
        versionCode 4
        versionName '1.1.0'
        testInstrumentationRunner 'androidx.test.runner.AndroidJUnitRunner'
    }
    ...
}
```

Change the version of your Material dependency:

```groovy
implementation 'com.google.android.material:material:1.6.0-alpha01'
```

## manifest.xml

Add `android:exported="true"` to your activity:

```xml
<activity
    android:name="pl.org.seva.checkers.main.MainActivity"
    android:launchMode="singleTop"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
```

## Application

Add `DynamicColors.applyToActivitiesIfAvailable(this)` to your `Application`:

```kotlin
class CheckersApplication : Application() {

    override fun onCreate() {
        super.onCreate()
        DynamicColors.applyToActivitiesIfAvailable(this)
    }
}
```

## styles.xml

In `styles.xml` change the parent theme to `Theme.Material3.DynamicColors.Light`. Remove the colors, so that colors are generated dynamically:

```xml
<!-- Base application theme. -->
<style name="AppTheme" parent="Theme.Material3.DynamicColors.Light">
</style>
```

## Men

Finally, to have the men in different shades of the primary code, remove the `Paint` from the `companion object` and move it to the top of the class:

```kotlin
package pl.org.seva.checkers.game.view

import android.content.Context
import android.graphics.Canvas
import android.graphics.Color
import android.graphics.Paint
import android.util.AttributeSet
import android.view.View
import pl.org.seva.checkers.R
import pl.org.seva.checkers.game.GameState
import java.lang.Float.min

class Pieces(context: Context, attrs: AttributeSet) : View(context, attrs) {

    private var gameState = GameState(emptyList(), emptyList(), emptyList(), emptyList())

    private val whiteFill = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = resources.getColor(R.color.material_dynamic_primary80, null)
        style = Paint.Style.FILL
    }

    private val blackFill = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = resources.getColor(R.color.material_dynamic_primary40, null)
        style = Paint.Style.FILL
    }

    fun setGameState(state: GameState) {
        gameState = state
        invalidate()
    }

    fun getX(x: Float) = x.toInt() * 8 / measuredWidth

    fun getY(y: Float) = y.toInt() * 8 / measuredHeight - 1

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        with (canvas) {
            val dx = right / 8f
            val dy = bottom / 8f
            val radius = min(dx, dy) / 2 * 0.85f
            gameState.whiteMen.forEach { whiteMan ->
                drawCircle(whiteMan.first * dx + dx / 2, whiteMan.second * dy + dy / 2, radius, whiteFill)
            }
            gameState.blackMen.forEach { blackMan ->
                drawCircle(blackMan.first * dx + dx / 2, blackMan.second * dy + dy / 2, radius, blackFill)
            }
            gameState.whiteKings.forEach { whiteKing ->
                drawCircle(whiteKing.first * dx + dx / 2, whiteKing.second * dy + dy / 2, radius, whiteFill)
                drawCircle(whiteKing.first * dx + dx / 2, whiteKing.second * dy + dy / 2, radius / 2, CROWN)
            }
            gameState.blackKings.forEach { blackKing ->
                drawCircle(blackKing.first * dx + dx / 2, blackKing.second * dy + dy / 2, radius, blackFill)
                drawCircle(blackKing.first * dx + dx / 2, blackKing.second * dy + dy / 2, radius / 2, CROWN)
            }
            if (gameState.movingWhiteMan != -1 to -1) {
                drawCircle(gameState.movingWhiteMan.first.toFloat(), gameState.movingWhiteMan.second.toFloat() - dy, radius, whiteFill)
            }
            if (gameState.movingWhiteKing != -1 to -1) {
                drawCircle(gameState.movingWhiteKing.first.toFloat(), gameState.movingWhiteKing.second.toFloat() - dy, radius, whiteFill)
                drawCircle(gameState.movingWhiteKing.first.toFloat(), gameState.movingWhiteKing.second.toFloat() - dy, radius / 2, CROWN)
            }
        }
    }

    companion object {

        private val CROWN = Paint(Paint.ANTI_ALIAS_FLAG).apply {
            color = Color.WHITE
            style = Paint.Style.FILL
        }
    }
}
```

## Conclusion

The article demonstrates how to refactor the code to migrate from a static Material theme to Material 3 and dynamic colors.

It refers to a particular [commit] where the changes were introduced.

[checkers]: https://github.com/syrop/Checkers
[checkers_play]: https://play.google.com/store/apps/details?id=pl.org.seva.checkers
[commit]: https://github.com/syrop/Checkers/commit/a666d551296e006687a1b71fce8f5293e189fa47
