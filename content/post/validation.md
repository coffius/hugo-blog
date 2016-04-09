+++
date = "2015-11-27T00:11:00+03:00"
draft = false
title = "Validation in scala"
slug = "292173-validation-in-scala"
+++

This article is about possible solutions for validation in scala. Validation is the process of checking input data in order to provide its correctness and requirements compliance. 

# Implementations
There are several libraries in scala which can be used for validation:

* **[Accord](https://github.com/wix/accord "GitHub: Accord")** - Accord is a validation library written in and for Scala. [Docs here](http://wix.github.io/accord/ "Accord: Docs")
* **Skinny validator** - skinny-validator is a portable library, so it is possible to use skinny-validator with Play2, Scalatra and any other web app frameworks. [Docs here](http://skinny-framework.org/documentation/validator.html "Skinny Validator: Docs")
* **[DValidation](https://github.com/tobnee/DValidation/ "GitHub: DValidation")** - A little, opinionated Scala domain object validation toolkit
* **[io.underscore.validation](https://github.com/davegurnell/validation "GitHub: io.underscore.validation")** - Work-in-progress library demonstrating a functional programming approach to data validation in Scala

The source code for this article is **[here](https://github.com/coffius/koffio-validation)**

<!--more-->
## General

Let's define domain classes which should be validated with the libraries above.

```scala
//Phone.scala
//Phone.scala
package io.koff.validation.domain

/**
 * User phone
 */
case class Phone(value: String)
```

```scala
//Address.scala
package io.koff.validation.domain

/**
 * User address
 * @param street street
 * @param house house number
 */
case class Address(street: String, house: Int)
```

```scala
//User.scala
package io.koff.validation.domain

/**
 * User - the main example domain class for validation
 * @param login user login
 * @param email user email
 * @param password user password
 * @param age user age
 * @param phone user phone
 * @param addresses list of user addresses - for showing a sequence validation
 * @param userInfo recursive data for showing a recursive validation
 */
case class User(login: String,
                email: String,
                password: String,
                age: Int,
                phone: Option[Phone],
                addresses: Seq[Address],
                userInfo: InfoNode[String] = InfoNode.default)
```

After we defined our domain we can implement validation rules for it using the listed libraries.

## Accord

**Accord** is a standalone scala library which is developed and used in [WiX](http://www.wix.com/ "WiX site").

The full example of using **Accord** can be found [here](https://github.com/coffius/koffio-validation/blob/master/src/main/scala/io/koff/validation/AccordExample.scala)

First what you need to do is to define validation rules using `validator(...)` method:

```scala
import io.koff.validation.domain.{Address, User, Phone}
import com.wix.accord._
import dsl._

/**
 * Define Accord validator for Address class
 */
implicit val addressValidator = validator[Address] { address =>
  address.street is notEmpty
  address.house is >(0)
}

/**
 * Define Accord validator for Phone class
 */
implicit val phoneValidator = validator[Phone]{ phone =>
  phone.value is notEmpty and startWith("+7")
}

/**
 * Define Accord validator for User class
 */
implicit val userValidator = validator[User] { user =>
  //It is not possible to make a negative predicate
  //for example there is no way to forbid using "admin" in user name except for creation of your own validator
  user.login is notEmpty and startWith("super_") and endWith("!")

  //just as a sample :)
  //don`t user regex to validate emails: 
  // - http://davidcel.is/posts/stop-validating-email-addresses-with-regex/
  //otherwise your users can face with problems using their emails on your service
  user.email is matchRegex("""\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z""")

  // all the definitions below are equal
  user.password.length is between(6, 12)
  user.password have size >= 6
  user.password has size <= 12

  //you can validate number with range
  user.age.must(between(13, 99))

  //can use >, <, >=, <= etc
  user.age must <=(99)
  user.age must >=(13)
  //can`t write it like this
  //user.age must >= 13
  //it will be a compilation error

  //check if a number field is equal or not equal to some value
  //user.age must equalTo(1000)
  //user.age must notEqualTo(1)

  //checks if an option field is defined with Some(value)
  //user.phone is notEmpty

  //make sure that if the phone field is defined then it is defined with correct value
  user.phone.each is valid

  user.addresses should notEmpty
  user.addresses.each is valid
}
```

Then you can check an object using `validate[T](...)` method. The validator for `T` must be accessed as an implicit value.

```scala
def main(args: Array[String]) {
  val correctUser = User(
    login = "super_user_!",
    email = "example@example.com",
    password = "1234567",
    age = 14,
    phone = Some(Phone("+78889993322")),
    addresses = Seq(Address("Baker st.", 221))
  )

  println("correct result: " + validate(correctUser))
  //prints: 'correct result: Success'

  val withWrongPhone = correctUser.copy(phone = Some(Phone("8889993322")))
  println("withWrongPhone: " + validate(withWrongPhone))
  //prints: 'withWrongPhone: Failure(Set(GroupViolation(Phone(8889993322),is invalid,Some(phone),Set(RuleViolation(8889993322,must start with '+7',Some(value))))))'
  
  //More examples are in the code....
}
```

In summary this library is quite good but there is one issue: an error message is represented as a sentence with a lexical description(`must start with '+7'`) instead of an error code like `User.phone.invalid`. It can be a problem when you want to localize error messages.

## Skinny validator

Skinny validator is a part of **[Skinny Framework](http://skinny-framework.org/ "Skinny Framework")** - a full-stack web app framework built on Skinny Micro. 

The full example of using **Skinny validator** can be found [here](https://github.com/coffius/koffio-validation/blob/master/src/main/scala/io/koff/validation/SkinnyExample.scala)

At first let's define a new trait `EntityValidator` which helps organising validations rules for domain classes.

```scala
trait EntityValidator[T] {
  def getValidator(entity: T): Validator
}
```

And define the validator for User objects

```scala
implicit val userValidator = new EntityValidator[User] {
  override def getValidator(entity: User): Validator = {
    import skinny.validator.{email => correctEmail}
    import entity._
    Validator(
      param("login" -> login) is notEmpty & startWith("super_") & endWith("!"),
      // skinny.validator.email using regex for checking email
      param("email" -> email) is correctEmail,

      //these definitions are equal
      param("password" -> password) is minMaxLength(6, 12),
      param("password" -> password) is minLength(6),
      param("password" -> password) is maxLength(12),

      //these definitions are equal
      param("age" -> age) is intMinMaxValue(13, 99),
      param("age" -> age) is intMinValue(13), // >=
      param("age" -> age) is intMaxValue(99), // <=
      //this is how to check equation of values
      param("age" -> (age, 14)) is same

      //there is no build-in way to check options(Some|None) and elements of collections
    )
  }
}
```

If we want to add new validation rule we should extend `ValidationRule` trait like this:

```scala
/**
 * For `startWith` we should create custom ValidationRule
 */
case class startWith(value: String) extends ValidationRule {
  def name = "startWith"
  def isValid(v: Any) = Option(v) match {
    case Some(x) => x.toString.startsWith(value)
    case None => true
  }
}

/**
 * For `endWith` we should create custom ValidationRule
 */
case class endWith(value: String) extends ValidationRule {
  def name = "endWith"
  def isValid(v: Any) = Option(v) match {
    case Some(x) => x.toString.endsWith(value)
    case None => true
  }
}
```

Let's add utility method `validate[T](...)` in order to make easier using of skinny-validator.

```scala
def validate[T](entity: T)(implicit entityValidator: EntityValidator[T]): Either[Errors, _] = {
  val validator = entityValidator.getValidator(entity)
  val isOk = validator.validate()
  if(isOk){
    Right(())
  } else {
    Left(validator.errors)
  }
}

def main(args: Array[String]) {
  val correctUser = User(
    login = "super_user_!",
    email = "example@example.com",
    password = "1234567",
    age = 14,
    phone = Some(Phone("+78889993322")),
    addresses = Seq(Address("Baker st.", 221))
  )

  println("correctUser: " + validate(correctUser))
  //prints: 'correctUser: Right(())'

  val allWrong = correctUser.copy(
    login = "not_super_user",
    email = """"Look at all these spaces!"@example.com""", //it is still a valid email address
    password = "short",
    age = 101
  )

  println("correctUser: " + validate(allWrong))
  //prints:
  //Left(
  //  Errors(
  //    Map(
  //      age -> List(
  //        Error(name = intMinMaxValue, messageParams = List(13, 99)),
  //        Error(name = intMaxValue, messageParams = List(99)),
  //        Error(name = same, messageParams = List())
  //      ),
  //      password -> List(
  //        Error(name = minMaxLength, messageParams = List(6, 12)),
  //        Error(name = minLength, messageParams = List(6))
  //      ),
  //      email -> List(Error(name = email, messageParams = List())),
  //      login -> List(Error(name = startWith, messageParams = List()))
  //    )
  //  )
  //)'
}
```

Skinny validator has a basic functionality for validation of simple objects. But if you want to validate more complex structures you have to write your own validation rules. It can be a problem to do it for sequences, options and other generic types.

## DValidation

This library is a small validation toolkit on top of scalaz.

The full example of using **DValidation** can be found [here](https://github.com/coffius/koffio-validation/blob/master/src/main/scala/io/koff/validation/DValidationExample.scala)

In the first place we have to define instances of `DValidator[T]` for our domain classes. The important notice - it is necessary to import scalaz.Order[Int] for `isInRange` and other number checkers.

```scala
val phoneValidator = Validator.template[Phone] { phone =>
  phone.validateWith(
    startWith(phone.value, "+7")
  )
}

val addressValidator = Validator.template[Address] { address =>
  address.validateWith(
    notBlank(address.street) forAttribute 'street,
    address.house.is_>(0) forAttribute 'house
  )
}

val userValidator = Validator.template[User]{ user =>
  user.validateWith(
    notBlank(user.login) forAttribute 'login,
    startWith(user.login, "super_") forAttribute 'login,
    ensure(user.login)("error.dvalidation.end_with")(_.endsWith("!")) forAttribute 'login,
    regex(user.email, """\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z""") forAttribute 'email,

    //these definitions are equal
    isInRange(user.password.length, 6, 12) forAttribute Symbol("User.password.length.between"),
    user.password.length.is_>=(6) forAttribute Symbol("User.password.length.lessThan"),
    user.password.length.is_<=(12) forAttribute Symbol("User.password.length.greaterThan"),

    user.age.is_>=(13) forAttribute 'age,
    user.age.is_<=(99) forAttribute 'age,
    user.age.is_==(14) forAttribute 'age, // or you can use `is_===` for a strict check
    validOpt(user.phone)(phoneValidator),

    hasElements(user.addresses) forAttribute 'addresses
  ).withValidations(
    // You also can user validSequence(...) to validate elements of a collection
    validSequence(user.addresses, addressValidator)
  )
}
```

You can defile own error codes using `scala.Symbol` and `forAttribute(...)` method

Also we need additional validation rules like `startWith`:

```scala
// We need to import scalaz.Order[Int] for `isInRange` and other number checkers
import scalaz.std.AllInstances._
/**
 * Example of a custom validator for DValidation
 * @param toCheck value that should be checked
 * @param startWith value that `toCheck` should start with
 */
def startWith(toCheck: String, startWith: String): DValidation[String] =
  ensure(toCheck)("error.dvalidation.start_with")(a => a.startsWith(startWith))

/**
 * Simple regex checker
 */
def regex(toCheck: String, regexStr: String): DValidation[String] = {
  val regex = regexStr.r
  ensure(toCheck)("error.dvalidation.regex")(a => regex.pattern.matcher(a).find())
}
```

Correctness can be checked in this way:

```scala
def main(args: Array[String]) {
  val correctUser = User(
    login = "super_user_!",
    email = "example@example.com",
    password = "1234567",
    age = 14,
    //it will also work if phone = None
    phone = Some(Phone("+78889993322")),
    addresses = Seq(Address("Baker st.", 221))
  )

  println("correct result: " + userValidator(correctUser))
  //prints: 'correct result:
  // Success(
  //   User(
  //     super_user_!,
  //     example@example.com,
  //     1234567,
  //     14,
  //     Some(Phone(+78889993322)),
  //     List(Address(Baker st.,221)),
  //     InfoNode(1,None,List())
  //   )
  // )
  //'

  val allWrong = correctUser.copy(
    login = "not_super_user",
    email = """"Look at all these spaces!"@example.com""", //it is still a valid email address
    password = "short",
    age = 101,
    phone = Some(Phone("invalid_phone"))
  )

  println("allWrong: " + userValidator(allWrong))
  //prints: 'allWrong:
  // Failure(
  //   DomainError(path: /login, value: not_super_user, msgKey: error.dvalidation.start_with),
  //   DomainError(path: /login, value: not_super_user, msgKey: error.dvalidation.end_with),
  //   DomainError(path: /email, value: "Look at all these spaces!"@example.com, msgKey: error.dvalidation.regex),
  //   DomainError(path: /User.password.length.between, value: 5, msgKey: error.dvalidation.notGreaterThen, args: 6,false),
  //   DomainError(path: /User.password.length.lessThan, value: 5, msgKey: error.dvalidation.notGreaterThen, args: 6,true),
  //   DomainError(path: /age, value: 101, msgKey: error.dvalidation.notSmallerThen, args: 99,true),
  //   DomainError(path: /age, value: 14, msgKey: error.dvalidation.notEqual, args: 101),
  //   DomainError(path: /, value: invalid_phone, msgKey: error.dvalidation.start_with)
  // )
}
```

Although there are not many build-in validators in DValidation, it can be considered as a good option for validation of complex structures if it is ok to have dependency on `scalaz` in your project.

## io.underscore.validation

Despite that this library is named as "Work-in-progress" it has quite a good functionality and it is pretty simple in use.

The full example of using **io.underscore.validation** can be found [here](https://github.com/coffius/koffio-validation/blob/master/src/main/scala/io/koff/validation/UnderscoreValidation.scala)

You can define validators for our domain classes using `validate[T](...)` method and define custom validation methods like `startWith` and `endWith` extending `Validator[T]` trait

```scala
  /**
   * Define custom validators for io.underscore.validation
   */
  def startWith(value: => String, msg: => String): Validator[String] = Validator[String] { in =>
    if(in.startsWith(value)) pass else fail(msg)
  }

  def endWith(value: => String, msg: => String): Validator[String] = Validator[String] { in =>
    if(in.endsWith(value)) pass else fail(msg)
  }

//  implicit val infoNodeValidator = validate[InfoNode]
//    .field(_.index)(gte(1, "InfoNode.index.gte"))

  implicit val phoneValidator = validate[Phone].field(_.value)(startWith("+7", "Phone.value.startWith"))

  implicit val addressValidator = validate[Address]
    .field(_.street)(nonEmpty("Address.street.notEmpty"))
    .field(_.house)(gt(0, "Address.house.gt"))

  implicit val userValidator = validate[User]
    .field(_.login)(nonEmpty  ("User.login.notEmpty"))
    .field(_.login)(startWith ("super_", "User.login.startWith") and endWith("!", "User.login.endWith"))
    .field(_.email)(matchesRegex("""\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z""".r, "User.login.invalid"))

    .field(_.password)(lengthGt   (6, "User.password.lengthGt"))
    .field(_.password)(lengthLte  (12, "User.password.lengthLte"))

    .field(_.age)(gte(13, "User.age.gte"))
    .field(_.age)(lte(99, "User.age.lte"))
    .field(_.age)(eql(14, "User.age.eql"))
    .field(_.phone)(optional(phoneValidator))
  // also you can use `required(...)` if you want to have a defined optional field
  //.field(_.phone)(required(phoneValidator))
  //and `.seqField(...)` for validate Seq[T]
    .field(_.addresses)(nonEmpty("User.address.notEmpty"))
    .field(_.userInfo)(infoNode(nonEmpty("User.userInfo.value.notEmpty")))              //using recursion
    .field(_.userInfo)(infoNode(startWith("correct_", "User.userInfo.value.startWith")))//using recursion
    .seqField(_.addresses)
```

Lines `//using recursion` will be discussed in the next section.

Object validation can be done by using implicit method `validate` in `io.underscore.validation`

```scala
def main(args: Array[String]) {
  val correctUser = User(
    login = "super_user_!",
    email = "example@example.com",
    password = "1234567",
    age = 14,
    phone = Some(Phone("+78889993322")),
    addresses = Seq(Address("Baker st.", 221)),
    userInfo = InfoNode(
      index = 1,
      value = Some("correct_parent"),
      children = Seq(
        InfoNode(10, Some("correct_child"), Seq.empty),
        InfoNode(20, None, Seq.empty)
      )
    )
  )

  println("correctUser: " + correctUser.validate.errors)
  // prints: 'correctUser: List()'

  val allWrong = correctUser.copy(
    login = "not_super_user",
    email = """"Look at all these spaces!"@example.com""", //it is still a valid email address
    password = "short",
    age = 101,
    addresses = Seq(Address("", -1))
  )

  println("allWrong: " + allWrong.validate)
  //prints: 'allWrong:
  // Validated(
  //    User(
  //      not_super_user,
  //      "Look at all these spaces!"@example.com,
  //      short,
  //      101,
  //      Some(Phone(+78889993322)),
  //      List(Address(,-1)),
  //      InfoNode(/* doesn't right now */),
  //    List(
  //      ValidationError(User.login.startWith,ValidationPath(login)),
  //      ValidationError(User.login.endWith,ValidationPath(login)),
  //      ValidationError(User.login.invalid,ValidationPath(email)),
  //      ValidationError(User.password.lengthGt,ValidationPath(password)),
  //      ValidationError(User.age.lte,ValidationPath(age)),
  //      ValidationError(User.age.eql,ValidationPath(age)),
  //      ValidationError(Address.street.notEmpty,ValidationPath(addresses[0].street)),
  //      ValidationError(Address.house.gt,ValidationPath(addresses[0].house))
  //    )
  // )
  //'
}
```
So davegurnell/validation can be considered as a great lib for validation of very complex structures like generics and recursions if you are not bothered concerning dependence on `scala.language.experimental.macros` and `scala.language.higherKinds` which are used in this lib.

## Bonus: Recursive validation
And here is a little bonus - validation of recursive structures. Recursive validation was implemented for `io.underscore.validation` but I think it can be easily ported at least to `DValidation`.
In our domains classes we have this generic recursive structure: 

```scala
//InfoNode.scala
package io.koff.validation.domain

/**
 * Generic recursive example of domain class
 * @param index here it is just a number without additional meaning
 * @param value some value which we want to store in a node and validate
 * @param children child nodes which should also be validated
 * @tparam T value type
 */
case class InfoNode[T](index: Int, value: Option[T], children: Seq[InfoNode[T]])

object InfoNode{
  /**
   * Just a default value for InfoNode
   */
  def default[T]: InfoNode[T] = InfoNode(1, None, Seq.empty)
}
```

In order to validate it `infoNode[T](rule: Validator[T])` method has been created:

```scala
  /**
   * Recursive validator for InfoNode
   */
  def infoNode[T](rule: Validator[T]): Validator[InfoNode[T]] = Validator[InfoNode[T]] { in =>
    val generalValidator = validate[InfoNode[T]]
      .field(_.index)(gte(1, "InfoNode.index.gte"))
      .field(_.children)(lengthLte(2, "InfoNode.children.lengthLte"))
      .field(_.value)(optional(rule))

    val resultValidator = if(in.children.isEmpty) {
      generalValidator
    } else {
      generalValidator.seqField(_.children)(infoNode(rule))
    }

    resultValidator(in)
  }
```

As the argument this method receives a validation rule for `T` type - `Validator[T]`. So you can use other rules or validators in order to check all children in this recursive structure:

```scala
.field(_.userInfo)(infoNode(nonEmpty("User.userInfo.value.notEmpty")))               //using recursion
.field(_.userInfo)(infoNode(startWith("correct_", "User.userInfo.value.startWith"))) //using recursion
```

## Links

* **[Article Examples](https://github.com/coffius/koffio-validation)**
* **[Accord](https://github.com/wix/accord "GitHub: Accord")**
* **[Skinny validator](http://skinny-framework.org/documentation/validator.html "Skinny Validator: Docs")**
* **[DValidation](https://github.com/tobnee/DValidation/ "GitHub: DValidation")**
* **[io.underscore.validation](https://github.com/davegurnell/validation "GitHub: io.underscore.validation")**
