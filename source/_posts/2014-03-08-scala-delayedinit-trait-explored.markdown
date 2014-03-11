---
layout: post
title: "Scala DelayedInit Trait Explored"
date: 2014-03-08 15:58:24 -0500
comments: true
categories: [Scala]
keywords: scala, delayedInit, trait, invocation order
description: "DelayedInit is a useful Trait but can cause frustrations in its subtleties"
---

The App trait is often used for making easily executable code. It is a display of how elegant the language can be. A hook for those new to the Scala language. Put a "Hello World!" app from Scala beside a Java app and you'll notice among other things the lack of a `main` method from the former. Cool - look how succinct Scala is!

The darker side of this nifty feature is rather subtle. Especially until the [open major bug since 2.9.1](https://issues.scala-lang.org/browse/SI-4680) is resolved. I came across it while using the [Smoke](https://github.com/mDialog/smoke) HTTP service library and mixing it into a trait. Smoke extends the DelayedInit trait for initializing the server before executing the body of the main App object. It's a nice use case.

The Scaladoc for [DelayedInit](http://www.scala-lang.org/api/2.10.3/#scala.DelayedInit) explicitly states that traits don't automagically benefit from mixing in DelayedInit like a class or object would. The doc has an example of how to define the  delayedInit method within a trait that will be executed as expected. In search of the `SI-4680` bug I performed a series of investigative scenarios, which are presented here.

The gist is [here](https://gist.github.com/tysonjh/9438697).

### Test Traits

A simple trait that extended the DelayedInit trait, along with a couple others for mixins.

``` scala 
trait Stage1 extends DelayedInit {
  println("Stage1 constructor")

  override def delayedInit(x: => Unit) {
    println("Stage1 delayedInit before x")
    x
    println("Stage1 delayedInit after x")
  }
}

trait Stage2 {
  println("Stage2 constructor")
}

trait Stage3 {
  println("Stage3 constructor")
}
```

Traits can be mixed in using several different techniques. Here are some with regular mixins and some with dependency injection.

``` scala
trait Stage123 extends Stage1 with Stage2 with Stage3 {
  println("Stage123 constructor")
}

trait Stage1Di23 extends Stage1 {
  this: Stage2 with Stage3 =>
  println("Stage1Di23 constructor")
}

trait Stage12Di3 extends Stage1 with Stage2 {
  this: Stage3 =>
  println("Stage12Di3 constructor")
}
```

### SI-4680 Bug Example

A trait with The empty initialization code in the instantiated trait results in the `delayedInit` method not being invoked.

``` scala
/*
Stage1 constructor
Stage2 constructor
Stage3 constructor
Stage123 constructor
Main application constructor end
 */
object Main123 extends App {
  println("Main application constructor start")
  new Stage123 {}
  println("Main application constructor end")
}
```

The result of running this object is shown in the comments. All of the initializations of the various traits were executed, but the `delayedInit` method was never called! Note that it is not possible to instantiate a trait, there is no constructor. By providing `{}` we are providing a constructor and an anonymous class that subclasses the trait `Stage123`. This is the empty constructor that results in bug SI-4680.

### SI-4680 Bug Workaround

``` scala
/*
Main application constructor start
Stage1 constructor
Stage2 constructor
Stage3 constructor
Stage123 constructor
Stage1 delayedInit before x
Stage123 NonEmpty constructor
Stage1 delayedInit after x
Main application constructor end
 */
object Main123NonEmpty extends App {
  println("Main application constructor start")
  new Stage123 {
    println("Stage123 NonEmpty constructor")
  }
  println("Main application constructor end")
}
```

In this example the constructor is populated with a `println` for illustrating the example but the constructor could be `{ { } }` or ` {Unit} `, comments don't count.

### Dependency Injection

``` scala
trait Stage1Di23 extends Stage1 {
  this: Stage2 with Stage3 =>
  println("Stage1Di23 constructor")
}

/*
Main application constructor start
Stage1 constructor
Stage1Di23 constructor
Stage2 constructor
Stage3 constructor
Stage1 delayedInit before x
Stage1Di23 NonEmpty constructor
Stage1 delayedInit after x
Main application constructor end
 */
object Main1Di23NonEmpty extends App {
  println("Main application constructor start")
  new Stage1Di23 with Stage2 with Stage3 {
    println("Stage1Di23 NonEmpty constructor")
  }
  println("Main application constructor end")
}
```

There are lots of great articles on DI in Scala, don't use this as one. What is interesting here is the order of execution. The body of our instantiated anonymous class of type `Stage1Di23 with Stage2 with Stage3` is executed within the `delayedInit` method now that it is not empty. I was surprised by the output, what I expected to see was `Stage1, Stage1Di23, Stage3, Stage2`, following the idea where traits are generally layered on from right to left. To better understand what is happening with the template evaluation, the `Stage2` trait has been mixed in as a base class instead of an injected dependency,

``` scala
trait Stage12Di3 extends Stage1 with Stage2 {
  this: Stage3 =>
  println("Stage12Di3 constructor")
}

/*
Main application constructor start
Stage1 constructor
Stage2 constructor
Stage12Di3 constructor
Stage3 constructor
Stage1 delayedInit before x
Stage12Di3 NonEmpty constructor
Stage1 delayedInit after x
Main application constructor end
 */
object Main12Di3NonEmpty extends App {
  println("Main application constructor start")
  new Stage12Di3 with Stage3 {
    println("Stage12Di3 NonEmpty constructor")
  }
  println("Main application constructor end")
}
```

This clears things up a bit more. This is similar to the Example 5.1.1 from the Scala Specification,

{% blockquote http://www.scala-lang.org/docu/files/ScalaReference.pdf Scala Specification %}
Example 5.1.1 Consider the following class deﬁnitions:
class Base extends Object {}
trait Mixin extends Base {}
object O extends Mixin {}
In this case, the deﬁnition of O is expanded to:
object O extends Base with Mixin {}
{% endblockquote %}

It would be nice to see the actual output from the compiler. The scalac `-print` option can do this and removes all Scala specific features and print to standard output. Now it's clear that first the superclass constructor of the anonymous class is evaluated. Followed by constructors of all base classes that are mixed in, in reverse order of the occurrence in linearization. Then the `delayedInit` method is called for each trait that extends the `DelayedInit` trait and implements the `delayedInit` method, first `App`, then `Stage1`.

``` scala
final class Test$$anon$1 extends Object with Stage12Di3 with Stage3 {
  override def delayedInit(x: Function0): Unit = Stage1$class.delayedInit(Test$$anon$1.this, x);
  def <init>(): anonymous class anon$1 = {
    Test$$anon$1.super.<init>();
    Stage1$class./*Stage1$class*/$init$(Test$$anon$1.this);
    Stage2$class./*Stage2$class*/$init$(Test$$anon$1.this);
    Stage12Di3$class./*Stage12Di3$class*/$init$(Test$$anon$1.this);
    Stage3$class./*Stage3$class*/$init$(Test$$anon$1.this);
    Test$$anon$1.this.delayedInit(new anonymous class anon$1$delayedInit$body(Test$$anon$1.this));
    ()
  }
}
```


