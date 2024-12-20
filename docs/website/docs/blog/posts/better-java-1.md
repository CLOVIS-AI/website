---
date:
  created: 2024-12-02
slug: better-java-1
tags:
  - JVM
  - Kotlin
---

# What does it mean to be a better Java? (Part 1)

In the old days, programming languages would come and go. In the modern age, languages remain for decades as platforms on top of which new ecosystems are born, never to be dethroned. Java is the venerable sage of the server-side world—but new languages still attempt to take it on.

<!-- more -->

A lot of (metaphorical and literal) ink has been used on debating whether Kotlin is the "future" of Java. When a Java developer asks me why I learned Kotlin, I usually mention something about Kotlin being "the same concepts as Java, but with a nicer syntax". At a high-level, even if the syntax is different, idiomatically-structured Kotlin code looks a _lot_ like idiomatically-structured Java code. But Java code is very rarely idiomatically-structured, especially because that tends to be quite verbose and error-prone.

I discovered Kotlin while I was learning Java in university. I had already been programming in Java for a few years, but it was all self-taught. I had burned myself multiple times with projects that were too big and too badly organized to maintain, and had to scrap a few good ideas just because I couldn't make sense of anything anymore. University courses talked of maintainability issues, and I felt what they meant on a deep level.

I loved Java because it was well-organized. Nice, but without hiding performance tweaks. Having different collection types with tradeoffs that you had to know about, but could use interchangeably through their interface, was and remains something few languages do. Java knew how to maintain a large project over time. But the way to do so, often, required very common but still verbose patterns. It always felt like Java could _do more_ to empower them.

When I first learned of Kotlin, I wasn't convinced. A new language? Will it really be better than Java? Well, Kotlin promised to be like Java, but _more_. More multiplatform (being available on the web and on mobile), more expressive, **more fun**.

It's that last claim I want to discuss today. I want to show how Kotlin improves on Java, in the way that convinced me to give it a try. Not because it was a new shiny thing, but because it let me do the same things as I did, simpler. In a way, it was a better Java, by Java's own definition.

## The Book

Effective Java, by Joshua Bloch, is one of the most impactful Java books written. It is extremely regularly cited as the source of truth for idiomatic Java and has, I think, aged fairly well.

In this blog post, I will go through some items of Effective Java, comparing what they claim to be idiomatic Java, and what Kotlin brings to the table. I don't have access to the Kotlin designers' thoughts, but it is my belief that they treated this book as a sort a todo-list of things to improve, as we will see.

Of course, this is a blog post and not an entire book, so I do not have the space (nor time) to discuss all items. The book starts with fundamental code organization patterns and ends with more specific tips (serialization, thread-safety) that are more of a library issue than a core language issue. I will therefore concentrate on the items near the start. For all items I skipped, consider they are either concepts that are identical in Java and Kotlin, or are matters of syntax where Kotlin kept the same solution as Java.

I was originally going to make this a single blog post, but I'm now 3 weeks and 4500 words into a single page, which is starting to be excessive. I'll thus split this article into multiple parts, following the same chapters as Effective Java.

## Creating and destroying objects

### Item 1. Consider static factory methods instead of constructors

Constructors are great, but have a few limitations:

- They do not have names. Non-straightforward constructors are unclear; what do you think `#!java BigInteger(int, int, Random)` does?
- Since they do not have names, two constructors cannot share the same signature.
- They are required to return a new object each time. However, immutable classes could benefit from reusing the same instances.
- They are required to return an instance of their exact type. They cannot return a subtype.

In contrast, factory methods are free from these requirements. It is a very common pattern to create a library that only has interfaces and factories as its API, with everything else being implementation details. For example, we can see this pattern in Java with `Collections.unmodifiableList`, in Kotlin with the entirety of `Flow`.

In Java, this would look like:

```java
// File Collection.java
public interface Collection<T> {
	T get(int index);
}

// File Collections.java
public class Collections {
	private Collections() {
	}

	public static <T> Collection<T> unmodifiableList(/* … */) {
		return new UnmodifiableList<>(/* … */);
	}
}

// File UnmodifiableList.java
class UnmodifiableList<T> implements Collection<T> { //(1)!
	@Override
	T get(int index) {
		// …
	}
}
```

1. `UnmodifiableList` has no visibility declared, which means it is package-private.

As you can see, each time we want to add a new implementation, we must modify `Collections`. This pattern is very common in the Java standard library: of course, the `Collections` class, but also `Files`, `Paths`, `StreamSupport`, or even `EnumSet` (whose factories return different implementations optimized for different set sizes).

Kotlin simplifies this pattern by allowing top-level functions. Even further, the implementation classes can go from package-private to file-private, which does improve the experience of the library author (but is identical to the consumer) :

```kotlin
// File Collection.kt
interface Collection<T> {
	fun get(index: Int): T
}

// File UnmodifiableList.kt
private class UnmodifiableList<T> : Collection<T> {
	override fun get(index: Int): T
}

fun <T> unmodifiableList(/* … */) = UnmodifiableList(/* … */)
```

Fundamentally, the same recommendation applies to Java and Kotlin. However, it is easier to apply in Kotlin, since less boilerplate is needed. On such a small example, it may not look game-changing, but it does reduce the amount of things to keep in mind while navigating code by a lot.

Kotlin transparently uses this to improve the performance of your program. For example, the `listOf` factory returns different implementations that are optimized for different sizes: a singleton for the 0 case, a simple wrapper class for the 1 case, and finally an `ArrayList` for the general case.

### Item 2. Consider a builder when faced with many constructor parameters

A common pattern in Java is to have constructors with telescoping arguments: overloads with each an additional argument, calling another overload after substituting a default value for its additional argument.

```java
public class Foo {

	Foo(int i) {
		this(i, null);
	}

	Foo(int i, String s) {
		this(i, s, "u");
	}

	Foo(int i, String s, String u) {
		this(i, s, u, true);
	}

	Foo(int i, String s, String u, boolean alive) {
		this(i, s, u, alive, false);
	}

	Foo(int i, String s, String u, boolean alive, boolean legal) {
		this(i, s, u, alive, legal, new Random());
	}

	Foo(int i, String s, String u, boolean alive, boolean legal, Random seed) {
		// …
	}

}
```

Not only is this painful to maintain, it also quite hard to use: who remembers the order between the two booleans?

The book recommends using builders instead, which indeed solve the problem but are quite complex by themselves. Before diving into builders, let's see another option Kotlin gives us: default and named parameters.

The entire previous example could be written in Kotlin as:

```kotlin
class Foo(
	i: Int,
	s: String? = null,
	u: String = "u",
	alive: Boolean = true,
	legal: Boolean = false,
	seed: Random = Random,
)
```

Just like Java, we can call this constructor by specifying just the arguments we want, in order:

```kotlin
Foo(5, "foo", "d")
```

However, unlike Java, we can also do so even if we want to skip parameters, by specifying names explicitly, in which case the order is not relevant:

```kotlin
Foo(5, "foo", seed = Random(12), alive = false)
```

Not only is this more flexible, it also makes boolean parameters much easier to use, as a reviewer doesn't have to open their IDE to know what they do.

Kotlin also provides the `@JvmOverloads` annotation to generate telescoping overloads, if you want to provide them to your Java consumers.

The builder pattern still has a few advantages over this, especially around binary compatibility. In Java and Kotlin, following the above approach, we cannot remove any argument from these overloads without breaking binary compatibility. Builders, however, do not suffer from these problems.

I won't elaborate on how to write builders here, because they are essentially identical in Java and Kotlin. Kotlin does give us a few new possibilities on the call-site, however. To illustrate them, consider this Java example:

```java
Car foo() {
	return new Car.Builder()
		.brand("…")
		.model("…")
		.yearOfProduction(2024)
		.numberOfSeats(5)
		.build();
}
```

Thanks to the fluent API pattern, this example has close to no boilerplate. Still, Java is primarily an imperative language, and this doesn't interact well with imperative language features. What if we wanted to set the year of production only if a given condition is true? We have to give up on the fluent API:

```java
Car foo() {
	var builder = new Car.Builder();
	builder.brand("…");
	builder.model("…");
	builder.numberOfSeats(5);

	if (/* … */)
		builder = builder.yearOfProduction(2024);

	return builder.build();
}
```

This is already significantly less nice to look at. As code grows, the builder pattern becomes unwieldy, limiting us.

Kotlin's `apply` method can be used to temporarily overload the receiver ("the `this`") of an operation. This is helpful as it allows avoiding repeating the builder. For example, the first example can be rewritten as:

```kotlin
fun foo() = Car.Builder().apply {
	brand("…")
	model("…")
	yearOfProduction(2024)
	numberOfSeats(5)
}.build()
```

This is already shorter (but not meaningfully so), and has the benefit of playing nice with imperative features; here is the second example:

```kotlin
fun foo() = Car.Builder().apply {
	brand("…")
	model("…")

	if (/* … */)
		yearOfProduction(2024)

	numberOfSeats(5)
}.build()
```

Since this is just powered by the `apply` standard library function, it works even if the builder is written in Java.

Typically, Kotlin libraries provide a simple top-level function to completely hide the builder. For example, in the standard library, `buildList`, `buildSet` and `buildString`, which would give the final result of:

```kotlin
fun foo() = buildCar {
	brand("…")
	model("…")

	if (/* … */)
		yearOfProduction(2024)

	numberOfSeats(5)
}
```

### Item 3. Enforce the singleton property

Usage of singletons in Java and Kotlin differs quite a lot. Java mostly uses them as a stateful static instance (which Kotlin doesn't need since static instances are stateful anyway), and Kotlin mostly uses them to model sentinels in sealed hierarchies. Still, both languages require the usage of singletons from time to time.

In Java, a singleton is usually declared as such:

```java
class Singleton {
	private Singleton() {
	}

	public final static Singleton INSTANCE = Singleton();
}
```

Note that there are many seemingly identical but incorrect implementations: for example, using a static method accessor which calls the constructor if the field is not `null` (would require synchronization to be correct). Read the Effective Java book for more information on such issues.

The equivalent can be declared in Kotlin as:

```kotlin
object Singleton
```

Interestingly, the final recommendation of the book is to use a single-element enum, mostly because it avoids the JVM's serialization issues. Kotlin's `#!kotlin object` declaration does not do this, but JVM serialization has fallen out of favor for more than a decade now, replaced by libraries such as Jackson.

### Item 4. Enforce noninstantiability with a private constructor

This item is different: not because Kotlin provides a nicer way to do this, but Kotlin removes the reason to do this. Non-instantiable Java classes are usually used to provide static-only APIs of which "instances" make no sense. In that way, the mere existence of this pattern is a sort of flaw in Java's "everything is an object" motto.

This use-case is entirely replaced by top-level functions, and maybe `#!kotlin object` declarations for more complex cases. Therefore, there is no need for non-instantiable classes.

There is still one, though: [the `Nothing` type](https://medium.com/@sandeepkella23/everything-about-nothing-in-kotlin-b62735d815e4).

### Item 8. Avoid finalizers and cleaners

Java recommends against using finalizers, and they are deprecated since Java 9. Kotlin doesn't have them at all, which stops incorrect usage.

Cleaners are part of the Java standard library (not the language) and are thus available as-is to Kotlin, with the same issues and possible misuse.

### Item 9. Prefer try-with-resources to try-finally

Concerns about this point are largely identical in Java and Kotlin: much like in Java, `try-finally` is often misused in Kotlin. The main difference is that Java provides a language-level solution, whereas Kotlin provides a standard library solution:

```java title="Java"
String firstLineOfFile(String path) throws IOException {
	try (BufferedReader br = new BufferedReader(new FileReader(path))) {
		return br.readLine();
	}
}
```

```kotlin title="Kotlin"
fun firstLineOfFile(path: String): String {
	return FileReader(path).buferred().use {
		it.readLine()
	}
}
```

## Conclusion of the first part

As time allows, I am planning on continuing to write this comparison, hopefully covering the whole book. After so much time working with Kotlin near exclusively, it is a bowl of fresh air to research Java in more depth and go back to my roots, in a way. When I first read Effective Java, I was still a beginner programmer, and I didn't have enough experience to appreciate the finer points; I'm glad I'm taking the time to revisit them now.

Writing this article (series of?) is a bit complex because I'm commenting on the book using its own structure, whereas I usually try to have a thread to follow to provide a better flow. I hope it's still interesting to read—don't hesitate to send feedback.  
