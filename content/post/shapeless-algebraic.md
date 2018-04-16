+++
date = "2018-04-16T10:00:00+03:00"
draft = false
title = "Sum Data Types with Shapeless"
slug = "shapeless-sums"
+++
Scala has support of algebraic data types out of the box but often it is not enough for complex cases.
In this article I will try to show its limits and how to bypass them using `shapeless` library by the example of `sum data types`.
## Contents

* [What is ADT?]({{<relref "shapeless-algebraic.md#what-is-adt">}})
* [Sealed trait as a sum type]({{<relref "shapeless-algebraic.md#sealed-trait-as-a-sum-type">}})
* [Extending ADTs using Wrapper + Evidence]({{<relref "shapeless-algebraic.md#extending-adts-using-wrapper-evidence">}})
* [Coproduct as a sum type]({{<relref "shapeless-algebraic.md#coproduct-as-a-sum-type">}})
* [Poly1 as pattern matching]({{<relref "shapeless-algebraic.md#poly1-as-pattern-matching">}})
* [Merging sum types with Shapeless]({{<relref "shapeless-algebraic.md#merging-sum-types-with-shapeless">}})

<!--more-->

## What is ADT?

ADT - is acronym for `Algebraic Data Type` which is a type that is composed from other types by using operations like "sum" or "product", where:

* "sum" is [the tagged union](https://en.wikipedia.org/wiki/Tagged_union). `sealed trait` is a common way to describe it in Scala:

```scala
sealed trait Option[T]
case class  Some[T](value: T) extends Option[T]
case object None extends Option[Nothing]
```

* ["product"](https://en.wikipedia.org/wiki/Product_type) is a combination of types. In Scala it can be described using tuples or case classes:

```scala
type Product = (Int, String, Long, Seq[Int])
case class Product(i: Int, s: String, l: Long, seq: Seq[Int])
```

## Sealed trait as a sum type
In scala a sum time can be represented as a sealed trait:

```scala
sealed trait Request
case class CreateUser(name: String)                   extends Request
case class ReadUserInfo(userId: Int)                  extends Request
case class UpdateUserInfo(userId: Int, name: String)  extends Request
case class DeleteUser(userId: Int)                    extends Request
```

This means that a value of `request: Request` can be equal only to an instance of one of classes mentioned above.
And the compile can check exhaustiveness of pattern matching for such a type.

```scala
// the compiler throws a warning/error if there is no `case ... => ...` for all possible subtypes:
// Error: match may not be exhaustive.
def exhaustivePatternMatch(request: Request) = {
  request match {
    case CreateUser(name)         => ???
    case ReadUserInfo(id)         => ???
    case UpdateUserInfo(id, name) => ???
    case DeleteUser(id)           => ???
  }
}
```

## Extending ADTs using Wrapper + Evidence
This approach works fine for simple cases but it fails in case if it is necessary to extend a predefined type which we can't control.
For example, `sealed trait Request` is defined in a library but we want to add new types of requests to the protocol:

```scala
sealed trait AdvancedRequest
case object  GetAllUsers     extends AdvancedRequest
case object  DeleteAllUsers  extends AdvancedRequest
```

In this case we have to introduce additional abstractions to make it possible:

```scala
case class Wrapper[T: Evidence](value: T)
object Wrapper {
  sealed trait Evidence[-T]
  implicit object RequestEvidence extends Evidence[Request]
  implicit object AdvancedRequestEvidence extends Evidence[AdvancedRequest]
}

// can be compiled without problems
val withBasicRequest = Wrapper(CreateUser("test-user"))
val withAdvancedRequest = Wrapper(GetAllUsers)

// throw an error:
// Error: could not find implicit value for evidence parameter of type Evidence[DeletedUsers]
val withInvalidValue = Wrapper(DeletedUsers(Seq(1, 2)))
```

Using Wrapper + Evidence we can't make a mistake and wrap a value of an invalid type - the compiler will find a problem and throw the error.
But we can make a mistake in pattern matching and the compiler won't help us:
```scala
private def patternMatch[T](wrapper: Wrapper[T]) = {
  // compiles with several matches missed - no warns or errors :(
  wrapper.value match {
    case CreateUser(name) => ???
    case ReadUserInfo(id) => ???
    //UpdateUserInfo, DeleteUser, GetAllUsers and DeleteAllUsers are missed here
  }
}
```

In order to deal with this problem we can use `Coproduct` types and `Poly1` functions from `shapeless` library.

## Coproduct as a sum type

`Coproduct` is a way of representing sum types in `shapeless` library. It can be used like this:

```scala
type Shape = Rectangle :+: Circle :+: Triangle :+: CNil
```

It means that `value: Shape` can be either `Rectangle` or `Circle` or `Triangle`.
More information about how to work with Coproduct in `shapeless` is available [here](https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0#coproducts-and-discriminated-unions).

`Coproduct` in contrast to `sealed trait` makes possible to combine an arbitrary set of predefined types in a sum type.
For example, we can use it to recombine types from above like this:

```scala
type ReadRequests   = ReadUserInfo :+: GetAllUsers.type :+: CNil
type WriteRequests  = CreateUser :+: UpdateUserInfo :+: DeleteUser :+: DeleteAllUsers.type :+: CNil
```

## Poly1 as pattern matching

Having our sum types defined we can implement pattern matching for them. For that we will use `Poly1` function.

```scala
object read extends Poly1 {
  implicit val readUserInfo = at[ReadUserInfo]    (_ => true)
  implicit val getAllUsers  = at[GetAllUsers.type](_ => 1)
}

object write extends Poly1 {
  implicit val createUser     = at[CreateUser]          (_ => true)
  implicit val updateUserInfo = at[UpdateUserInfo]      (_ => 1)
  implicit val deleteUser     = at[DeleteUser]          (_ => "2")
  implicit val deleteAllUsers = at[DeleteAllUsers.type] (_ => List(3))
}
```
`object function extends Poly1 {...}` is a definition of [a polymorphic function](https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0#polymorphic-function-values) - a function that is defined for an arbitrary set of types. There are a lot of use cases for such functions but we are interested in the fact that these functions can work as exhaustive pattern matching being applied to `Coproduct` values.

In the example above we can see that `read` function is defined for `ReadUserInfo` and `GetAllUsers` requests, `write` - for `CreateUser`, `UpdateUserInfo`, `DeleteUser` and `DeleteAllUsers`. Inasmuch as `read` is defined for all types in `ReadRequests` and `write` for all types in `WriteRequests` we can use it for pattern matching of values of these types:

```scala
private def readPatternMatch(request: ReadRequests): Unit = {
  val result = request.map(read)
}

private def writePatternMatch(request: WriteRequests): Unit = {
  val result = request.map(write)
}
```

Check of exhaustiveness of such functions during compilation is implemented in `shapeless`. The compiler will throw an error if a poly function does not have handlers for all types in `Coproduct`.

```scala
object read extends Poly1 {
  implicit val readUserInfo = at[ReadUserInfo](_ => true)
// Assume we forgot to add a handler for `GetAllUsers` request
// implicit val getAllUsers  = at[GetAllUsers.type](_ => 1)
}

private def readPatternMatch(request: ReadRequests): Unit = {
  val result = request.map(read)
}
```

The code above will generate:
```text
Error: could not find implicit value for parameter mapper: shapeless.ops.coproduct.Mapper[io.koff.shapeless_algebraic.UsingPolys.read.type,io.koff.shapeless_algebraic.UsingPolys.ReadRequests]
    val result = request.map(read)
```
Though this error does not provide information about a specific type that does not have a handler, it makes impossible to omit it without notice.

## Merging sum types with Shapeless
Using `shapeless` you get another cool feature - ability to join sum types together keeping compile time checks.

```scala
// joined coproducts
val joined = Adjoin[WriteRequests :+: ReadRequests]
type AllRequests = joined.Out
```

Such a definition of `AllRequests` is an equivalent to:

```scala
type AllRequests2 = CreateUser :+: UpdateUserInfo :+: DeleteUser :+: DeleteAllUsers.type :+: ReadUserInfo :+: GetAllUsers.type :+: CNil

val value1: AllRequests2 = Coproduct[AllRequests] (GetAllUsers)
val value2: AllRequests  = Coproduct[AllRequests2](DeleteAllUsers)
```

The next step is to join different sealed traits together. In order to do it we need `shapeless.Generic`.

```scala
/* It is possible to extract Coproducts from sealed traits */
val genRequest    = Generic[Request]
val genAdvRequest = Generic[AdvancedRequest]
val joinedTraits  = Adjoin [genRequest.Repr :+: genAdvRequest.Repr]
/* And join them together */
type JoinedTraits = joinedTraits.Out
```

It is also possible to "merge" poly functions if they are defined as traits:

```scala
trait read { self: Poly1 =>
  implicit val readUserInfo = at[ReadUserInfo]    (_ => true)
  implicit val getAllUsers  = at[GetAllUsers.type](_ => 1)
}

trait write { self: Poly1 =>
  implicit val createUser     = at[CreateUser]          (_ => 2.0F)
  implicit val updateUserInfo = at[UpdateUserInfo]      (_ => 3.0D)
  implicit val deleteUser     = at[DeleteUser]          (_ => "4")
  implicit val deleteAllUsers = at[DeleteAllUsers.type] (_ => List(5))
}

//Ad Hoc definition of a merged poly function
object polyFunc extends Poly1 with read with write

private def patternMatchForAllRequests(all: AllRequests) = {
  all.map(polyFunc)
}
```

## Conclusion

So it is possible to use `shapeless.Coproduct` to define sum types on the analogy of sealed traits.
But when to use what?

Use sealed traits for:

1. simple well-defined types like `Option[_]`
1. sum types you control
1. defining basic pieces that might be composed later(using `Generic`)

Use `shapeless` when you need:

1. to create a sum type from arbitrary set of types - `Coproduct`
1. to compose and use sealed traits you don't control - `Generic`
1. to have compile time checks for exhaustiveness of pattern-matching for composed sum types - `Poly` functions

## Links

* [Code for this article](https://github.com/coffius/shapeless-algebraic)
* [Feature overview for shapeless](https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0)
* [The Type Astronaut’s Guide](https://underscore.io/books/shapeless-guide/) - a free book about `shapeless`
