---
layout: post
title: "Feature Toggling using Scala Quasiquote Macros"
date: 2014-01-29 02:36:36 -0500
published: true
comments: true
categories: [Scala, Macros]
keywords: scala, macros, quasiquotes, feature toggle, kill switch
description: "A discussion of Scala macros and quasiquotes to facilitate feature toggling"
---

It's tough to dream up reasons to use macros in Scala. When our team began considering feature toggling 
to avoid experiencing another merging nightmare, it seemed like an excellent use case for a macro. A feature 
toggle is simply a construct that wraps a block of code, executing it if the feature is enabled.
It is also referred to as kill switching, feature flipping, etc. If you are interested on learning more, check out
[this article](http://martinfowler.com/bliki/FeatureToggle.html) by Martin Fowler. According to Fowler's article
the toggles we will be supporting are _business toggles_. Here is the basic idea,

```scala Toggle Flip Pseudocode
    if(featureEnabled) {
    	doFeatureStuff()
    } else {
    	doNotDoFeatureStuff()
    }
```

Scala macros are great for generating code, so it should be a simple matter to generate the conditional wrapper
around a feature block.

### Goals

1. Macro is simple to use
2. Macro wraps feature code with a conditional
3. Macro boolean condition implementation is extensible

The third point is to facilitate different mechanisms for querying a feature's toggle state. The first use case
for this feature will be writing hard wired scenarios of the boolean condition for testing purposes. Already I
am working on a Redis backed boolean conditional that polls periodically for changes in feature flags.

### Macros

Jumping right into the coding turned out to be the wrong approach. Macro programming in Scala is tough. It requires
an intimate knowledge of how the AST is built and the symbols used. The documentation scratches the surface but leaves many 
questions unanswered. Finding examples online helped but I still struggled to massage them into a workable solution. Furthermore the
examples I found used a lot of deprecated methods.

I'm not going to bother writing about directly manipulating the AST to create macros in Scala. At the end of
this post exists a link section, in it you will find some great articles and examples of manipulating the AST. 
We are going to take a look at using *quasiquotes*.

### Quasiquotes

Shipping with Scala 2.11 but supported in 2.10 with the [macro paradise](http://docs.scala-lang.org/overviews/macros/paradise.html)
plugin is [quasiquotes](http://docs.scala-lang.org/overviews/macros/quasiquotes.html). The basic idea is that they allow macros 
to be written similar to normal Scala code. Quasiquotes use the familiar concept of 
[string interpolation](http://docs.scala-lang.org/overviews/core/string-interpolation.html). *If you are new
to writing macros start with quasiquotes!*

There is an excellent [sbt-example-paradise](https://github.com/scalamacros/sbt-example-paradise) repository on github that will
help get started.

#### Define the macro

```scala Flipper Macro Definition https://github.com/tysonjh/dolphin/blob/master/src/main/scala/Dolphin.scala Source File
trait Dolphin {
  def flipper[T](featureName: String, config: QueryableConfig)
                (fn: => T): Unit = macro DolphinMacro.flipperImpl[T]
 }
```

Nothing too magical happening here, the `macro` keyword is followed by the name of the static macro implementation method.

#### Implement the macro

```scala Flipper Macro Implementation https://github.com/tysonjh/dolphin/blob/master/src/main/scala/Dolphin.scala Source File
object DolphinMacro {
  def flipperImpl[T](c: Context)
                    (featureName: c.Expr[String],
                     config: c.Expr[QueryableConfig])
                    (fn: c.Expr[T]): c.Expr[Unit] = {
    import c.universe._
    val result = {
      q"""
        if($config.isFeatureOn($featureName)) {
          $fn
        } else {}
      """
    }
    c.Expr[Unit](result)
  }
}
```

This represents the implementation of the macro using quasiquotes. The import of `c.universe._` is common in most
macro definitions. It provides many useful methods, for us it provides the `q` string interpolator on _line 8_. 
This is the start of the quasiquote. Appreciate how much it looks like plain old scala, compare it to any 
AST constructions on the [macro overview](http://docs.scala-lang.org/overviews/macros/overview.html) page on the 
Scala macro documentation to understand how *awesome* this is.

Conceptually we are printing out the symbols that construct the desired outcome and letting the macro paradise
plugin handle the rest of the heavy lifting. Much easier to read isn't it? Compare it to the `showRaw` output of the
AST we would need to build without quasiquotes in `res0`:

```scala The desired AST
scala> import scala.reflect.runtime.{universe => u}
import scala.reflect.runtime.{universe=>u}

scala> def flipper(featureEnabled: Boolean)(doFeatureStuff: => Unit)(doNotDoFeatureStuff: => Unit) = {
     | if(featureEnabled){ doFeatureStuff }
     | else doNotDoFeatureStuff
     | }
flipper: (featureEnabled: Boolean)(doFeatureStuff: => Unit)(doNotDoFeatureStuff: => Unit)Unit

scala> val expr = u reify (flipper _)
expr: reflect.runtime.universe.Expr[Boolean => ((=> Unit) => ((=> Unit) => Unit))] = 
Expr[Boolean => ((=> Unit) => ((=> Unit) => Unit))] ({((featureEnabled) => ((doFeatureStuff) => ((doNotDoFeatureStuff) => $read.flipper(featureEnabled)(doFeatureStuff)(doNotDoFeatureStuff))))})

scala> u showRaw expr.tree
res0: String = Block(List(), Function(List(ValDef(Modifiers(PARAM | SYNTHETIC), newTermName("featureEnabled"), TypeTree(), EmptyTree)), Function(List(ValDef(Modifiers(PARAM | SYNTHETIC), newTermName("doFeatureStuff"), TypeTree(), EmptyTree)), Function(List(ValDef(Modifiers(PARAM | SYNTHETIC), newTermName("doNotDoFeatureStuff"), TypeTree(), EmptyTree)), Apply(Apply(Apply(Select(Select(Select(Ident($line7.$read), newTermName("$iw")), newTermName("$iw")), newTermName("flipper")), List(Ident(newTermName("featureEnabled")))), List(Ident(newTermName("doFeatureStuff")))), List(Ident(newTermName("doNotDoFeatureStuff"))))))))
```

A more accurate way according to Imran Rashid's great post on [Learning Scala Macros](http://imranrashid.com/posts/learning-scala-macros/) is to let the
scalac compiler do it for you (below). Though I still prefer quasiquotes to this!

```scala
scalac -Xplugin macro-paradise_2.10.2-2.0.0-SNAPSHOT.jar -deprecation -Xprint:parser -Ystop-after:parser -Yshow-trees-compact *.scala
```

#### Test the macro

```scala Flipper Macro Tests https://github.com/tysonjh/dolphin/blob/master/src/test/scala/DolphinSpec.scala Source File
class DolphinSpec extends FunSpec with Dolphin {
  def featureAlwaysOnConfig = new QueryableConfig {
    def isFeatureOn(name: String): Boolean = true
  }

  def featureAlwaysOffConfig = new QueryableConfig {
    def isFeatureOn(name: String): Boolean = false
  }

  describe("Flipper") {
    it("should execute function if feature is on"){
      var cnt = 0
      flipper("a", featureAlwaysOnConfig){cnt = cnt + 1}
      assert(cnt == 1)
    }
    
    it("should not execute function if feature is off"){
      var cnt = 0
      flipper("a", featureAlwaysOffConfig){cnt = cnt + 1}
      assert(cnt == 0)
    }
  }
}		
```

This is how the macro can be tested, again nothing special here. An interesting detail to note is that in the
macro definition we have a call by name parameter `fn: => T`, but in this implementation of the macro we have
the parameter `fn: c.Expr[T]` which appears to be call by value. The tests clearly show that the call by name
parameter is only evaluated by name. Attempting to change the implementation to `fn: c.Expr[=> T]` or `fn: => c.Expr[T]`
causes a compilation error. 

The limitation is that one may only inline `fn` and not invoke it, at least until
Scala 2.11 where `fn: c.Tree` may be used as the implementation parameter ([see here](https://issues.scala-lang.org/browse/SI-5778)).

### Summary

Quasiquotes are a welcomed addition to the macro scene. They make writing macros more accessible by eliminating the
necessity for deep AST knowledge before one gets started. The code in this post is part of 
the [Dolphin](https://github.com/tysonjh/dolphin) library for feature toggling in Scala.

### Links

* [Adventures with Scala Macros](http://www.scottlogic.com/blog/2013/06/05/scala-macros-part-1.html)
* [Simple macro-based scala example](https://gist.github.com/travisbrown/4234441)
* [Scala Macros](http://docs.scala-lang.org/overviews/macros/overview.html)
* [Quasiquotes](http://docs.scala-lang.org/overviews/macros/quasiquotes.html)
* [Use reify to get AST of expression](http://stackoverflow.com/questions/11055210/whats-the-easiest-way-to-use-reify-get-an-ast-of-an-expression-in-scala)
* [Learning Scala Macros](http://imranrashid.com/posts/learning-scala-macros/)
