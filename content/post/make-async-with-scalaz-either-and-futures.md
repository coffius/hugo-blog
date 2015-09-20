+++
date = "2015-09-05T22:01:13+03:00"
draft = false
title = "Practical Scalaz: Make async operations with scalaz.Either and Futures"
+++

Many of us know about such library as scalaz. For those who don't know it is a library for functional programming in scala. You can find it [here](https://github.com/scalaz/scalaz "Scalaz on github"). 
Then I was trying to learn and understand this lib it was quite difficult to realize how exactly it can be used in a real code in a real system. I looked thought lots of articles about it, but there were only abstract examples. So I`ve decided to write a little example in order to show how scalaz can be used in a real system.

## Intro

**Idea:** Future and scalaz.Either can be used as a result of an asynchronous operation.

**Reason:** We must compose futures and eithers in order to deal with possible errors which may occur during execution of async operations.

For example we have to gather information from several different DBs and an external service like Twitter. We also don`t want to use exceptions as notifications about errors. Why? Because it sucks ^_^ It will be better if exceptions are used for something really exceptional. One more thing which we should implement is fail-fast error handling because it will be wasteful to continue program execution if it already contains some errors.

<!--more-->

## Simple example

You can find sources for this article [here](https://github.com/coffius/koffio-examples/blob/master/src/main/scala/io/koff/examples/async_services/EitherFutureSimpleExample.scala "Example")
As a result of an operation we use scalaz.Either. Let`t redefine either classes for better readability like that:

```scala
type ActionResult[T] = String \/ T
val ActionSuccess = \/-
val ActionFailure = -\/
```

`ServiceResult[T]` is a generic type for passing an operation result
We use `String` as error object for passing information about error, `-\/` indicates an error and `\/-` indicates success.

Define one more type to make possible async actions:

```scala
type FutureActionResult[T] = Future[ActionResult[T]]
```

and one more as the outcome

```scala
case class Outcome(value: String)
```

Then define async methods:

```scala
def doSuccessAction1(): FutureActionResult[Outcome] = {
  Future {
    Thread.sleep(5000)
    println("success-action#1 is completed")
    ActionSuccess(Outcome("success#first"))
  }
}
def doSuccessAction2(): FutureActionResult[Outcome] = { /*...*/ }
def doSuccessAction3(): FutureActionResult[Outcome] = { /*...*/ }
```

Now we have almost all what we need. Right now we have to write something like that to compose results of our methods:

```scala
def doUglyComplexAction(): FutureActionResult[String] = {
  //start execution in parallel
  val futureResult1 = doSuccessAction1()
  val futureResult2 = doSuccessAction2()
  val futureResult3 = doSuccessAction3()

  for{
    eitherResult1 <- futureResult1
    eitherResult2 <- futureResult2
    eitherResult3 <- futureResult3
  } yield {
    for {
      result1 <- eitherResult1
      result2 <- eitherResult2
      result3 <- eitherResult3
    } yield {
      // if all operations complete successfully we will concatenate string
      result1.value + " " + result2.value + " " + result3.value
    }
  }
}
```

There are several problems in the code above. 
The first one is that the second `for-comprehension` block will execute only after all of the futures will have finished. And the second problem is that code is quite a big bunch of text and we can write it in shorter way. But for this we need some additional scalaz magic: Monad[Future] and monad transformer.

Let`s define `Monad[Future]`:

```scala
  //define Monad[Future] for work with future`s monad transformer
  implicit val FutureMonad = new Monad[Future] {
    def point[A](a: => A): Future[A] = Future(a)
    def bind[A, B](fa: Future[A])(f: (A) => Future[B]): Future[B] = fa flatMap f
  }
```

And now we can replace two blocks of `for-comprehension` by one nice block:

```scala
val result = for {
  result1 <- eitherT(futureResult1)
  result2 <- eitherT(futureResult2)
  result3 <- eitherT(futureResult3)
} yield {
  // if all operations complete successfully we will concatenate string
  result1.value + " " + result2.value + " " + result3.value
}
//return final result
result.run
```

If we run it like this:

```scala
//print result of calculation
val futResult = doSuccessfulComplexAction().map{
  case ActionSuccess(value) => println("success value: " + value)
  case ActionFailure(error) => println("err value: " + error)
}

//Wait 20 seconds for result
Await.result(futResult, 20.seconds)
```

We will see that in console:

```
success-action#1 is completed
success-action#2 is completed
success-action#3 is completed
success value: success#first success#second success#third
```

## Error handling

Let's add a failed action in order to show how the code above behaves itself in case of return of `ActionFailure`:

```scala
def doShortFailedAction(): FutureActionResult[Outcome] = {
  Future {
    println("short failed action is completed")
    ActionFailure("failed#short")
  }
}
```

Add a call of `doShortFailedAction()` in `for-comprehension`:

```scala
//...
val failedAction  = doShortFailedAction()
//...
val result = for {
  result1 <- eitherT(futureResult1)
  failed  <- eitherT(failedAction) //<-- this is fail
  result3 <- eitherT(futureResult3)
} yield {
  // if all operations complete successfully we will concatenate string
  result1.value + " " + failed.value + " " + result3.value
}
```

If we execute this code we will see this in console:

```
short failed action is completed
success-action#1 is completed
err value: failed#short
```

As you can see this code doesn't wait until all of futureResult-s will be available because in any case there will be `ActionFailure`. 

But there is still a problem. If we swap lines like that:

```scala
val result = for {
  result1 <- eitherT(futureResult1)
  result3 <- eitherT(futureResult3)
  failed  <- eitherT(failedAction)
} yield {
  //...
}
```

We will see that we have to wait completion of all the futures despite `failedAction` always finishes in first. In general there is no way to do using common scala. But after some magic tricks it will be possible ^_^ This kind of scala magic will be discussed in the next chapter.

## Links
* [EitherFutureSimpleExample.scala](https://github.com/coffius/koffio-examples/blob/master/src/main/scala/io/koff/examples/async_services/EitherFutureSimpleExample.scala) - an example app for this article
* [coffius/koffio-examples](https://github.com/coffius/koffio-examples "Sources of the example") - Repository with all the examples
* [scalaz/scalaz](https://github.com/scalaz/scalaz "Scalaz on GitHub") - sources of scalaz. You can also find [here](https://github.com/scalaz/scalaz/tree/series/7.2.x/example/src/main/scala/scalaz/example "Simple scalaz examples") simple examples how to use different parts of scalaz
* [Learning scalaz](http://eed3si9n.com/learning-scalaz/ "Learning scalaz") - a really big source of information about most of parts of the library with shotr examples

## UPD
* (14/08/2015) Get rid of `Function[Future]` in the article and in the code example because it's unnecessary
