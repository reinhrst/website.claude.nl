---
title: Using KaTeX on github pages
description: Howto on using KaTex for math on your github pages page
tags:
  - tech
aliases:
  - /tech/github pages/using-katex-on-github-pages/
header:
  image: header.jpg
---

Every now and then I write a blog post with some math in it.
Math is hard enough as it is, without the added frustration of having to do a lot of effort (as blog writer) to make it look nice, or a lot of effort (as reader) to decode some math formula written in text, such as Newton's law of gravity: `F = G * (m_1*m_2)/ r^2`.
Luckily there are tools such as [{{<katex inline keepfont>}}{\KaTeX}{{</katex>}}](https://katex.org/) that make math look good: {{<katex inline>}}F = G \times \frac{m_1 \times m_2}{r^2}{{</katex>}}
This post explains how to get it working on your github pages (or any html page) through the use of javascript and client-side rendering.

{{< note >}}
**TL;DR** See [this section](#bringing-it-all-together) for the code that you need to add to the top of your page, and the [example section](#examples) for examples.
{{< /note >}}

{{< note type=info >}}
There are other math rendering engines out there as well; during my search I also came by [MathJax](https://www.mathjax.org).
Supposedly MathJax is supported out of the box in github pages, but I couldn't find a clear description on how to do it.
Since I prefer KaTeX's way of doing things (rending everything to HTML rather than to a png image), I quickly decided to go for that.
{{< /note >}}

{{< note type=info >}}
This blog used to be hosted on standard github pages (jekyll). These days it's a hugo blog, meaning I don't use the tools described in this post anymore, and the math in this post in formatted using similar, but different, methods.

If you want the same functionality in Hugo, contact me :).
{{< /note >}}

There is a [{{<katex inline keepfont>}}\KaTeX{{</katex>}} jekyll plugin](https://github.com/linjer/jekyll-katex), however since github pages doesn’t allow custom plugins, this will not work.
The solution is to render the {{<katex inline keepfont>}}\KaTeX{{</katex>}} expressions on the client, through the use of javascript.
Advantage is that this will work not just on github pages, but on any html page.

Since I like to be explicit, I set the system up in such a way that it looks for "html-like" tags `<katex-inline>...</katex-inline>` (or `<katex-block>...</katex-block>`), and anything between these tags will be rendered by {{<katex inline keepfont>}}\KaTeX{{</katex>}}.
I will introduce 2 methods to  do this; one based on [Web Components](https://developer.mozilla.org/en-US/docs/Web/Web_Components), and the other based on a one time render.
We will then use the second as a fallback method for the first, since [not all webbrowsers support web components](https://caniuse.com/custom-elementsv1) yet.


### Web Components
Using Web Components, one can make custom "html-like"-tags.
It is (in its simplest form) a way to tell the browser "hey, if you encounter a tag with this name, please execute this code in order to show it".
Web Components have pretty decent browser support by now (95% of global users; Internet Explorer being the notable exception...); this may or may not be good enough for you (if you have a github pages page, you may not expect your content to be interesting to IE users anyways :)).
The great advantage of Web Components over other methods is that the system tracks exactly what is going on: it will run your code once for every element it encounters. If elements are added/removed, the system will take care of it all.

Creating a web component is super easy:

```javascript
class KatexInline extends HTMLElement {
    constructor() {
        super();
        katex.render(this.innerText, this, {throwOnError: false, displayMode: false});
    }
}
customElements.define("katex-inline", KatexInline)
```
We create a `KatexInline` class that extends from `HTMLSpanElement`.
In the constructor, we call `katex.render` based on the current contents, and use the current element as target.
Finally, we call `customElements.define` to register our new custom element.

Note: the first version of this blog actually inherited KatexInline from HTMLSpanElement.
This works fine on Firefox, however gives an error on Safari: `TypeError Illegal Constructor` on the line where `super()` is called.
The solution was to extend from `HTMLElement` rather than `HTMLSpanElement`.
The current version of this acticle has the fixes in it.
{: .notice--warning}

We do the same for `KatexBlock`; only change being that we call `katex.render` with `displayMode: true`, which is the display mode for blocks of {{<katex inline keepfont>}}\KaTeX{{</katex>}}.

```javascript
class KatexBlock extends HTMLElement {
    constructor() {
        super();
        katex.render(this.innerText, this, {throwOnError: false, displayMode: true});
    }
}
customElements.define("katex-block", KatexBlock)
```

### One time render
The fallback method is quite easy; we just search for all `<katex-inline>` tags and render them.
Obviously this will only pick up those tags that exist at the time of render, which is probably good enough.
```javascript
document.querySelectorAll("katex-inline").forEach(
    (el) => {
        katex.render(el.innerText, el, {throwOnError: false, displayMode: false});
    }
)
document.querySelectorAll("katex-block").forEach(
    (el) => {
        katex.render(el.innerText, el, {throwOnError: false, displayMode: true});
    }
)
```

## Bringing it all together
Before we can use either method, we need to make sure the {{<katex inline keepfont>}}\KaTeX{{</katex>}} library is loaded.
We achieve this by putting the code in a callback for the onload of the library.
The full code therefore to be added to the header of your page is (if you want, you can put the `renderKatex()` function in a separate file; probably good practice, but not essential for this tutorial):
```javascript
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.13.11/dist/katex.min.css" integrity="sha384-Um5gpz1odJg5Z4HAmzPtgZKdTBHZdw8S29IecapCSB31ligYPhHQZMIlWLYQGVoc" crossorigin="anonymous">

<script>
    function renderKatex() {
        let macros = {}
        if (customElements) {
            class KatexInline extends HTMLElement {
                constructor() {
                    super();
                    katex.render(this.innerText, this, {throwOnError: false, displayMode: false, macros: macros, output: "html"});
                }
            }
            customElements.define("katex-inline", KatexInline)

            class KatexBlock extends HTMLElement {
                constructor() {
                    super();
                    katex.render(this.innerText, this, {throwOnError: false, displayMode: true, macros: macros, output: "html"});
                }
            }
            customElements.define("katex-block", KatexBlock)
        } else {
            document.querySelectorAll("katex-inline").forEach(
                (el) => {
                    katex.render(el.innerText, el, {throwOnError: false, displayMode: false, macros: macros, output: "html"});
                }
            )
            document.querySelectorAll("katex-block").forEach(
                (el) => {
                    katex.render(el.innerText, el, {throwOnError: false, displayMode: true, macros: macros, output: "html"});
                }
            )
        }
    }
</script>
<!-- The loading of KaTeX is deferred to speed up page rendering -->
<script defer src="https://cdn.jsdelivr.net/npm/katex@0.13.11/dist/katex.min.js" integrity="sha384-YNHdsYkH6gMx9y3mRkmcJ2mFUjTd0qNQQvY9VYZgQd7DcN7env35GzlmFaZ23JGp" crossorigin="anonymous" onload="renderKatex()"></script>
```

### Fonts

{{< note >}}
This section was written when I still had a sans-serif as my main blog font.
Leaving it here, because it's still interesting, but some of the information may not make sense anymore.
{{< /note >}}

The {{<katex inline keepfont>}}\KaTeX{{</katex>}} library uses a serif-font by default (and makes it 20% larger).
This may or may not be ideal for your use.
Probably if you have a nice big formula on your page, it looks kind of fancy to have a different font.
<figure>
  <katex-block>
      \sum_{\substack{0<i<m\\0<j<n}}i^j
  </katex-block>
  <figcaption>Some nice example sum</figcaption>
</figure>

Even a slightly complex formula (like {{<katex inline>}}\frac{1}{\infty}{{</katex>}}) actually looks quite good. However saying that avogadro's number is {{<katex inline>}}6.02214 \times 10^{23} mol^{-1}{{</katex>}} just looks weird.

One can argue whether one is not just better off using plain HTML for such simple things: avogadro's number is {{< rawhtml >}}6.02214 &times; 10<sup>23</sup> <i>mol<sup>-1</sup></i>{{< /rawhtml >}} (`6.02214 &times; 10<sup>23</sup> <i>mol<sup>-1</sup></i>`), but it feels not good to create html-math when you actually have a math library.

In order to allow such usage, I include some SCSS in my system to counter the {{<katex inline keepfont>}}\KaTeX{{</katex>}} fonts:

```scss
/* scss: */
katex-inline.keepfont {
    .katex, .mathnormal, .mathit, .mathrm {
        font-family: inherit;
    }
    .katex {
        font-size: 1em;
    }
}
/* or plain css: */
katex-inline.keepfont .katex, katex-inline.keepfont .mathnormal, katex-inline.keepfont .mathit, katex-inline.keepfont .mathrm { font-family: inherit; }
katex-inline.keepfont .katex { font-size: 1em; }
```

As a result I can write `avogadro's number is <katex-inline class="keepfont">6.02214 \times 10^{23} mol^{-1}</katex-inline>` and the result is avogadro's number is {{<katex inline keepfont>}}6.02214 \times 10^{23} mol^{-1}{{</katex>}}.

It should be noted that there is probably a good reason why {{<katex inline keepfont>}}\KaTeX{{</katex>}} has its own fonts; probably lots of glyphs don't exist in normal fonts, or alignments don't work out in normal fonts (~~you can actually already see that in the example, where the last letter of _mol_ intersects the minus sign of the superscript~~ -- current css doesn't update `font-family` only `font-size`, so you cannot see that here anymore).
So using this is at your own risk, and I plan not to use it for anything but the simplest formulas inline.


## Escaping
The code described in this blog gets the text to render from the `innerText` field of the element.
This means that all non-html-safe characters _should_ be escaped.
For instance, in order to produce {{<katex inline>}}x^2 < x^3 | x > 1{{</katex>}} should be written as

```
<katex-inline>x^2 &lt; x^3 | x &gt; 1</katex-inline>
```

The HTML parser is generally very forgiving through, and things will work just fine if you write

```
<katex-inline>x^2 < x^3 | x > 1</katex-inline>
```

However by being sloppy, you do run into the possibility that you create something that could be seen by the HTML parser as an HTML tag, and then things go wrong.
For instance (the non-sensical):
```
<katex-inline>x^2 <x x> 1</katex-inline>
```
actually sees the `<x x>` as an opening tag `<x>` with an attribute `x`, so things break horribly while searching for the `</x>` tag.
*For good practice, always HTML escape your element.*

Be aware that if your source-code goes through other systems (markdown, liquid, etc), you may need to escape / work around that too.
However liquid can also help you:
```
{% raw %}<katex-inline>{{ 'x^2 <x x> 1' | escape }}</katex-inline>{% endraw %}
```
results in {{<katex inline>}}x^2 <x x> 1{{</katex>}} (because the `| escape` HTML-escapes the content. Be aware though that because you're within liquid quotes, you have problems with curly brackets and backslashes and quotes....)

To really be safe (from almost everything) use liquid `{% capture %}` combined with `{% raw %}`..
```
{% capture formula %}
{% raw %}
  \text{Murphy's law} <x>  | \frac{1}{0}
{% endraw %}
{% endcapture %}
<katex inline>{{ formula | escape }}</katex-inline>
```
{{<katex inline keepfont>}}\text{Murphy's law} <x>  | \frac{1}{0}{{</katex>}}

## Errors
The code in this blogpost has the setting `throwOnError: false`.
This means that in case of an error in the thing to generate, the formula is shown in red.
For instance:
```
<katex-inline>\frac{3}</katex-inline}}
```
renders as {{<katex inline nothrowonerror>}}\frac{3}{{</katex>}} (because `\frac` expects 2 parameters).

## Available functions and codes
I find [the supported functions page](https://katex.org/docs/supported.html#style-color-size-and-font) and [the support table page](https://katex.org/docs/support_table.html) extremely helpful as a cheatsheet on how to do things in {{<katex inline keepfont>}}\KaTeX{{</katex>}}.
Probably it only describes {{<katex inline>}}\frac{1}{100}^{th}{{</katex>}} of what is possible; feel free to link to your favourite cheatsheet/documentation in the comments.

## Security
The method described in this blog tells {{<katex inline keepfont>}}\KaTeX{{</katex>}} to render things securely, specifically `trust` is set to `false` (the default value) on render.
This means that (in theory) you could render `<katex-inline>` tags on your webpage that were created by untrusted users ([read more on the trust-setting](https://katex.org/docs/options.html)), however if you intend to do that, I would advise to have an expert double-check if things actually work as you  (and I) expect.

Note that by setting the `trust` option to `true`, some more {{<katex inline keepfont>}}\KaTeX{{</katex>}} tags are unlocked.
If you need these, it should be easy to enable this option (put it in the code right next to `throwOnError`).


## Examples
```xml
avogadro's number is <katex-inline class="keepfont">6.02214 \times 10^{23} mol^{-1}</katex-inline>.
```
avogadro's number is {{<katex inline keepfont>}}6.02214 \times 10^{23} mol^{-1}{{</katex>}}.

```xml
The progress is controlled by <katex-inline>x = \overbrace{a+b+c}^{\text{these are the major terms}} + \epsilon</katex-inline>.
```
The progress is controlled by {{<katex inline>}}x = \overbrace{a+b+c}^{\text{these are the major terms}} + \epsilon{{</katex>}}.

```xml
<katex-block>
\begin{CD}
   A @>a>> B \\
@VbVV @AAcA \\
   C @= D
\end{CD}
</katex-block>
```

{{<katex>}}
\begin{CD}
   A @>a>> B \\
@VbVV @AAcA \\
   C @= D
\end{CD}
{{</katex>}}

Probably there are much fancier examples; however it's probably better to google them :).

OK one more...
```
{% capture formula %}
{% raw %}
\begin{align*}
  \mathcal{L} = &- \frac{1}{4} F_{\mu \nu} F^{\mu \nu} \\
      &+ i \bar{\psi} \cancel{D} \psi + h.c. \\
      &+ \bar{\psi}_i y_{ij} \psi_j \phi + h.c. \\
      &+ |D_\mu \phi|^2 - V(\phi)
\end{align*}
{% endraw %}
{% endcapture %}
<katex-block>{{ formula | escape }}</katex-block>
```

{{<katex>}}
\begin{align*}
  \mathcal{L} = &- \frac{1}{4} F_{\mu \nu} F^{\mu \nu} \\
      &+ i \bar{\psi} \cancel{D} \psi + h.c. \\
      &+ \bar{\psi}_i y_{ij} \psi_j \phi + h.c. \\
      &+ |D_\mu \phi|^2 - V(\phi)
\end{align*}
{{</katex>}}
