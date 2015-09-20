+++
date = "2015-09-12T22:01:13+03:00"
draft = false
title = "True fail-fast async error handling with Expression"

+++

How it was said in the previous [article](http://koff.io/posts/290071-make-async-with-scalaz-either-and-futures) there is no way to do truly fail-fast async error handling using only scala or scalaz.

Look at the example below:

```scala
val longFut = longFuture() // very long future
val shortFut = shortFuture()
val failedFut = failedFuture() // throw new IllegalStateException("future is failed")

val result = for {
  long <- longFut
  short <- shortFut
  failed <- failedFut
} yield {
  long + " | " + short + " | " + failed
}
```
In that example we will wait all the futures until we get `IllegalStateException` because `for-comprehension` always handle futures in the order which we define them since Scala translates the example above to this:

```scala
longFut.flatMap { long =>
  shortFut.flatMap { short =>
    failedFut.map { failed =>
      long + " | " + short + " | " + failed
    }
  }
}
```

But it is possible to avoid this problem with Expression library([link](https://github.com/jedesah/computation-expressions "Computation-expressions"))

<!--more-->

## Simple example
You can checkout a project with examples for this article [here](https://github.com/coffius/koffio-expression-example "Sources for examples")
Let's take a look at a simple example at first:

```scala
def future1(): Future[String] = Future.successful("future1")
def future2(): Future[String] = Future.successful("future2")

def resultCalc(str1: String, str2: String): String = str1 + " | " + str2

def expression(): Future[String] = Expression[Future, String] {
  val result1 = extract(future1())
  val result2 = extract(future2())
  resultCalc(result1, result2)
}
```

In the example above we defined `Expression` block, executed our async methods inside it, used `extract(...)` method to get a result from a future and passed the future resuts as params to `resultCalc(...)`.
`expression()` method will behave almost like this code:

```scala
def forComprehension(): Future[String] = {
  val fut1 = future1()
  val fut2 = future2()
  for {
    result1 <- fut1
    result2 <- fut2
  } yield {
    resultCalc(result1, result2)
  }
}
```
except the case when the futures have finished with exceptions.

## Async fail-fast error handling
In this case `Expression` block will be completed with an error as soon as any of the futures is finished with an failure.
Take a look:

```scala
def expressionFailFast(): Unit = {
  val startTime = System.currentTimeMillis()
  val result = Expression[Future, String] {
    //same order as in the for-comprehension example
    val long = extract(longFuture())    //Thread.sleep(15000)
    val short = extract(shortFuture())  //Thread.sleep(3000)
    val failed = extract(failedFuture())//Thread.sleep(6000);throw new IllegalStateException()
    short + " | " + long + " | " + failed
  }
  try {
    Await.result(result, 30 seconds)
  } catch {
    case err: Throwable => println("error: " + err.getMessage)
  }

  val duration = System.currentTimeMillis() - startTime
  println("duration: " + duration / 1000.0d)
}
```

If you run this code you will see that the code will be completed during ~6 seconds instead of 15 seconds for `forComprehensionFailFast(...)` below:

```scala
def forComprehensionFailFast(): Unit = {  
  val startTime = System.currentTimeMillis()

  val longFut = longFuture()
  val shortFut = shortFuture()
  val failedFut = failedFuture()

  val result = for {
    long <- longFut
    short <- shortFut
    failed <- failedFut
  } yield {
    long + " | " + short + " | " + failed
  }

  try {
    Await.result(result, 30 seconds)
  } catch {
    case err: Throwable => println("error: " + err.getMessage)
  }

  val duration = System.currentTimeMillis() - startTime
  println("duration: " + duration / 1000.0d)
}
```
So now it is not nessesary to wait long operations if any other operation already completed with an error. It might be helpful for working with external services which can be very slow.

## Futures and If-Statements

One more thing. Sometimes you may need to use `if-statement` blocks along with async operations. In this case Expression library can also be useful. Let's imagine that we want to get and to check some information. After the checking we decide what we will do next. And all these operations should be asynchronous.
If we use `for-comprehension` then a code will be like this:

```scala
def forComprehensionIf(): Future[String] = {
  (for {
    condition <- trueFuture()
  } yield {
    if(condition) {
      for{
        normal <- normalFuture()
        long <- longFuture()
      } yield {
        resultCalc(normal, long)
      }
    } else {
      for{
        failed <- failedFuture()
      } yield {
        resultCalc("default", failed)
      }
    }
  }).flatMap(identity)
}
```
It's quite tricky, isn't it? 
But if we use `Expression` then we can rewrite it like this:

```scala 
def expressionIf(): Future[String] = {
  //we can use extract(trueFuture())
  //or we can import Expression.auto.extract instead
  import com.github.jedesah.Expression.auto.extract
  Expression[Future, String] {
    if(trueFuture()) {
      //A-branch
      resultCalc(normalFuture(), longFuture())
    } else {
      //B-branch
      resultCalc("default", failedFuture())
    }
  }
}
```
Much simpler :)

## Git as Maven Repo

And one more trick for github. Sometimes you can find useful projects or libs which you want to use it in your sbt project as a managed dependency but they are not published on any public repositories. In this case [jitpack.io](https://jitpack.io/ "jitpack.io") might help you.
I've used it in the example project for this article to add [jedesah/computation-expressions](https://github.com/jedesah/computation-expressions "Computation-expressions") to `build.sbt`

```scala
//add jitpack.io to resolvers
resolvers += "jitpack.io" at "https://jitpack.io"

//add github project as a dependency
//https://github.com/jedesah -> "com.github.jedesah" as a group id
//project name "computation-expressions" as an artifact id
//and commit id "5ef11fc97c" as a revision
libraryDependencies += "com.github.jedesah" % "computation-expressions" % "5ef11fc97c"
```
That`s all. Now you can use any github projects as a sbt dependency.

## Links
* [Sources for examples](https://github.com/coffius/koffio-expression-example "Sources for examples") - sbt project with examples
* [jedesah/computation-expressions](https://github.com/jedesah/computation-expressions "Computation-expressions") - Expression project
* [jitpack.io](https://jitpack.io/ "jitpack.io") - package repository for GitHub
* [Building a Better Future](https://youtu.be/tU4pU5vaddU "Lecture from creators of Expression library") - Jean-Remi Desjardins & Eddie Carlson talk about how they have created Expression library
* [Slides](http://slides.com/jedesah/deck-1#/) - slides of a presentation. Can be useful.

