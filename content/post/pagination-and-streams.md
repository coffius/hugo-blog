+++
date = "2016-03-30T22:00:00+03:00"
draft = true
title = "Pagination and Streams"
slug = "pagination-and-streams"
+++

In this article we will see how to use different streams(like akka-stream) for pagination and when it can be useful. 
The main idea of pagination is partition of a big sequence of objects into several parts(or pages) in order to make possible its processing page by page. 
For example, you have a 1.000.000 users in the database and you need to send an email to all of them. 
You could try to load all user records in a big list and process it at once but it would not be memory-efficient approach. 
Instead you can partition the list of users into pages by 100 users per page, load one page, send emails to users in this page, load next page and so on. 
This will be a much more efficient way to deal with big collections of records.

So lets try to implement this approach but for a more complex case.

<!--more-->

Lets say that we have the big collection of users and each user has the big collection of products and we need to send emails to each user about each his product. 
In this case we can not load all users and/or all user's products in memory at once. So we have to use pagination for users and for user's products.

## Possible solutions

To solve that issue I tried to use:

* Recursion
* Akka-stream
* Twitter AsyncStream

All examples can be found [here](https://github.com/coffius/koffio-pagination "GitHub: Example project")

## Using recursion

The main idea is to use the recursive helper method `forEachPage(...)` which executes page functions `forUserPage(...)` and `forWarePage(...)` until they return false. 
It is a simple approach but it requires plenty of code. 

```scala
import io.koff.pagination.domain.User
import io.koff.pagination.repo.Repository
import scala.concurrent.duration._

import scala.concurrent.{Await, ExecutionContext, Future}
import scala.language.postfixOps

/**
 * Example of recursive approach
 */
object RecursiveMain {
  import scala.concurrent.ExecutionContext.Implicits.global
  private val WaitTime  = 10 seconds

  val repo = new Repository
  val emailService = new EmailService

  def main(args: Array[String]) {
    val result = forEachPage()(forUserPage)
    Await.result(result, WaitTime)
  }

  /**
   * Execute this function for each page of user wares
   */
  private def forWarePage(user: User, page: PageRequest): Future[Boolean] = {
    for {
      //get page of user wares
      wares <- repo.getPageOfWares(user, page)
      //send email
      futSeq = wares.map { ware => emailService.sendEmail(user, ware) }
      _ <- Future.sequence(futSeq)
    } yield {
      wares.nonEmpty
    }
  }

  /**
   * Execute this function for each page of users
   */
  private def forUserPage(page: PageRequest): Future[Boolean] = {
    for{
      //get a page of users
      users <- repo.getPageOfUsers(page)
      //traverse though user wares for each user
      futSeq = users.map(user => forEachPage()(forWarePage(user, _)))
      _ <- Future.sequence(futSeq)
    } yield {
      users.nonEmpty
    }
  }

  /**
   * Recursive function for traverse through a collection page by page using `func`
   * @param currPage current page
   * @param func function which is called on each step of recursion
   * @param ctx execution context
   * @return result future
   */
  private def forEachPage(currPage: PageRequest = PageRequest.FirstPage)
                         (func: PageRequest => Future[Boolean])
                         (implicit ctx: ExecutionContext): Future[_] = {
    func(currPage).flatMap {
      case true =>
        forEachPage(currPage.next)(func)
      case false =>
        Future.successful(())
    }
  }
}
```

## Using akka-streams
It is easy to implement the same logic using the method `Source.unfoldAsync(...)` from akka-stream. But it is necessary to have ActorSystem in order to use akka streams.

```scala
import akka.actor.ActorSystem
import akka.stream.ActorMaterializer
import akka.stream.scaladsl.Source
import io.koff.pagination.repo.Repository

import scala.concurrent.duration._
import scala.concurrent.{Await, Future}
import scala.language.postfixOps

/**
 * Pagination using akka-streams
 */
object ReactiveMain {
  import scala.concurrent.ExecutionContext.Implicits.global
  private val WaitTime  = 10 seconds
  private val repo = new Repository
  private val emailService = new EmailService 
  implicit val system = ActorSystem("reactive-pagination")
  implicit val materializer = ActorMaterializer()
  def main(args: Array[String]): Unit = {
    //Define the computation stream
    val source =
      //get users page by page
      asyncPageToSource(repo.getPageOfUsers)
      .flatMapConcat {
        //for each user get wares page by page
        user => asyncPageToSource(repo.getPageOfWares(user, _)).map((user, _))
      }.takeWhile{
        //it is possible to check errors here to stop processing after the first error
        case (user, _) => user.name != "user#5"
      }.mapAsync(1){
        //send email for each (user, ware) pair
        (emailService.sendEmail _).tupled
      }

    //Execute computations
    //Because all computations are executed inside the stream
    //We don't need to do anything here
    val result = source.runForeach(value => value)
    //Wait the result
    Await.result(result, WaitTime)
    //Shutdown the actor system
    system.terminate()
  }

  /**
   * Converts page function to Source of pages
   * @param pageFunc function which receives PageRequest as a parameter and returns an async result
   * @return [[Source]] of pages
   */
  private def asyncPageToPageSource[T](pageFunc: PageRequest => Future[Seq[T]]): Source[Seq[T], Unit] = {
    Source.unfoldAsync(PageRequest.FirstPage){ page =>
      val pageData = pageFunc(page)
      pageData.map(data => if (data.isEmpty){ None } else { Some(page.next, data) })
    }
  }

  /**
   * Converts page function to Source of elements of T
   * @param pageFunc function which receives PageRequest as a parameter and returns an async result
   * @return [[Source]] of pages
   */
  private def asyncPageToSource[T](pageFunc: PageRequest => Future[Seq[T]]): Source[T, Unit] = {
    asyncPageToPageSource(pageFunc).mapConcat{ seq => seq.toList }
  }
}
```

## Using twitter AsyncStream
It is possible to implement pagination using twitter AsyncStream in almost the same manner as it is made using akka-stream but with one note. AsyncStream does not have `unfoldAsync(...)` so we need to implement it: `unfold2(...)`.

```scala
import com.twitter.concurrent.exp.AsyncStream
import com.twitter.util.{Await, Future}
import io.koff.pagination.repo.Repository
import io.koff.pagination.utils.TwitterScalaFuture._

/**
 * Example for twitter AsyncStream
 */
object TwitterMain {
  import scala.concurrent.ExecutionContext.Implicits.global

  private val repo = new Repository
  private val emailService = new EmailService

  def main(args: Array[String]) {
    //Define the computation stream
    val stream =
      //get users page by page
      //also convert scala future to twitter using '.toTwitter'
      //see implicit class TwitterScalaFuture.ScalaToTwitter for details
      asyncPageToStream(repo.getPageOfUsers(_).toTwitter)
      .flatMap{
        //for each user get wares page by page
        user => asyncPageToStream(repo.getPageOfWares(user, _).toTwitter).map((user, _))
      }.takeWhile {
        //it is possible to check errors here to stop processing after the first error
        case (user, _) => user.name != "user#5"
      }.mapF {
        //send email for each (user, ware) pair
        case (user, ware) => emailService.sendEmail(user, ware).toTwitter
      }

    //Execute computations
    //Because all computations are executed inside the stream
    //We don't need to do anything here
    val res = stream.foreach { value => value }

    //Wait the result
    Await.ready(res)
  }

  /**
   * Almost the same as akka-stream Source.unfoldAsync(...)
   * This function unfolds a value to a sequence of elements
   */
  def unfold2[T, K](value: T)(func: (T) => Future[Option[(T, K)]]): AsyncStream[K] = {
    val result = func(value)
    val stream = AsyncStream.fromFuture(result)
    stream.flatMap{
      case Some((t,k)) => stream ++ unfold(t)(func)
      case None => AsyncStream.empty
    }.map(_.get._2)
  }

  def unfold[T, K](zero: T)(value: (T) => Future[Option[(T, K)]]): AsyncStream[Option[(T, K)]] = {
    val result = value(zero)
    val stream = AsyncStream.fromFuture(result)
    stream.flatMap {
      case Some((t,_)) => stream ++ unfold(t)(value)
      case None => AsyncStream.empty
    }
  }

  /**
   * Converts page function to Source of pages
   * @param pageFunc function which receives PageRequest as a parameter and returns an async result
   * @return [[AsyncStream]] of pages
   */
  def asyncPageToPageStream[T](pageFunc: PageRequest => Future[Seq[T]]): AsyncStream[Seq[T]] = {
    unfold2[PageRequest, Seq[T]](PageRequest.FirstPage){ page =>
      val pageData = pageFunc(page)
      pageData.map { data => if (data.isEmpty) { None } else { Some(page.next, data) } }
    }
  }

  /**
   * Converts page function to Source of elements of T
   * @param pageFunc function which receives PageRequest as a parameter and returns an async result
   * @return [[AsyncStream]] of pages
   */
  def asyncPageToStream[T](pageFunc: PageRequest => Future[Seq[T]]): AsyncStream[T] = {
    asyncPageToPageStream(pageFunc).flatMap(AsyncStream.fromSeq)
  }
}
```

## Links

* [Example project](https://github.com/coffius/koffio-pagination "GitHub: Example project") - sources for this article
* [Akka Stream](http://doc.akka.io/docs/akka/2.4.3/scala/stream/stream-introduction.html "Akka docs") - documentation for akka-stream
* [Twitter AsyncStream](https://finagle.github.io/blog/2016/02/15/asyncstream "Hello AsyncStream!") - an article about AsyncStream
