---
title: 'Making speech bubbles in SVG'
description: TODO
tags:
    - tech
    - svg
series: Scalar Vector Graphics (SVG)
aliases:
  - /tech/making-speech-bubbles-in-svg/
header:
  image: header-low.svg
---
Earlier today I wanted to make some speech bubbles for an SVG image I was working on.
They're not hard to make, but getting them right takes a bit of fiddling.
So I wrote down some examples, for later reference.

{{< note >}}
UPDATE: I made a small [SVG speech bubble generator](/speechbubblegenerator/), based on the method described on this post.
Have fun with it!
{{< /note >}}

{{<figure-with-caption caption="Go can talk to Browsers and Node. The (svg) Go Gopher above is licensed under the [Creative Commons 3.0 Attribution License](https://creativecommons.org/licenses/by/3.0/), and was drawn in SVG by [Renee French](http://reneefrench.blogspot.com/).">}}
  ![Javascript engines and Go Gopher talking](interface.svg)
{{< /figure-with-caption >}}

Here I show how to create speech bubbles.

They consist of three parts:
- An `ellipse` for the black border
- A `path` for the speech "arrow"
- Another `ellipse` to fill in the part of the speech "arrow" that is inside the bubble

This manual assumes that you have experience with SVG.


## Step 1: Make the bubble

We start by making an ellipse that is in the right spot and of the right size.
Make this ellipse in a `defs` section, so that we can refer to it twice.

The first `<use>` creates an ellipse with a stroke width of twice the size that we want (in this example we want a stroke width of 3px, so we set it to 6px for the first ellipse).
Then a second `<use>` creates a second ellipse filled with white (this ellipse will overlap with half the original stroke, so only a width of 3px is left).

{{< svg-with-code >}}
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 210">
    <defs>
        <ellipse id="bubble" cx="300" cy="75" rx="150" ry="70" />
    </defs>

    <use href="#bubble" style="stroke: black; stroke-width: 6px; fill: none;" />
    <use href="#bubble" style="stroke: none; fill: white;" />
</svg>
{{< /svg-with-code>}}



## Step 2: the talk arrow

We know use `<path>` to create a talk arrow.
Since I want the arrow to point to left-bottom, just pick a spot inside the ellipse, in the left-bottom corner (I choose 200, 100).
From this, we draw a line to the left-bottom (-100, 100), and back but slightly to the right (120, 100).
Give this path stroke-width 3.

Note: in the `d=""` section of path, there is a difference between upper and lower case letters: upper case are absolute coordinated, lower case means the coordinates are relative to where you were before. Don't mix them up.

There is *no* difference between space and comma; you will see both used online. Personally I like to put a comma between the x and y coordinates, and a space between different parts of a command (in the next sections we will get to commands with multiple coordinates).

{{< svg-with-code >}}
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 210">
    <defs>
        <ellipse id="bubble" cx="300" cy="75" rx="150" ry="70" />
    </defs>

    <use href="#bubble" style="stroke: black; stroke-width: 6px; fill: none;" />
    <use href="#bubble" style="stroke: none; fill: white;" />
    <path style="stroke: black; stroke-width: 3px; fill: white;" d="
        M 200,100
        l -100,100
        l 120,-100
    " />
</svg>
{{< /svg-with-code >}}

## Optional: 2b
If you like this style, you may wiggle a bit with the coordinates to make it look nicer, else continue to the next step.

{{< svg-with-code >}}
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 210">
    <defs>
        <ellipse id="bubble" cx="300" cy="75" rx="150" ry="70" />
    </defs>

    <use href="#bubble" style="stroke: black; stroke-width: 6px; fill: none;" />
    <use href="#bubble" style="stroke: none; fill: white;" />
    <path style="stroke: black; stroke-width: 3px; fill: white;" d="
        M 200,123
        l -100,80
        l 115,-74
    " />
</svg>
{{< /svg-with-code >}}


## Step 3: Hide the talk-arrow inside the bubble
By having the second ellipse draw *over* the arrow, we can hide the part inside the bubble:

{{< svg-with-code >}}
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 210">
    <defs>
        <ellipse id="bubble" cx="300" cy="75" rx="150" ry="70" />
    </defs>

    <use href="#bubble" style="stroke: black; stroke-width: 6px; fill: none;" />
    <path style="stroke: black; stroke-width: 3px; fill: white;" d="
        M 200,100
        l -100,100
        l 120,-100
    " />
    <use href="#bubble" style="stroke: none; fill: white;" />
</svg>
{{< /svg-with-code >}}


## Step 4: Bend the talk-arrow
Finally we bend the talk-arrow a bit, by replacing the lines (noted with `l` in the path) with curves (`q`).
This is something that has to be done by wiggling a bit, by hand.
Easiest is to first replace `l A,B` with `q A/2,B/2 A,B`.
This should change nothing on the picture, but prepare for bending

{{< svg-with-code >}}
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 210">
    <defs>
        <ellipse id="bubble" cx="300" cy="75" rx="150" ry="70" />
    </defs>

    <use href="#bubble" style="stroke: black; stroke-width: 6px; fill: none;" />
    <path style="stroke: black; stroke-width: 3px; fill: white;" d="
        M 200,100
        q -50,50 -100,100
        q 60,-50 120,-100
    " />
    <use href="#bubble" style="stroke: none; fill: white;" />
</svg>
{{< /svg-with-code >}}

Now, bend the arrow-lines, by changing the *first* pair of coordinates of the `q` section.
The change you make, will move the middle of the line in that direction.
So say that we would like the middle of the line to be pulled towards the bottom right, just add 10 to those values (take care of the signs: -50 + 10 = -40!)

{{< svg-with-code >}}
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 210">
    <defs>
        <ellipse id="bubble" cx="300" cy="75" rx="150" ry="70" />
    </defs>

    <use href="#bubble" style="stroke: black; stroke-width: 6px; fill: none;" />
    <path style="stroke: black; stroke-width: 3px; fill: white;" d="
        M 200,100
        q -40,60 -100,100
        q 70,-40 120,-100
    " />
    <use href="#bubble" style="stroke: none; fill: white;" />
</svg>
{{< /svg-with-code >}}

Or bend even more:

{{< svg-with-code >}}
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 210">
    <defs>
        <ellipse id="bubble" cx="300" cy="75" rx="150" ry="70" />
    </defs>

    <use href="#bubble" style="stroke: black; stroke-width: 6px; fill: none;" />
    <path style="stroke: black; stroke-width: 3px; fill: white;" d="
        M 200,100
        q -20,80 -100,100
        q 90,-20 120,-100
    " />
    <use href="#bubble" style="stroke: none; fill: white;" />
</svg>
{{< /svg-with-code >}}


## Insight: see the layers
In order to see what is actually going on, this last one below has the white fill not fully cover....

{{< svg-with-code >}}
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 210">
    <defs>
        <ellipse id="bubble" cx="300" cy="75" rx="150" ry="70" />
    </defs>

    <use href="#bubble" style="stroke: black; stroke-width: 6px; fill: none;" />
    <path style="stroke: black; stroke-width: 3px; fill: rgba(255, 255, 255, .8);" d="
        M 200,100
        q -20,80 -100,100
        q 90,-20 120,-100
    " />
    <use href="#bubble" style="stroke: none; fill: rgba(255, 255, 255, .8);" />
</svg>
{{< /svg-with-code >}}

# Bonus: shadow

I couldn't resist a small 3D shadow effect in there....
{{< svg-with-code >}}
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 500 210">
    <defs>
        <ellipse id="bubble" cx="300" cy="75" rx="150" ry="70" />
        <filter id="shadow">
            <feOffset result="offOut" in="SourceAlpha" dx="2" dy="3" />
            <feGaussianBlur result="blurOut" in="offOut" stdDeviation="2" />
            <feBlend in="SourceGraphic" in2="blurOut" mode="normal" />
        </filter>
    </defs>

    <g filter="url(#shadow)">
        <use href="#bubble" style="stroke: black; stroke-width: 6px; fill: none;" />
        <path style="stroke: black; stroke-width: 3px; fill: white;" d="
            M 200,100
            q -20,80 -100,100
            q 90,-20 120,-100
        " />
        <use href="#bubble" style="stroke: none; fill: white;" />
    </g>
    <text y="45" style="font-family: 'Courier New', monospace;">
        <tspan x="250">Is that</tspan>
        <tspan x="155" dy="1.2em">a train in your pocket</tspan>
        <tspan dy="1.2em" x="160">or are you just happy</tspan>
        <tspan dy="1.2em" x="240">to see me</tspan>
        </text>
</svg>
{{< /svg-with-code >}}

# Bonus 2: I made a speech bubble generator
Next time you need an SVG bubble, just create one in the [SVG bubble generator](/speechbubblegenerator/).
