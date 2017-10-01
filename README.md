Quicklens
=========

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.softwaremill.quicklens/quicklens_2.11/badge.svg)](https://maven-badges.herokuapp.com/maven-central/com.softwaremill.quicklens/quicklens_2.11)

**Modify deeply nested fields in case classes:**

```scala
scala> import com.softwaremill.quicklens._
import com.softwaremill.quicklens._

scala> case class Street(name: String)
defined class Street

scala> case class Address(street: Street)
defined class Address

scala> case class Person(address: Address, age: Int)
defined class Person

scala> val person = Person(Address(Street("1 Functional Rd.")), 35)
person: Person = Person(Address(Street(1 Functional Rd.)),35)

scala> val p2 = person.modify(_.address.street.name).using(_.toUpperCase)
p2: Person = Person(Address(Street(1 FUNCTIONAL RD.)),35)

scala> val p3 = person.modify(_.address.street.name).setTo("3 OO Ln.")
p3: Person = Person(Address(Street(3 OO Ln.)),35)

scala> // or
     |  
     | val p4 = modify(person)(_.address.street.name).using(_.toUpperCase)
p4: Person = Person(Address(Street(1 FUNCTIONAL RD.)),35)

scala> val p5 = modify(person)(_.address.street.name).setTo("3 OO Ln.")
p5: Person = Person(Address(Street(3 OO Ln.)),35)
```

**Chain modifications:**

```scala
scala> person.
     | modify(_.address.street.name).using(_.toUpperCase).
     | modify(_.age).using(_ - 1)
res2: Person = Person(Address(Street(1 FUNCTIONAL RD.)),34)
```

**Modify conditionally:**

```scala
scala> val shouldChangeAddress = true
shouldChangeAddress: Boolean = true

scala> person.modify(_.address.street.name).setToIfDefined(Some("3 00 Ln."))
res3: Person = Person(Address(Street(3 00 Ln.)),35)

scala> person.modify(_.address.street.name).setToIf(shouldChangeAddress)("3 00 Ln.")
res4: Person = Person(Address(Street(3 00 Ln.)),35)
```

**Modify several fields in one go:**

```scala
scala> case class Person(firstName: String, middleName: Option[String], lastName: String)
defined class Person

scala> val person = Person("john", Some("steve"), "smith")
person: Person = Person(john,Some(steve),smith)

scala> person.modifyAll(_.firstName, _.middleName.each, _.lastName).using(_.capitalize)
res5: Person = Person(John,Some(Steve),Smith)
```

**Traverse options/lists/maps using .each:**

```scala
scala> case class Street(name: String)
defined class Street

scala> case class Address(street: Option[Street])
defined class Address

scala> case class Person(addresses: List[Address])
defined class Person

scala> val person = Person(List(
     |   Address(Some(Street("1 Functional Rd."))),
     |   Address(Some(Street("2 Imperative Dr.")))
     | ))
person: Person = Person(List(Address(Some(Street(1 Functional Rd.))), Address(Some(Street(2 Imperative Dr.)))))

scala> val p2 = person.modify(_.addresses.each.street.each.name).using(_.toUpperCase)
p2: Person = Person(List(Address(Some(Street(1 FUNCTIONAL RD.))), Address(Some(Street(2 IMPERATIVE DR.)))))
```

`.each` can only be used inside a `modify` and "unwraps" the container (currently supports `List`s, `Option`s and
`Maps`s - only values are unwrapped for maps).
You can add support for your own containers by providing an implicit `QuicklensFunctor[C]` with the appropriate
`C` type parameter.

**Traverse selected elements using .eachWhere:**

Similarly to `.each`, you can use `.eachWhere(p)` where `p` is a predicate to modify only the elements which satisfy
the condition. All other elements remain unchanged.

```scala
scala> def filterAddress: Address => Boolean = _ => true
filterAddress: Address => Boolean

scala> person.
     |   modify(_.addresses.eachWhere(filterAddress)
     |            .street.eachWhere(_.name.startsWith("1")).name).
     |   using(_.toUpperCase)
res6: Person = Person(List(Address(Some(Street(1 FUNCTIONAL RD.))), Address(Some(Street(2 Imperative Dr.)))))
```

**Modify specific sequence elements using .at:**

```scala
scala> person.modify(_.addresses.at(1).street.each.name).using(_.toUpperCase)
res7: Person = Person(List(Address(Some(Street(1 Functional Rd.))), Address(Some(Street(2 IMPERATIVE DR.)))))
```

Similarly to `.each`, `.at` modifies only the element at the given index. If there's no element at that index,
an `IndexOutOfBoundsException` is thrown.

**Modify specific map elements using .at:**

```scala
scala> case class Property(value: String)
defined class Property

scala> case class Person(name: String, props: Map[String, Property])
defined class Person

scala> val person = Person(
     |   "Joe",
     |   Map("Role" -> Property("Programmmer"), "Age" -> Property("45"))
     | )
person: Person = Person(Joe,Map(Role -> Property(Programmmer), Age -> Property(45)))

scala> person.modify(_.props.at("Age").value).setTo("45")
res8: Person = Person(Joe,Map(Role -> Property(Programmmer), Age -> Property(45)))
```

Similarly to `.each`, `.at` modifies only the element with the given key. If there's no such element,
an `NoSuchElementException` is thrown.

**Modify Either fields using .eachLeft and eachRight:**

```scala
scala> case class AuthContext(token: String)
defined class AuthContext

scala> case class AuthRequest(url: String)
defined class AuthRequest

scala> case class Resource(auth: Either[AuthContext, AuthRequest])
defined class Resource

scala> val devResource = Resource(auth = Left(AuthContext("fake")))
devResource: Resource = Resource(Left(AuthContext(fake)))

scala> val prodResource = devResource.modify(_.auth.eachLeft.token).setTo("real")
prodResource: Resource = Resource(Left(AuthContext(real)))
```

**Modify fields when they are of a certain subtype:**

```scala
scala> trait Animal
defined trait Animal

scala> case class Dog(age: Int) extends Animal
defined class Dog

scala> case class Cat(ages: List[Int]) extends Animal
defined class Cat

scala> case class Zoo(animals: List[Animal])
defined class Zoo

scala> val zoo = Zoo(List(Dog(4), Cat(List(3, 12, 13))))
zoo: Zoo = Zoo(List(Dog(4), Cat(List(3, 12, 13))))

scala> val olderZoo = zoo.modifyAll(
     |   _.animals.each.when[Dog].age,
     |   _.animals.each.when[Cat].ages.at(0)
     | ).using(_ + 1)
olderZoo: Zoo = Zoo(List(Dog(5), Cat(List(4, 12, 13))))
```

This is also known as a *prism*, see e.g. [here](http://julien-truffaut.github.io/Monocle/optics/prism.html).

**Re-usable modifications (lenses):**

```scala
scala> case class Street(name: String)
defined class Street

scala> case class Address(street: Street)
defined class Address

scala> case class Person(address: Address)
defined class Person

scala> val modifyStreetName = modify(_: Person)(_.address.street.name)
modifyStreetName: Person => com.softwaremill.quicklens.PathModify[Person,String] = <function1>

scala> val person = Person(Address(Street("1 Functional Rd.")))
person: Person = Person(Address(Street(1 Functional Rd.)))

scala> val anotherPerson = Person(Address(Street("2 Imperative Dr.")))
anotherPerson: Person = Person(Address(Street(2 Imperative Dr.)))

scala> modifyStreetName(person).using(_.toUpperCase)
res9: Person = Person(Address(Street(1 FUNCTIONAL RD.)))

scala> modifyStreetName(anotherPerson).using(_.toLowerCase)
res10: Person = Person(Address(Street(2 imperative dr.)))

scala> val upperCaseStreetName = modify(_: Person)(_.address.street.name).using(_.toUpperCase)
upperCaseStreetName: Person => Person = <function1>

scala> upperCaseStreetName(person)
res11: Person = Person(Address(Street(1 FUNCTIONAL RD.)))
```

**Composing lenses:**

```scala
scala> val modifyAddress = modify(_: Person)(_.address)
modifyAddress: Person => com.softwaremill.quicklens.PathModify[Person,Address] = <function1>

scala> val modifyStreetName = modify(_: Address)(_.street.name)
modifyStreetName: Address => com.softwaremill.quicklens.PathModify[Address,String] = <function1>

scala> val p6 = (modifyAddress andThenModify modifyStreetName)(person).using(_.toUpperCase)
p6: Person = Person(Address(Street(1 FUNCTIONAL RD.)))
```

**Modify nested sealed hierarchies:**

> *Note: this feature is experimental and might not work due to compilation order issues.
> See https://issues.scala-lang.org/browse/SI-7046 for more details.*

````tut
sealed trait Pet { def name: String }
case class Fish(name: String) extends Pet
sealed trait LeggedPet extends Pet
case class Cat(name: String) extends LeggedPet
case class Dog(name: String) extends LeggedPet

val pets = List[Pet](
  Fish("Finn"), Cat("Catia"), Dog("Douglas")
)

val juniorPets = pets.modify(_.each.name).using(_ + ", Jr.")
```

---

Similar to lenses ([1](http://eed3si9n.com/learning-scalaz/Lens.html),
[2](https://github.com/julien-truffaut/Monocle)), but without the actual lens creation.

Read [the blog](http://www.warski.org/blog/2015/02/quicklens-modify-deeply-nested-case-class-fields/) for more info.

Available in Maven Central:

```scala
val quicklens = "com.softwaremill.quicklens" %% "quicklens" % "1.4.8"
```

Also available for [Scala.js](http://www.scala-js.org)!
