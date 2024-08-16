---
layout: post
title:  "Separating Presentation layer from Android (refactoring)"
date:   2024-08-16 9:37:00 +0200
categories: jekyll update
---

In this article I will explain how to implement a slideshow with continuously updated progress bar.

## Topics covered

* Implementing your own View with dynamic colors.
* Flow.
* Jetpack compose.

## The project

The project I use for the demonstation is [Slideshow]. It is a copy of the prototype project [MyApplication], so it may contain some redundant code and dependencies, which you may want to remove for the sake of [YAGNI].

## The images

The images come from a Wikipedia article about [Michael Jackson][michael].  Their respective URLs are hardcoded.

## Segmented progress bar

The segmented progress bar View draws three rounded rectangles that corelate with the three images of Michael Jackson.

The problem is that while the subsequent segments are rounded, the edge indicating the continuous progress is straight. To achieve this effect I use ```clipRect```:

```kotlin
@SuppressLint("DrawAllocation")
override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)
    val segmentSize = (width / numberOfSegments)
    fun drawSegment(id: Int, paint: Paint) {
        canvas.drawRoundRect(
            3F + id * segmentSize,
            0F,
            (id + 1) * segmentSize - 3F,
            height.toFloat(),
            40F,
            40F,
            paint,
        )
    }

    for (i in 0 until  numberOfSegments) {
        drawSegment(i, if (i < progress) progressedPaint else backgroundPaint)
    }
    if (fraction != null) {
        canvas.clipRect(
            Rect(
                (3F + progress * segmentSize + fraction!! * segmentSize).roundToInt(),
                0,
                (id + 1) * segmentSize - 3,
                height,
            )
        )
    }
    drawSegment(progress, progressedPaint)
}
```

The colors of the progress bar are taken from your wallpaper, if you are using Android 12 or above. Using of dynamics colors is explained in my [other article][colors]:

```kotlin
private val backgroundPaint = Paint(Paint.ANTI_ALIAS_FLAG)
private val progressedPaint = Paint(Paint.ANTI_ALIAS_FLAG)
...
init {
    backgroundPaint.color = resources.getColor(com.google.android.material.R.color.material_dynamic_primary20, null)
    backgroundPaint.style = Paint.Style.FILL
    progressedPaint.color = resources.getColor(com.google.android.material.R.color.material_dynamic_primary80, null)
	progressedPaint.style = Paint.Style.FILL
}
```

## Jetpack Compose

This is how I place the view inside Jetpack Compose:

```kotlin
val timer = ...
Column {
    repeat(slides.size) {
        val slide = slides[it]
        AnimatedVisibility(visible = timer == it) {
            val fraction = ...
            Box(modifier = Modifier.padding(16.dp)) {
                Image(
                    painter = rememberCoilPainter(
                        request = slide,
                    ),
                    contentDescription = null,
                    contentScale = ContentScale.FillWidth,
                    modifier = Modifier.fillMaxWidth(),
                )
                key(fraction) {
                    AndroidView(
                        factory = { context ->
                            SegmentedProgressbar(
                                context,
                                null,
                            ).apply {
                                this.fraction = fraction
                                numberOfSegments = slides.size
                                progress = it
                            }
                        },
                        modifier = Modifier
                            .padding(bottom = 24.dp)
                            .align(Alignment.BottomCenter)
                            .width(120.dp)
                            .height(6.dp)
                    )
                }
            }
        }
    }
}
```

For creation of the `SegmentedProgressBar` instance I could use the [builder pattern][builder], but using the correct design patterns is not within the scope of this article. Furthermore, I save some allocations by using `apply` instead, and this code is being executed several times per second.

The ```key``` function above forces redrawing of the SegmentedProgressBar whenever ```fraction``` changes. Without this, the progress bar would be updated only once every five seconds when ```timer``` updates, and this is not what I want to demonstrate.

I use ```Column``` at the top to limit the height of the contained ```Box```. Without ```Column```, the ```Box``` would occupy all available space (the whole screen), but I want the progress bar to be displayed on the top of the pictures. Using of ```Column``` here limits the height of the included content.

## Animation

This is the animation:

```kotlin
private val animation =  TargetBasedAnimation(tween(5000, easing = LinearEasing), Int.VectorConverter, 0, 1000)
```

The main parts worth noticing are ```tween``` and ```LinearEasing```. This assures that the animation will be performed in a linear way. 

Because the desired animation is linear, I could just increment the progress by a constant value every so many milliseconds, but I want this to be more universal. You can experiment with replacing ```tween``` and ```LinearEasing``` with other options.

## Coroutines

These are the coroutines that measure time:

```kotlin
val timer = remember {
    flow {
        while (true) {
            repeat(slides.size) {
                emit(it)
                delay(5_000L)
            }
        }
    }
}.collectAsState(initial = 0).value
...
val fraction = remember(it) {
    flow {
        val beginning = System.nanoTime()
        while (true) {
            emit(
                animation.getValueFromNanos(System.nanoTime() - beginning)
                    .toFloat() / 1000f
            )
            delay(10L)
        }
    }
}.collectAsState(initial = 0f).value
```

## Conclusion

The article shows how to achieve the effect of continuously updated segmented progress bar displayed at the top of a slideshow.

Assumpion is made that the reader is able to download the code from [GitHub][slideshow], and that they understand the basics of View, coroutines and Jetpack Compose.

The code could be further improved by using the [builder] pattern, and by avoiding the use of [magic numbers][numbers], but it is not the point of this article.

Perhaps I could use a custom ```@Composable``` instead of a ```View```, but I haven't been able to find documentation on that.

[slideshow]: https://github.com/syrop/Slideshow
[myapplication]: https://github.com/syrop/MyApplication
[yagni]: https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it
[michael]: https://en.wikipedia.org/wiki/Michael_Jackson
[colors]: https://syrop.github.io/jekyll/update/2021/02/23/moving-checkers-men.html
[builder]: https://en.wikipedia.org/wiki/Builder_pattern
[numbers]: https://en.wikipedia.org/wiki/Magic_number_(programming)

