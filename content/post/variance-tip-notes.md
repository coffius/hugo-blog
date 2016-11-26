+++
date = "2016-11-26T22:51:18+03:00"
draft = false
title = "Tips about variance in scala"
slug = "variance-tip-notes"
+++

Type variance in scala is quite a tricky topic especially if you do not use it often - details might slip out of mind easily in this case. So below you can find very short tips about it which purpose is to remind how it works.

<!--more-->
All code for examples below can be found [here](https://github.com/coffius/koffio-examples/tree/master/src/main/scala/io/koff/examples/variance).

## Java problem
In Java all generic types are covariant and because of it we can write this non-working code:

[CovarianceProblem.java](https://github.com/coffius/koffio-examples/blob/master/src/main/scala/io/koff/examples/variance/CovarianceProblem.java)

```scala
/**
 * Example of a problem with covariant types in Java
 */
public class CovarianceProblem {
    public static void main(String[] args) {
        String[] strArray = { "str#1", "str#2" };
        Object[] objArray = strArray;
        objArray[0] = 1; //throws ArrayStoreException: java.lang.Integer
    }
}
```

## Why do we need it?

In order to deal with the problem above we need type variance to define and distinguish different relations between generic classes which specific types have inherited relationships between them.

There are three possible options:

* Invariant
* Covariant
* Contravariant

## Examples

Let's use this simple class tree for examples(in [Variance.scala](https://github.com/coffius/koffio-examples/blob/master/src/main/scala/io/koff/examples/variance/Variance.scala)):


```scala
class GrandParent
class Parent extends GrandParent
class Child1 extends Parent
class Child2 extends Parent
```

### Invariant
Invariant means that there is no relation between `InVariant[Parent]` and `InVariant[Child1]` or `InVariant[Child2]`

```scala
val parentInvariant = new InVariant[Parent]
// Next lines cannot be compiled because of invariant
//val child1Invariant: InVariant[Child1] = parentInvariant
//val child2Invariant: InVariant[Child2] = parentInvariant
```

### Covariant
Covariant means that `CoVariant[Child1]` and `CoVariant[Child2]` are subclasses of `CoVariant[Parent]`

```scala
val child1Covariant = new CoVariant[Child1]
val child2Covariant = new CoVariant[Child2]
//We can use values of CoVariant[Child1|Child2] as CoVariant[Parent]
val parentCovariant1: CoVariant[Parent] = child1Covariant
val parentCovariant2: CoVariant[Parent] = child2Covariant
```

Example of correct usage:



```scala
//we can use a covariant type as
class Producer[+A](val value: A) {          // a type for immutable values
  private[this] var variable: A = ???       // a type for private mutable variables
  def simpleProduce(): A = ???              // a type for method outputs
  def complexProduce[B >: A](b: B): A = ??? // a lower type bound
}
```

Example of incorrect usage:

```scala
//code below cannot be compiled
//we can not use a covariant type as
class Producer[+A](var variable: A) { // a type for public mutable variables
  def consume(a: A): Unit = ???       // a type for method parameters
}
```

### Contravariant
Contravariant means that `ContraVariant[Parent]` is a subclass of `ContraVariant[Child2]` and `ContraVariant[Child2]`

```scala
val parentContravariant = new ContraVariant[Parent]
//Looks awkward but it is totally legit :)
val child1Contravariant: ContraVariant[Child1] = parentContravariant
val child2Contravariant: ContraVariant[Child2] = parentContravariant
```

Example of correct usage:

```scala
//we can use a contravariant type as
class Consumer[-A]() {
  private[this] var variable: A = ???   // a type for private mutable variables
  def consume(a: A): Unit = ???         // a type for method parameters
  def complex[B <: A](b: B): Unit = ??? // a upper type bound
}
```

Example of incorrect usage:

```scala
//code below cannot be compiled
//we can not use a contravariant type as
class Producer[-A](val value: A, var variable: A) { // a type for public immutable values and publis mutable variables
  def produce(): A = ???                            // a type for for method outputs
  def complexProduce[B >: A](b: B): A = ???         // a lower type bound
}
```

### How to use Contravariance

Code for this part is [here](https://github.com/coffius/koffio-examples/blob/master/src/main/scala/io/koff/examples/variance/example/HandlingEventsExample.scala).

Contravariance probably is a bit counterintuitive thing but the next example should help to understand when it is possible to use it.
Let's imagine that we want to implement our own type-safe event system with a handler register and event handlers.
The classic thing but with one requirement - we want to be able to handle events in a generalised way. More details below.

So we have `Event` trait as a base trait for all possible events, `UpdateEvent` - for events about update operations and several more specific traits and classes(in `Event.scala`)

```scala
/**
  * The very basic trait to describe an event
  */
trait Event
/**
  * The marker trait for events for data updates
  */
trait UpdateEvent extends Event{
  /** Id of the updated entity */
  def entityId: UUID
}
/**
  * The marker trait for events about wares
  */
trait WareEvent extends Event {
  /**Id the ware which is a reason of the event */
  def wareId: UUID
}
/**
  * The marker trait for events about users
  */
trait UserEvent extends Event {
  /** Email of the user who is a reason of the event */
  def email: String
}
/*
 * Specific events
 */
case class ChangeWarePriceEvent(wareId: UUID,
                                oldPrice: BigDecimal,
                                newPrice: BigDecimal) extends WareEvent with UpdateEvent {
  override def entityId: UUID = wareId
}

case class ChangeUserPhoneEvent(userId: UUID, userEmail: String, oldPhone: String, newPhone: String) 
	extends UserEvent 
	with UpdateEvent 
{
  override def email: String = userEmail
  /* Let's say that a user email is an unique identifier of a user */
  override def entityId: UUID = userId
}
```

Next thing is handling events. And as it was said before we want to handle events in a generalised way which means next:

* We have a generalised event handler `ConsoleLogEventHandler` - can handle all possible types of events

```scala
/**
  * Logs all events to stdout using .toString
  */
class ConsoleLogEventHandler extends EventHandler[Event]{
  override def handle(event: Event): Unit = {
    println(s"logger - event has been received: [$event]")
  }
}
```

* We have specialised handlers like `DiscountEventHandler` or `CheckNumberEventHandler` - can handle just one type `ChangeWarePriceEvent` or `ChangeUserPhoneEvent`

```scala
/**
  * Notifies users about discounts (the new price of a ware is less than the old one)
  */
class DiscountEventHandler extends EventHandler[ChangeWarePriceEvent]{
  override def handle(event: ChangeWarePriceEvent): Unit = {
    if(event.oldPrice > event.newPrice) {
      val discount = 1 - event.newPrice / event.oldPrice
      println(s"there is a discount[${discount * 100} %] for the ware[id: ${event.wareId}]")
    }
  }
}
/**
  * Sends SMS to the new phone number if a user change it.
  */
class CheckNumberEventHandler extends EventHandler[ChangeUserPhoneEvent]{
  override def handle(event: ChangeUserPhoneEvent): Unit = {
    println(s"SMS has been sent to the phone:${event.newPhone}")
  }
}
```
* We want to add several handlers for an event type and make it type-safely:

```scala
// ok
handlerRegister.addHandler(classOf[ChangeUserPhoneEvent], ConsoleLogEventHandler)
handlerRegister.addHandler(classOf[ChangeWarePriceEvent], ConsoleLogEventHandler)
handlerRegister.addHandler(classOf[ChangeUserPhoneEvent], CheckNumberEventHandler)
handlerRegister.addHandler(classOf[ChangeWarePriceEvent], DiscountEventHandler)

// generates compilation errors
handlerRegister.addHandler(classOf[ChangeUserPhoneEvent], DiscountEventHandler)
```
In order to make it possible we should use a contravariant event handler:

```scala
/**
  * Basic trait for an event handler
  */
trait EventHandler[-T <: Event] {
  /**
    * Handle a particular event
    * @param event an event to handle
    */
  def handle(event: T): Unit
}
```

And define `addHandler(...)` as:

```scala
/**
  * Adds a handler for a particular type of events
  * @param eventType the class of events handled by the eventHandler
  * @param eventHandler the event handler which should handle all events of the specific type(eventType)
  * @tparam E type of the event we want to handle
  * @tparam T intermediate type.
  *           Means that we want to handle an event of type `E` as `T`.
  *           And `T` should be between Event and `E` in the chain of inheritance.
  *           If we have such types: `Event <- SubEventType <- SubSubEventType <- ... <- E`
  *           then `T` can be one of these types
  * @tparam H type of the handler which should handle all events of type `E`
  */
def addHandler[E <: Event, T >: E <: Event, H <: EventHandler[T]](eventType: Class[E], eventHandler: H): Unit
```
This is possible only if we use contravariance for EventHandler. If we define it like invariant(`EventHandler[T <: Event]`) and use generalised handlers then we will have errors like that:

```scala
Error:(12, 19) inferred type arguments [ChangeUserPhoneEvent, ChangeUserPhoneEvent, ConsoleLogEventHandler.type] do not conform to method addHandler's type parameter bounds [E <: Event, T >: E <: Event, H <: EventHandler[T]]
  handlerRegister.addHandler(classOf[ChangeUserPhoneEvent], ConsoleLogEventHandler)
  
Error:(12, 37) type mismatch;
 found   : Class[ChangeUserPhoneEvent](classOf[ChangeUserPhoneEvent])
 required: Class[E]
  handlerRegister.addHandler(classOf[ChangeUserPhoneEvent], ConsoleLogEventHandler)
  
Error:(12, 61) type mismatch;
 found   : ConsoleLogEventHandler.type
 required: H
  handlerRegister.addHandler(classOf[ChangeUserPhoneEvent], ConsoleLogEventHandler)
```

And of course it is also impossible to use covariance in this case.

## Links

* [Official tutorial for variance](http://docs.scala-lang.org/tutorials/tour/variances.html)
* [Examples for this article](https://github.com/coffius/koffio-examples/tree/master/src/main/scala/io/koff/examples/variance)
* [About variance in Scala school from Twitter](https://twitter.github.io/scala_school/type-basics.html#variance)
