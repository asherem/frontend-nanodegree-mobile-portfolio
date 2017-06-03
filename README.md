# Udacity's Front End Website Optimization Project
---
The aim of this project is to ensure a Google PageSpeed score of at least 90 for `index.html`, as well as achieving 60 FPS for `pizza.html`. This will be accomplished by optimizing images, re-writing code, and re-structuring code blocks.


### index.html
---
Right now, the profile picture is far too large at 16 KB given how tiny it is. Ideally, it should be around ~3 KB maximum. A good optimization code for ImageMagick is the following:

`convert original-image.jpg -sampling-factor 4:2:0 -strip -quality 85 -interlace Plane -colorspace RGB optimized-image.jpg`

This reduces quality, uses Chroma sampling, and progressive JPEG rendering to save resources. In order to resize appropriately, I would first use a visual editor such as [paint.NET](https://www.getpaint.net/) in order to have greater control over the work. You can find more information about these strategies [here](https://developers.google.com/speed/docs/insights/OptimizeImages).

Notice that when you download the image of the pizzeria, it is actually a huge, 2.25 mb file which the browser is being asked to compress down to a low-quality, tiny version of itself. This means there are two separate hits to performance. The first is the fact that an extremely large image needs to be downloaded. The second, and less well-known one, is that the browser needs to perform a smaller rendering of that image, which wastes resources in and of itself. The key, then, is to not only optimize images, but to scale them down at the source to their ultimate display size before they even leave the code. Correcting this will provide great results.

There are also ways to improve the speed of Google Fonts and Google Analytics. Often, you might wish to inline JavaScript code in order to improve page speed, but this is not a good idea here since Google's is so large and complex to begin with. Another option is asynchronous loading, which renders the critical path before JavaScript. Note that this could create a flash of unstyled content (FOUC) if the page is too large, which is not at all the case here.

Thus the code:

`<script src="http://www.google-analytics.com/analytics.js"></script>`

can be transformed into:

`<script async src="http://www.google-analytics.com/analytics.js"></script>`

for asynchronous loading.

The same can be done for the Google Fonts code, although it is a more laborious process. In short, to achieve asynchronous loading for Google Fonts, you need to first remove this bit of code:

`<link href="//fonts.googleapis.com/css?family=Open+Sans:400,700" rel="stylesheet">`

and achieve the same effect (without loading up the `<head>`) by leveraging a JavaScript library called [Web Font Loader](https://github.com/typekit/webfontloader). It requires the following script in your page's `<head>` element, under your script `<script>` elements:

```javascript
<script>
    WebFontConfig = {
        google: { families: [ 'Open+Sans:400, 700:latin'] }
    };
    
    (function(d) {
        var wf = d.createElement('script'), s = d.scripts[0];
        wf.src = 'https://ajax.googleapis.com/ajax/libs/webfont/1.6.26/webfont.js';
        wf.async = true;
        s.parentNode.insertBefore(wf, s);
    })(document);
</script>
```

The following is also added to your CSS file:
```css
.wf-loading p { font-family: sans-serif; }
.wf-inactive p { font-family: sans-serif; }
.wf-active p { font-family: 'Alegreya', sans-serif; width:400px; margin-left:20px; }
.wf-loading h1 { font-family: sans-serif; font-weight: 400; font-size: 24px; }
.wf-inactive h1 { font-family: sans-serif; font-weight: 400; font-size: 24px; }
.wf-active h1 { font-family: 'Aclonica', sans-serif; font-weight: 400; font-size: 24px; }
.wf-loading h2 { font-family: sans-serif; font-weight: 300; font-size: 20px; color:#5E5E5E; }
.wf-inactive h2 { font-family: sans-serif; font-weight: 300; font-size: 20px; color:#5E5E5E; }
.wf-active h2 { font-family: 'Acme', sans-serif; font-weight: 300; font-size: 20px; color:#5E5E5E; }
```

We can also optimize the CSS delivery by deferring some of it for later in the critical rendering path. Although we cannot inline style.css due to the size of the code, the print.css code is both smaller and non-critical, since most users will not elect to print the page, anyway. Thus, we can defer print.css until later, like this, into the <body> section of the page:

```javascript
    <noscript id="deferred-styles">
      <link rel="stylesheet" types="text/css" href="css/style.css">
      <link rel="stylesheet" type="text/css" href="css/print.css"/>
    </noscript>
    <script>
      var loadDeferredStyles = function() {
        var addStylesNode = document.getElementById("deferred-styles");
        var replacement = document.createElement("div");
        replacement.innerHTML = addStylesNode.textContent;
        document.body.appendChild(replacement)
        addStylesNode.parentElement.removeChild(addStylesNode);
      };
      var raf = requestAnimationFrame || mozRequestAnimationFrame ||
          webkitRequestAnimationFrame || msRequestAnimationFrame;
      if (raf) raf(function() { window.setTimeout(loadDeferredStyles, 0); });
      else window.addEventListener('load', loadDeferredStyles);
    </script>
```

This means you take `print.css` out of the `<head>` portion of index.html. Due to the small nature of `index.html`, the same can be done, in fact, with `style.css`, while still having a low possibility of a FOUC event. For larger, more complex pages, the CSS should usually be loaded
in the `<head>`.

For a peculiar sort of FOUC event, if the CSS in `pizza.html` where likewise deferred, there will be a brief flash before the load is completed.

### project-2048.html
---

For images, the steps are the same as above. Make sure that you resize the original image down to its actual display size to avoid using the browser's own compressive resources.

The rest is the same: set the JS files to `async`, and defer CSS shortly before `<body>` closes.

### project-mobile.html
---
Follow the above instructions.

### project-webperf.html
---
Follow the above instructions.

### pizza.html
---
This is the page with the largest and most complex optimizations. Most of the work is done in JavaScript, although there is some image optimization work, as well, particularly for the pizzeria image. It will likely be around ~100 kb, this, which is large but still over 95% smaller than what it currently is.

The individual pizzas might be optimized, as well, but this is tricky, as the JavaScript code itself re-renders some pizzas, and there are potentially 4-5 pizza images to adjust. Also, we will be reducing the actual number of displayed pizzas from ~200 down to a responsive number, depending upon one's viewport. Finally, the images are saved as `.png` files, and cannot be saved as `.jpeg` for compression. This is because `.png`, like `.gif`, offer image transparency which is needed in this case. Thus, any adjustments need to keep this in mind.

As noted, the biggest issue here is the JavaScript, so let's get into it. This is the page's scroll function:

```javascript
function updatePositions() {
  frame++;
  window.performance.mark("mark_start_frame");

  var items = document.querySelectorAll('.mover');
  for (var i = 0; i < items.length; i++) {
    var phase = Math.sin((document.body.scrollTop / 1250) + (i % 5));
    items[i].style.left = items[i].basicLeft + 100 * phase + 'px';
  }
```

The easiest part to resolve is the `.querySelectorAll` method. In fact, if you search through the entire file and look for `querySelector` in the buffer (around 10 lines, maybe), these are bits of code which need to be replaced with `getElementsByClass` or `getElementsById`, depending on whether you are grabbing a CSS class or ID. According to [JS Perf](https://jsperf.com/), this might lead to a 10-15% performance increase over `querySelector`.

Moving on, we get to this:

```javascript
for (var i = 0; i < items.length; i++) {
```

This is a slightly expensive operation, as the `items.length` method is a double-lookup of what might otherwise be a single variable. In this way, `items.length` can be cached by use of a variable for some increase in speed: `var totalLength = items.length;`. Note that this replicates what is already happening in `var items = document.querySelectorAll('.mover');`.

Later, the same issue comes up:

```javascript
var phase = Math.sin((document.body.scrollTop / 1250) + (i % 5));
```

As before, we have wasted resources, not just via a double look-up, but a triple look-up. Same as before, let's set a variable to deal with it: `var scroll = document.body.scrollTop;`.

Note that `element.scrollTop` returns the number of pixels the element's content has been scrolled vertically. It is a standard feature of scrollable webpages. The `%` is a modulo which forces phase to yield 5 of the same numbers for instances of i > 4.

These will actually be cached as with the previous code, using a variable set to an array, with the calculation set outside of the loop so that it's called only once.

```javascript
items[i].style.left = items[i].basicLeft + 100 * phase + 'px'
```

This is actually what creates the pizza motions as you scroll. In short, it changes the 'left' styling of the element to a string with a pixel value. The line before deals with scrolling; this deals with the on-page motion.

But `.left` is one of the many paint-triggering events in CSS, an effect which can be achieved through the less-expensive `.transform:`

```javascript
    var left = -items[i].basicLeft + 1000 * phase + 'px';
 		items[i].style.transform = "translateX("+left+") translateZ(0)";
```

In the end, we get the following code [adapted from mcs](https://discussions.udacity.com/t/project-4-how-do-i-optimize-the-background-pizzas-for-loop/36302?source_topic_id=248974):

```javascript
function updatePositions() {
    frame++;
    window.performance.mark("mark_start_frame");

    var items = document.getElementsByClassName('mover');
    var scroll = document.body.scrollTop;
    var constArray = [];
    var i;

    for (i = 0; i < 5; i++) {
      constArray.push(Math.sin((scroll / 1250) + i));
    }

    for (i = 0; i < items.length; i++) {
        var phase = constArray[i % 5];
        var left = -items[i].basicLeft + 1000 * phase + 'px';
                    items[i].style.transform = "translateX("+left+") translateZ(0)";
    }

    window.performance.mark("mark_end_frame");
    window.performance.measure("measure_frame_duration", "mark_start_frame", "mark_end_frame");
    if (frame % 10 === 0) {
        var timesToUpdatePosition = window.performance.getEntriesByName("measure_frame_duration");
        logAverageFrame(timesToUpdatePosition);
    }
}
```

This covers the majority of code. One last optimization can be made below:

```javascript
// runs updatePositions on scroll
window.addEventListener('scroll', updatePositions);

// Generates the sliding pizzas when the page loads.
document.addEventListener('DOMContentLoaded', function() {
  var cols = 8;
  var s = 256;
  for (var i = 0; i < 200; i++) {
    var elem = document.createElement('img');
    elem.className = 'mover';
    elem.src = "images/pizza.png";
    elem.style.height = "100px";
    elem.style.width = "73.333px";
    elem.basicLeft = (i % cols) * s;
    elem.style.top = (Math.floor(i / cols) * s) + 'px';
    document.querySelector("#movingPizzas1").appendChild(elem);
  }
  updatePositions();
});
```

According to [jshank](https://github.com/jshanks24/Udacity-Website-Optimization), there is some fine-tuning to be made for the number of pizzas on screen, which ought to be loaded dynamically rather than statically. The new code is as follows:  

```javascript
// runs updatePositions on scroll
window.addEventListener('scroll', updatePositions);

// Generates the sliding pizzas when the page loads; README for attribution
document.addEventListener('DOMContentLoaded', function() {
    var cols = 8;
    var s = 256;
    var pizzaNum = (window.innerHeight / 75) + (window.innerWidth / 75);
    for (var i = 0; i < pizzaNum; i++) {
        var elem = document.createElement('img');
        elem.className = 'mover';
        elem.src = "images/pizza.png";
        elem.style.height = "100px";
        elem.style.width = "73.333px";
        elem.basicLeft = (i % cols) * s;
        elem.style.top = (Math.floor(i / cols) * s) + 'px';
        document.getElementById("movingPizzas1").appendChild(elem);
    }
    updatePositions();
});
```