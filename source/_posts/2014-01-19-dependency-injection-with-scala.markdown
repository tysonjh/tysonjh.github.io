---
layout: post
title: "Dependency Injection with Scala"
date: 2014-01-19 11:56:43 -0500
published: false
comments: true
categories: [Scala, Design Patterns, Dependency Injection]
keywords: Scala, Design Patterns, Dependency Injection, Cake, Macwire
description: "Discussion and example of two differing dependency injection design patterns in Scala - Cake & Macwire"
---

Dependency injection (DI) is a design pattern that has been described in depth both in print and on the web, my goal is 
to illustrate an example of how I used it in practice with satisfying results. If you would like to learn more about DI,
do what I did and look to an expert in your area of interest, [Real-World Scala: Dependency Injection (DI)](http://jonasboner.com/2008/10/06/real-world-scala-dependency-injection-di/) by Jonas Bonér is a great read, in fact I question the worth of this post after reading it again.

[John Munsch](http://stackoverflow.com/a/1638961/1274818) provides an elegant and nostalgic explanation,

>When you go and get things out of the refrigerator for yourself, you can cause problems. You might leave the door open, you might get something Mommy or Daddy doesn't want you to have. You might even be looking for something we don't even have or which has expired.

>What you should be doing is stating a need, "I need something to drink with lunch," and then we will make sure you have something when you sit down to eat.

## Example Use Case: NOSQL data cleanup

* [Source using Cake Pattern](http://github.com/tysonjh/)
* [Source using Macwire](http://github.com/tysonjh/)

#### Description

There is a data issue in a production NOSQL database, we need to clean it up. Anything touching data in production makes me sweat. Generally with NOSQL you design your schema to support specific queries since joins must be done by the application or through creative indexing. When an unexpected need to join unrelated data arises things get interesting. So we must proceed with caution.

#### Triggers

An administrator will execute the script from the command line.

#### Actors

1. A database access object that provides a domain specific view into our database
2. A database client library that provides access to the operations of a database
3. A command line interface for controlling the flow of the application

#### Preconditions

1. The database is available
2. The data model is a key/value store that _for all keys:K there exists a value:V where K:String -> V:Int_

#### Goals

1. All keys are capitalized and trimmed of white space
2. The lowest value of any keys that differ only by case and whitespace are kept, all others are removed 

### Design Considerations

You should be screaming TDD right now. This means unit tests and lots of them, which means we need a mock/stub implementation of our database. I've written two different implementations of the DI pattern, the first using Cake as discussed by Jonas Bonér and the next using Macwire. The source files for both examples are provided as links above. I'm providing a Macwire example because I wanted to try it out, in practice I used the cake pattern in this instance.

### Implementation Highlights

Here is a snippet of the `DatabaseComponent` trait we use for accessing the database,

> insert DatabaseComponent snippet

Notice we leave the `client` value as an abstract member, of type `DatabaseClient` which is a nested trait. This is what we will be using to inject different implementations of the `client`. The nested trait simply cleans up the namespace so all of those methods (get, set, exist, delete) are only visible from within the `DatabaseClient` object as we would expect, it also provides us with a greater flexibility for where we instantiate our `client` object.

#### Components

Below are the three implementations of the `DatabaseComponent`, the real `Database`, the write simulated `DatabaseWriteSimulator` and the test `TestDatabase` respectively

> database snippet

The `DatabaseWriteSimulator` turned out to be quite useful and simply prints operations that alter the database instead of actually performing them. It was a nice way to test the script on real data in an environment.

> simulator snippet

The `TestDatabase` is very similar to the `Database` trait which fully implements a `DatabaseComponent`, it could easily be replaced using a mocking framework.

> test snippet

#### Wiring Components with Modules

Modules are one way to wire the various traits together as required by DI. This really lets me visualize the whole cake pattern, I like to picture the class names as the icing and Scala symbols like `class`, `extends` and `with` as the cake.

`object Production extends ProductDaoComponent with Database`

Stack those words up starting at the bottom with `object`, moving right ending with `Database` on top. Methods and members at the bottom of the cake get overridden or implemented by things higher up in the cake. As long as you have no unimplemented methods or instantiated members you've got a full cake!

> insert cake image

