---
date: 
  created: 2024-12-09
slug: better-java-2
tags:
  - JVM
  - Kotlin
---

# What does it mean to be a better Java? (Part 2)

Last week, we studied ways in which Kotlin improves upon Java when it comes to creating and destroying objects, eliminating Java pitfalls and footguns. This week, we discuss the common supertypes: `Object`, `Any` and `Comparable`.

<!-- more -->

!!! note ""
    This article is the second part in a series in which I use Effective Java to grade whether Kotlin is a good language, according to Java's own rules. If you haven't already, [read the first article](better-java-1.md).

In this post, we continue where we left off, and discuss chapter 3 of Effective Java: _Methods common to all objects_.

### Items 10–12. Overriding equals, hashCode and toString

The `equals`, `hashCode` and `toString` methods are universal methods that all objects have—either they implement it themselves, or they get a default implementation. In Java, they are defined in `Object`, and in Kotlin they are defined in `Any`, but are otherwise completely identical in both languages.

`toString` is the simplest to implement of the three: `toString` is used to help programmers understand the data the program is working with. By overriding it, programmers can customize how the debugger and print statements display information.

`equals` and `hashCode` are much more sensitive: they define whether two things are _equal to each other_. Both methods have a strong contract that implementors should respect. Otherwise, most data structures will behave erratically, often not respecting their own contracts (the typical case being a `Set` which starts to contain duplicates).

By default, `equals` and `hashCode` consider that two objects are equal based on their referential identity: that is, two objects are equal if they are a single object in memory. Two objects in different memory locations, even if they are strictly identical bitwise, are not considered equal.

A realization of modern Java is that all objects can be grouped into two categories:

- Objects that are primarily data holders. These objects should be modeled as records/enums (in Java) or sealed/data/value/enum classes (in Kotlin), and preferably be immutable. These objects should define their respective relationships (equality, ordering…).
- Objects that are primarily behavior providers, that are mainly used to trigger actions against the rest of the system. These objects should be modeled as regular classes. These objects should rarely be added to containers, and they should be understood to be unique (relying on the default implementation of `equals` and `hashCode`).

Using referential identity where logical equality should be used is a common source of bugs. One especially pervasive case is the referential identity of `String`, which [is notoriously unpredictable](https://www.baeldung.com/java-compare-strings). In Java, this transforms the easy-to-read (but buggy):

```java
boolean isTest(String tenantName) {
	return tenantName == "test";
}
```

into the less readable:

```java
boolean isTest(String tenantName) {
	return tenantName.equals("test");
}
```

Yet, we must still go further. This example is still wrong, as it will fail with a `NullPointerException` if `tenantName` is `null`. Instead, we must use:

```java
import java.util.Objects;

boolean isTest(String tenantName) {
	return Objects.equals(tenantName, "test");
}
```

Certainly, this example is better in terms of behavior, as it will properly return `false` when the `tenantName` is `null`. However, I find it quite harder to read, as well as quite harder to write: we can no longer rely on auto-complete to write the call.

The first way in which Kotlin simplifies this is to flip the cases: since logical equality is what is used the vast majority of the time, it should get the simpler code.

|        | Referential identity | Logical equality       |
|--------|----------------------|------------------------|
| Java   | `a == b`             | `Objects.equals(a, b)` |
| Kotlin | `a === b`            | `a == b`               |

The previous example can thus translate to the following Kotlin code:

```kotlin
fun isTest(tenantName: String?) = 
	tenantName == "test"
```

with all the same features: if the value is `null`, `false` is returned, and otherwise `String.equals` is called.

Recently, Java added records, which greatly simplify implementing `equals` and `hashCode` for simple data holders. In the future, Project Valhalla may introduce proper value classes that improve on this further. Kotlin has `data`, `sealed`, `object` and `value` classes that all help in this regard in similar ways. In other situations, developers usually use their IDE to generate the method anyway ([though you may be interested in watching this JEP Café](https://www.youtube.com/watch?v=kuzjX_efuDs)).

**Note that this plays very badly with mutability.**
In both languages, it is expected that `equals` and `hashCode` are stable: that is, they return the same result as long as the object hasn't changed. But more than that, the object is not allowed to change while observable by a data structure. For example, when an object is stored in an `HashSet`, if it is modified, and later a `contains` check is made, the set will answer that it doesn't contain the object: the set is comparing the object with the `hashCode` value from _when it was added to the set_.
Both languages therefore encourage immutability; we will see what Kotlin provides in [item 17](#item-17-minimize-mutability).

Implementing `equals` and `hashCode` themselves is still tricky in both languages. I recommend reading Effective Java, which goes into details on multiple strategies, pitfalls, and facts I haven't seen mentioned anywhere else.

### Item 13. Override clone judiciously

Java's `Cloneable` interface and `Object#clone` method are known to be poorly designed and dangerous to use in practice (the book goes into details on this topic). Kotlin simply doesn't have this feature: Kotlin classes do not have a `clone` method.

Just like modern Java, Kotlin recommends that classes that can be meaningfully cloned expose this feature either as additional constructors, or as dedicated methods, completely avoiding the `Cloneable` interface.

### Item 14. Consider implementing Comparable

The `Comparable` interface describes the _natural ordering_ of some objects. It is typically implemented by classes that have clear ordering semantics, like `String`, `BigDecimal` and `Instant`.

The `Comparator` interface describes any other ordering. Unlike `Comparable`, which is implemented directly on the object (and thus can only be implemented by the object's author), `Comparator` is implemented externally. `Comparator` is convenient to provide additional ways of ordering objects, without necessarily saying they are the correct way to do so. For example, a `Car` class could provide a `Comparator` instance that orders by model launch date, another one that compares by the power of the engine, and another one by the amount of space in the boot.

Typically, a data structure that contains `Comparable` instance will expose additional methods using these capabilities:

```kotlin
listOf(1, 2, 3, 4, 5, 6)
	.sorted()
```

The same data structures typically provide additional methods to use a specific `Comparator` instance:

```kotlin
listOf(Car("Peugeot 207", 2012), Car("Mercedes c220"), 2019)
	.sortedWith(Car.releaseDate)
```

In Java, you could check whether two objects are greater than each other through the `compareTo` method:

```java
import java.math.BigDecimal;

BigDecimal max(BigDecimal first, BigDecimal second) {
	return first.compareTo(second) >= 0 ? first : second;
}
```

Much like how Kotlin created the operator `==` that calls the `equals` method, Kotlin allows the use of the typical range operators on anything that implements `Comparable`:

```kotlin
fun max(first: BigDecimal, second: BigDecimal) =
	if (first >= second) first
	else second
```

Kotlin extends this ability to use common symbols for other specific operators. Unlike C++, which allows the creator of any operator, Kotlin is stricter with the contracts to follow. Using mathematical operators, in particular, helps with `BigInteger` or `BigDecimal` code massively. 

I was originally going to mention the poorly-known comparator helpers such as [`compareBy`](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.comparisons/compare-by.html), [`maxOf`](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.comparisons/max-of.html) and [`minOf`](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.comparisons/min-of.html), but Java also has them.

***

The Kotlin super-type, `Any`, has many fewer features than Java's `Object`. In this chapter, we mentioned the removal of `clone`, but Kotlin essentially removes everything other than `equals`, `hashCode` and `toString`. As we will see, this will be a recurring point in future posts.

This is mainly made possible by Kotlin's superpower: extension functions. Extension functions are useful for application developers, but that is nothing compared to their usefulness for library authors; extension functions provide extremely powerful ways to evolve APIs without introducing breaking changes. Many functionalities that must be available anywhere can be provided as extensions, without polluting the objects themselves. You can read more about this in [Roman Elizarov's blog post on extension-oriented design](https://elizarov.medium.com/extension-oriented-design-13f4f27deaee).

Next week, we will visit classes, interfaces, and object-oriented architecture.
