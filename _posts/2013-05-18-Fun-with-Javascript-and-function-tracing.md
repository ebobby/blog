---
layout: post
title: Fun with Javascript and function tracing
date:  2013-05-18
tags: [programming]
---

<img style="float: right" src="/assets/bigbug.jpg" />

## The problem

It was a *normal* day at my [job](http://www.expensify.com), my team
mates were putting the finishing touches and bug fixes into our latest
[features](http://blog.expensify.com/2013/05/14/announcing-bill-processing-and-invoices-that-dont-suck/),
due to be released during [Finovate Spring](http://finovate.com/). I
wasn’t part of that project but found myself free of my own tasks and
decided to jump in to help. And so a bug report came to me.

The bug was simple, when adding some expenses to a report they weren’t
being displayed until you refreshed (this was our webapp) which is a bad
thing, you want to see stuff as you do it. So I embarked into a journey
of code exploring, setting breakpoints, running through the debugger. It
just took a couple of minutes to realize that the new expenses were
being displayed just to be immediately hidden again by some other part
of the code.

## What to do?

It didn’t take much guessing to know that the culprit was an incorrect
call to `jQuery.hide()`. So I needed to find out if the call
was being made on the element directly or on one of its parents. So I
set up to monitor all the calls to `jQuery.hide()` after
adding an expense and find out what was being hidden and where in the
code the call originated from. So what to do?

Normally is just a couple of choices:

  - [rgrep](http://www.gnu.org/software/emacs/manual/html_node/emacs/Grep-Searching.html)
    all the occurrences of the id or class of the element being hidden
    as to see what parts of the code were making a reference to it, this
    of course would only work if one of the parents of said element
    weren’t being hidden as opposed to just said element.
  - Place a [conditional breakpoint](https://developers.google.com/chrome-developer-tools/docs/javascript-debugging)
    on jQuery’s hide function. This is a moderately complex web
    application, we have dialogs, popover tips, etc., so the calls to
    <code>jQuery.hide()</code> are many.

None of these seemed very appealing or particularly fast. Specially the
debugging one which certainly would take me to the right answer albeit
after taking the time to filter out all the calls that weren’t the one I
was looking for.

I reminisced of Common Lisp’s [trace/untrace](http://clhs.lisp.se/Body/m_tracec.htm) macros. That
would be the ideal, having some code run whenever
<code>jQuery.hide()</code> was being called and display the
parameters/context so I could see what it was being called with, that
would allow me to narrow my search significantly. Then I realized this
is easily doable in Javascript (comments added for clarity):

{% highlight javascript %}
$.fn.hide = (function() {          // Executed on the spot to create a closure.
  var oldfn = $.fn.hide;           // Save jQuery's hide().
  return function () {             // Create the closure and return the function to replace it.
    console.log(this);             // Simply print the context.
    oldfn.apply(this, arguments);  // Apply jQuery's hide() so everything works normally.
  };
}());                              // Execute it and rewrite jQuery's.
{% endhighlight %}

I ran that in Chrome’s console, did the steps to reproduce the bug and
voilà. After that I knew which HTML elements were being hidden and I
easily found the one that was hiding the element I wanted. After that I
refreshed the page, and ran the same code as before, just with a
conditional to check for the element id and printing the stack trace
when that element was hidden and I found the part of the code that was
doing it and after that it was an easy fix.

## Tracing.js

After wrapping up the bug fix, I realized that the little trick to hook
into another function could be easily abstracted into a tiny library and
so I took it as a weekend project to do it. And so
[Tracing.js](https://github.com/ebobby/tracing.js) was born.

The premise is simple, it allows you to hook functions on your own to
other functions. There are two types of hooks, **before** and **after**,
as their name imply, *before* functions will be called before the actual
function call, and *after* will be called after the fact. It is been
pointed out to me that this pattern can also be classified as [Aspect
Oriented Programming](http://en.wikipedia.org/wiki/Aspect-oriented_programming)
but that wasn’t what I had in mind when I wrote this.

As a plus and because I always found this very useful in Common Lisp, I
added a `trace()` functionality that works exactly the same
as the source of inspiration.

{% highlight javascript %}
// Assuming we have previously defined these functions:
Tracing.trace('square', 'multiply', 'add');

square(2);
>  square called with arguments: (2)
>    multiply called with arguments: (2, 2)
>      add called with arguments: (0, 2)
>        add called with arguments: (1, 1)
>          add called with arguments: (2, 0)
>          add returned: 2
>        add returned: 2
>      add returned: 2
>      multiply called with arguments: (2, 1, 2)
>        add called with arguments: (2, 2)
>          add called with arguments: (3, 1)
>            add called with arguments: (4, 0)
>            add returned: 4
>          add returned: 4
>        add returned: 4
>        multiply called with arguments: (2, 0, 4)
>        multiply returned: 4
>      multiply returned: 4
>    multiply returned: 4
>  square returned: 4
{% endhighlight %}

You can find this code on [Github](https://github.com/ebobby/tracing.js).
