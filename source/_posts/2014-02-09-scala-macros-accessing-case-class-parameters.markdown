---
layout: post
title: "Scala Macros: accessing case class parameters"
date: 2014-02-09 00:05:21 -0500
comments: true
categories: [Scala, Macros]
keywords: scala, macros, quasiquotes, case class, fields
description: "How to access case class parameters with Scala Macros"
---

Gaining access to a case class' constructor parameters within a macro implementation was not obvious at first. One approach is
 to make use of the `WeakTypeTag`. Consider the example macro implementation with the following signature,

```scala 
def macroImpl[A: c.WeakTypeTag](c: Context)
```

To gain access to the `Type` of `A`,

```scala
val tpe: Type = weakTypeOf[A]
```

The trait `Type` has a method `declarations` that returns a `MemberScope` which is essentially an `Iterable[Symbol]` collection. The `Symbol` is of interest, for it contains useful information about anything in Scala that can be assigned a name.

```scala
val decl: MemberScope  = tpe.declarations
```

A simple `collect` and the right partial function and presto,

```scala
val params: Iterable[Name] = decl.collect { 
  case param if param.isTerm && param.asTerm.isCaseAccessor => field.name 
}
```

`params` contains the iterable collection of the case class parameter names.