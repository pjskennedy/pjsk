---
layout: post
title:  "Scala Case Class Hashcodes"
date:   2017-11-22T19:00:00-08:00
categories:
---

If you're new to scala, you will come across these things called "Case classes" and you'll think to yourself, "gee, these things are pretty great!". I thought the same until today. They handily define the `toString()`, `equals()` and `hashCode()` methods based on the constructor arguments.

This allows you to do the compare instances of case classes at ease.

```scala
case class Dog(name: String)

val c1 = Dog("bob")
val c2 = Dog("bob")
val c3 = Dog("sarah")

c1 == c2 // returns true
c1 == c3 // returns false
```

And if we look at the hash codes of these two objects?

```scala
c1.hashCode
// -1773925543
c2.hashCode
// -1773925543
c3.hashCode
// -1051523283
```

And all was good in the world.

### Why is `hashCode` so important?

Every object in the JVM has a `hashCode()` function defined to digest the object down a single hash value (a signed integer). This hashcode is evenly distributed for all the variations of the class of object to optimize performance in hash tables and other collections.

I naively thought I could de-dupe case class objects by the hash code. I'm lazy, the code didn't need to be perfect, and using hash codes required writing no additional code. If case classes are so great then surely they have well defined hash code functions.

```scala
trait Animal

case class Dog(name: String) extends Animal
case class Cat(name: String) extends Animal

val bobTheDog = Dog("bob")
val bobTheCat = Cat("bob")

bobTheDog == bobTheCat // returns false
```

This is great, obviously a cat is not a dog.

```scala
bobTheDog.hashCode
// -1773925543
bobTheCat.hashCode
// -1773925543
```

But their hashcodes are not different. My guess is the hash code method returns values based on the objects contained in the case class, but not the case class itself. Let's take a look in the generated code of a case class:

```scala
// Dog.scala
case class Dog(name: String)
```

```bash
$ scalac -Xprint:typer Dog.scala
```

The output (cleaned up a tiny bit) shows the implementation of the three methods mentioned earlier.

```scala
case class Dog extends AnyRef with Product with Serializable {
  // omitted
  def hashCode(): Int = scala.runtime.ScalaRunTime._hashCode(Dog.this);
  def toString(): String = scala.runtime.ScalaRunTime._toString(Dog.this);
  def equals(x$1: Any): Boolean = Dog.this.eq(x$1.asInstanceOf[Object]).||(x$1 match {
    Dog.this.eq(x$1.asInstanceOf[Object]).||(x$1 match {
      case (_: Dog) => true
      case _ => false
    }.&&({
      <synthetic> val Dog$1: Dog = x$1.asInstanceOf[Dog];
      Dog.this.name.==(Dog$1.name).&&(Dog$1.canEqual(Dog.this))
    }))
  })
}
```

While `hashCode` is defined, it is passed off to `scala.runtime.ScalaRunTime._hashCode`. After some digging around, this method is a "MurMur3" hash implementation over the values of the `Product` (seen below), but not the type of class.

Taken from [the Scala source](https://github.com/scala/scala).

```scala
//src/library/scala/runtime/ScalaRunTime.scala
def _hashCode(x: Product): Int = scala.util.hashing.MurmurHash3.productHash(x)
```

```scala
// src/library/scala/util/hashing/MurmurHash3.scala
final def productHash(x: Product, seed: Int): Int = {
  val arr = x.productArity
  // Case objects have the hashCode inlined directly into the
  // synthetic hashCode method, but this method should still give
  // a correct result if passed a case object.
  if (arr == 0) {
    x.productPrefix.hashCode
  }
  else {
    var h = seed
    var i = 0
    while (i < arr) {
      h = mix(h, x.productElement(i).##)
      i += 1
    }
    finalizeHash(h, arr)
  }
}
```

Knowing this, the following is valid.

```scala
case class LoginInfo(userName: String, password: String)
case class Dog(owner: String, name: String)

LoginInfo("peter", "spot").hashCode
// 713115107
Dog("peter", "spot").hashCode
// 713115107
```

Two completely separate objects with separate names of attributes can have the same hash code.

### What did I learn?

Objects, although seemingly unrelated in function, use, type, or interface can have the same hash code.

As of scala 2.11, there is an explanation in the [Scala documentation](http://www.scala-lang.org/api/2.11.x/index.html#scala.Product@hashCode():Int).

> Note that it is allowed for two objects to have identical hash codes (o1.hashCode.equals(o2.hashCode)) yet not be equal (o1.equals(o2) returns false). A degenerate implementation could always return 0. However, it is required that if two objects are equal (o1.equals(o2) returns true) that they have identical hash codes (o1.hashCode.equals(o2.hashCode)). Therefore, when overriding this method, be sure to verify that the behavior is consistent with the equals method.

This is backed up in the [Java documentation](http://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#hashCode--) for hash codes.

> As much as is reasonably practical, the hashCode method defined by class Object does return distinct integers for distinct objects. (This is typically implemented by converting the internal address of the object into an integer, but this implementation technique is not required by the Java™ programming language.)

The safest thing you can do is not use hash codes at all in your application code. Do **not** use hash codes for uniqueness.

### How did I hack around this?

All I really wanted was hashCodes to be unique for a certain set of case classes. They _did_ luckily share a common trait; I simply patched the `hashCode` method to include the class name. Realistically though, this is still bad practice.

```scala
trait Animal {
  override def hashCode: Int =
    41 * (41 + this.getClass.getName.hashCode) + super.hashCode()
}

case class Dog(name: String) extends Animal
case class Cat(name: String) extends Animal

Dog("bob").hashCode == Cat("bob").hashCode // false!
```

