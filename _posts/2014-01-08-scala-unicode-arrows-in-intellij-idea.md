---
layout: post
title:  "Scala Unicode Arrows in IntelliJ IDEA"
date:   2014-01-08 11:50:13
categories: scala intellij
---

Several of my colleagues love [IntelliJ](http://www.jetbrains.com/idea/) for coding in Scala.  I was pretty happy with [Sublime Text 2](http://www.sublimetext.com) (and still use it for Ruby/Python/Shell/whatever) but the lack of code completion was really starting to affect my productivity.  I spent way too much time looping through the edit/compile/fix typo cycle.

Before I could switch though, I really wanted my fancy arrows in Scala!  I have a GitHub project which adds tab completion in Sublime Text for Scala to [turn "=>" into "⇒", "->" into "→" and "<-" into "←"](https://github.com/themodernlife/SublimeScalaArrows).  Turns out this is really easy to accomplish in IntelliJ as well!  You just need to create a few [Live Templates](http://www.jetbrains.com/idea/webhelp/live-templates.html)

The `Abbreviation` field should be set to "=>" and the `Template text` field should be "⇒ " (I left a trailing space so my cursor position gets updated).  Repeat for any other operators.  

Also ensure that you change the applicable context to `Scala`.

Now when you type "=>" and hit <TAB> you should get a unicode arrow, "⇒"!

![Adding a live template for Scala Unicode arrows](/images/live-templates.png)

![Applicable in Scala](/images/applicable-in-scala.png)
