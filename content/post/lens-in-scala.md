+++
date = "2015-09-19T22:01:13+03:00"
draft = false
title = "Lens in scala"
slug = "292173-lens-in-scala"
+++


In this article let's take a look at such a thing as lens(or lenses).
A Lens is an abstraction from functional programming which helps to deal with a problem of updating complex immutable nested objects like this:
```scala
case class User(id: UserId, generalInfo: GeneralInfo, billInfo: BillInfo)
case class UserId(value: Long)
case class GeneralInfo(email: Email,
                       password: String,
                       siteInfo: SiteInfo,
                       isEmailConfirmed: Boolean = false,
                       phone: String,
                       isPhoneConfirmed: Boolean = false)
case class SiteInfo(alias: String, avatarUrl: String, userRating: Double = 0.0d)
case class Email(value: String)
case class BillInfo(addresses: Seq[Address], name: Name)
case class Name(firstName: String, secondName: String)
case class Address(country: Country, city: City, street: String, house: String, isConfirmed: Boolean = false)
case class City(name: String)
case class Country(name: String)
```
If we want to increase `userRating` in this model then we will have to write such a code:
```scala
val updatedUser = user.copy(
  generalInfo = user.generalInfo.copy(
    siteInfo = user.generalInfo.siteInfo.copy(
      userRating = user.generalInfo.siteInfo.userRating + 1
    )
  )
)
```
And we have to write the code below to confirm all of the addresses in BillInfo
```scala
val updatedAddresses = user.billInfo.addresses.map(_.copy(isConfirmed = true))
val updatedUser = user.copy(
	billInfo = user.billInfo.copy(addresses = updatedAddresses)
)
```
If we increase a level of nesting in our structures then we will considerably increase amount of a code like this. In such cases lens give a cleaner way to make changes in nested structures.

<!--more-->

Using quicklens we can do it much simpler:
```scala
import com.softwaremill.quicklens._
//update rating using quicklens
val userWithRating = user.modify(_.generalInfo.siteInfo.userRating).using(_ + 1)
//confirm all the addresses of a user with quicklens
val userWithConfimedAddresses = user.modify(_.billInfo.addresses.each.isConfirmed).using(_ => true)
```
Now we have base understanding of lens purpose. In the next parts of this article we will see how to use each of the libraries from the list.

# Available implementations

There are several implementations in scala:

* scalaz.Lens
* [Quicklens](https://github.com/adamw/quicklens "Quicklens")
* [Sauron](https://github.com/pathikrit/sauron "Sauron")
* [Monocle](http://julien-truffaut.github.io/Monocle/ "Monocle")

We will take a look at all of them in this article. A project with examples can be found [here](https://github.com/coffius/koffio-lenses "Examples of lens use")

# scalaz.Lens
If we want to use `scalaz.Lens` at first we should define lens:
```scala
val siteInfoRatingLens = Lens.lensu[SiteInfo, Double](
  (info, value) => info.copy(userRating = value),
  _.userRating
)
```
The first type parameter is needed to set in which class(`MainClass`) we will change value and the second type parameter defines the class(`FieldClass`) of the field which we will change with the lens. As you can see we should also send two functions to `lensu(...)` method. The first function defines how to change `MainClass` using a new value. The second function is used to get value of the field which we want to change.
In order to make possible changes of `userRating` field directly in `User` object we should create additional lens.
```scala
val generalInfoSiteInfoLens = Lens.lensu[GeneralInfo, SiteInfo](
  (general, site) => general.copy(siteInfo = site),
  _.siteInfo
)

val userGeneralInfoLens = Lens.lensu[User, GeneralInfo](
  (user, info) => user.copy(generalInfo = info),
  _.generalInfo
)
```
and compose them in the chain `User.generalInfo -> GeneralInfo.siteInfo -> SiteInfo.userRating`.
We can use different approaches:

* `>=>` - alias for `andThen(...)` method
* `<=<` - alias for `compose(...)` method

Example below:
```scala
//andThen
val userRatingLens1 = userGeneralInfoLens >=> generalInfoSiteInfoLens >=> siteInfoRatingLens
val userRatingLens2 = userGeneralInfoLens.andThen(generalInfoSiteInfoLens).andThen(siteInfoRatingLens)

//compose
val userRatingLens3 = siteInfoRatingLens <=< generalInfoSiteInfoLens <=< userGeneralInfoLens
val userRatingLens4 = siteInfoRatingLens.compose(generalInfoSiteInfoLens).compose(userGeneralInfoLens)

val user = ProblemExample.user

//same operations
println(userRatingLens1.set(user, 1).generalInfo.siteInfo.userRating)
println(userRatingLens2.set(user, 2).generalInfo.siteInfo.userRating)
println(userRatingLens3.set(user, 3).generalInfo.siteInfo.userRating)
println(userRatingLens4.set(user, 4).generalInfo.siteInfo.userRating)
```
If you want to change `isConfirmed` to true in each address as it is described in the introduction example then you should use a different operator: `=>=` - alias for `mod(...)` method
This operator get value using lens, modify it and create a new object with a changed value.
```scala
val userBillInfoLens = Lens.lensu[User, BillInfo](
  (user, info) => user.copy(billInfo = info),
  _.billInfo
)

val billInfoAddressesLens = Lens.lensu[BillInfo, Seq[Address]](
  (info, addresses) => info.copy(addresses = addresses),
  _.addresses
)

val isConfirmedLens = (userBillInfoLens >=> billInfoAddressesLens) =>= { _.map(_.copy(isConfirmed = true)) }

val user = ProblemExample.user
println(isConfirmedLens(user).billInfo.addresses)
```
That is how we can use scalaz.Lens. It is quite hard and we will reduce amount of the code only if we have very complex nesting and implement enough lens to compose them. But now we have a notion about how we can use scalaz.Lens

# Quicklens
Use of scalaz.Lens is quite difficult but if we are not afraid to use macros in a project we might use `quicklens` instead. You have already seen a simple example for `quicklens` so let's go deeper and see what else `quicklens` can do.

`Quicklens` has support of chain modifications which can be helpful if you want to change several fields at the same time
```scala
import com.softwaremill.quicklens._
val user = ProblemExample.user
val updatedUser = user
   .modify(_.generalInfo.siteInfo.userRating).using(_ + 1)
   .modify(_.billInfo.addresses.each.isConfirmed).using(_ => true)

println(updatedUser.generalInfo.siteInfo.userRating)
println(updatedUser.billInfo.addresses)
```
It is also possible to create reusable lens as well as in scalaz.Lens
```scala
import com.softwaremill.quicklens._
val userRatingLens = modify(_:User)(_.generalInfo.siteInfo.userRating).using _
val user = ProblemExample.user
val updatedUser1 = userRatingLens(user)(_ + 10)
val updatedUser2 = userRatingLens(user)(_ + 12)

println(updatedUser1.generalInfo.siteInfo.userRating)
println(updatedUser2.generalInfo.siteInfo.userRating)
```
Of course lens composition is also possible:
```scala
import com.softwaremill.quicklens._
//create lens
val generalInfoLens = modify(_:User)(_.generalInfo)
val emailConfirmedLens = modify(_:GeneralInfo)(_.isEmailConfirmed)
val phoneConfirmedLens = modify(_:GeneralInfo)(_.isPhoneConfirmed)

//compose the lens
val confirmEmail = generalInfoLens.andThenModify(emailConfirmedLens)(_:User).using(_ => true)
val confirmPhone = generalInfoLens.andThenModify(phoneConfirmedLens)(_:User).using(_ => true)

val user = ProblemExample.user
//compose the functions in order to make both changes at once
val updatedUser = confirmEmail.andThen(confirmPhone)(user)

println(updatedUser.generalInfo.isEmailConfirmed)
println(updatedUser.generalInfo.isPhoneConfirmed)
```
# Sauron
As it is said on the main page of `Sauron` repo it has been inspired by `quicklens` but it has much simpler implementation and less number of features. And also has additional dependency on `"org.scalamacros" % "paradise" % "2.1.0-M5"`

So lets see what exactly `sauron` can do.
The first is changing of value of userRating
```scala 
import com.github.pathikrit.sauron._

val user = ProblemExample.user
val updatedUser = lens(user)(_.generalInfo.siteInfo.userRating)(_ + 10)
println(updatedUser.generalInfo.siteInfo.userRating)
```
Then reusing lens in order to change a specific object:
```scala
import com.github.pathikrit.sauron._
val user = ProblemExample.user
val userRatingLens = lens(user)(_.generalInfo.siteInfo.userRating)
val userWith20Rating = userRatingLens(_ => 20)
val userWith100Rating = userRatingLens( _ + 100 )

println(userWith20Rating.generalInfo.siteInfo.userRating)
println(userWith100Rating.generalInfo.siteInfo.userRating)
```
And the example below shows hot to define lens for changing different objects:
```scala
import com.github.pathikrit.sauron._

val userRatingLens = lens(_:User)(_.generalInfo.siteInfo.userRating)

val user = ProblemExample.user

val userWith20Rating = userRatingLens(user)(_ => 20)
val userWith100Rating = userRatingLens(user)( _ + 100 )

println(userWith20Rating.generalInfo.siteInfo.userRating)
println(userWith100Rating.generalInfo.siteInfo.userRating)
```
Also `sauron` has lens composition:
```scala
import com.github.pathikrit.sauron._
val generalInfoLens = lens(_:User)(_.generalInfo)
val emailConfirmedLens = lens(_:GeneralInfo)(_.isEmailConfirmed)
val phoneConfirmedLens = lens(_:GeneralInfo)(_.isPhoneConfirmed)

val user = ProblemExample.user
val confirmEmail = generalInfoLens.andThenLens(emailConfirmedLens)(_:User)(_ => true)
val confirmPhone = generalInfoLens.andThenLens(phoneConfirmedLens)(_:User)(_ => true)

//compose the functions in order to make both changes at once
val updatedUser = confirmEmail.andThen(confirmPhone)(user)

println(updatedUser.generalInfo.isEmailConfirmed)
println(updatedUser.generalInfo.isPhoneConfirmed)
```
You can see that the example above is quite similar to `quicklens` example of lens composition.
# Monocle
The last lens library which we will direct our attention to is `Monocle`. It is not just a lens library. It also contains logic for work with prisms but here we will only look at a lens's part of the library.
As other libraries `Monocle` supports lens creation. Common way to create lens is very similar to scalaz.Lens - we should create individual lens for our types manually:
```scala
import monocle.Lens

//create lens
val generalInfoLens = Lens[User, GeneralInfo](_.generalInfo)(info => user => user.copy(generalInfo = info))
val siteInfoLens = Lens[GeneralInfo, SiteInfo](_.siteInfo)(site => general => general.copy(siteInfo = site))
val userRatingLens = 
        Lens[SiteInfo, Double](_.userRating)(rating => siteInfo => siteInfo.copy(userRating = rating))

//and compose them together
val changeRatingLens = generalInfoLens.composeLens(siteInfoLens).composeLens(userRatingLens)
val user = ProblemExample.user

val updatedUser = changeRatingLens.set(20)(user)
println(updatedUser.generalInfo.siteInfo.userRating)
```
Simpler way is to use macros
```scala
import monocle.macros.GenLens
val changeRatingLens = GenLens[User](_.generalInfo.siteInfo.userRating)
val plus100RatingLens = changeRatingLens.modify(_ + 100)

val user = ProblemExample.user
val updatedUser = plus100RatingLens(user)
println(updatedUser.generalInfo.siteInfo.userRating)
```
There also is support of the annotation `@Lenses` which generates `monocle.Lenses` for all fields of a case class. If we define a case class using this annotation all its fields will have type like `monocle.Lens[S, A]`
```scala
import monocle.macros.Lenses
@Lenses case class Address(name: String)
@Lenses case class Person(address: Address)

val addressNameLens = Person.address composeLens Address.name
val changeNameFunc = addressNameLens.modify(_.toUpperCase)(_:Person)

val person = Person(Address("person_address"))
val updatedPerson = changeNameFunc(person)

println(updatedPerson)
```
This annontation might be helpful if you want to use lens pretty often in your code.

# Results

* scalaz.Lens - if you already have scalaz in a project and you are not bothered to write some code in order to define lens
* [Quicklens](https://github.com/adamw/quicklens "Quicklens") - easy to use and powerful enough to deal with the described problem
* [Sauron](https://github.com/pathikrit/sauron "Sauron") - very similar to [Quicklens](https://github.com/adamw/quicklens "Quicklens") and has a less size but also has less fucntionality
* [Monocle](http://julien-truffaut.github.io/Monocle/ "Monocle") - a powerful library which can help if there is necessity to use lots of lens in a code.

My choise is [Quicklens](https://github.com/adamw/quicklens "Quicklens") because it is not so complex as scalaz.Lens and it does what is needed.

# Links

* [coffius/koffio-lenses](https://github.com/coffius/koffio-lenses "coffius/koffio-lenses") - examples for this article
* [Quicklens](https://github.com/adamw/quicklens "Quicklens")
* [Sauron](https://github.com/pathikrit/sauron "Sauron")
* [Monocle](http://julien-truffaut.github.io/Monocle/ "Monocle")