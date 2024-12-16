---
date: 2024-12-16
slug: better-java-3
tags:
  - JVM
  - Kotlin
---

# What does it mean to be a better Java? (Part 3)

This week, we explore software architecture: interfaces, classes, objects and inheritance. We grade Kotlin through Java's prism, and thus mainly focus on object-oriented design.

<!-- more -->

!!! note ""
    This article is the third part in a series in which I use Effective Java to grade whether Kotlin is a good language, according to Java's own rules. If you haven't already, [read the first](better-java-1.md) and [second articles](better-java-2.md).

In this post, we continue where we left off, and discuss chapter 4 of Effective Java: _Classes and interfaces_.

### Item 15. Minimize the accessibility of classes and members

Kotlin makes a few changes to accessibility of members. First, Kotlin introduces the `internal` visibility, which is similar to Java's "public but in a package that isn't exported" visibility when using modules—but is used as a regular keyword on a specific declaration, not on an entire package.

Second, Kotlin removes the package-private visibility. This is a [controversial choice](https://discuss.kotlinlang.org/t/another-call-for-package-private-visibility/9577) which mostly stems in a few limitations of packages, including that they are accessible by classes in other modules that declare the same package, which makes incremental compilation and other analysis more complex.

Third, Kotlin introduces private-in-file visibility, declared using the `private` modifier on a top-level symbol. Java doesn't allow declaring top-level private classes, since they would not be visible from the outside world (see [item 25](#item-25-limit-source-files-to-a-single-top-level-class)). Kotlin allows private top-level symbols (classes, functions, properties…), which are then available to other declarations in the same file.

It is worth noting that Kotlin changes the default visibility from package-private (which it doesn't have) to `public`. This may seem like a step back (why not default to `internal`?), and in a way it is, but it stems from a [practical decision](https://blog.jetbrains.com/kotlin/2015/09/kotlin-m13-is-out/#visibilities): in application code, the vast majority of declarations are `public`, and having to specify so everywhere would clutter the code. `internal` is mostly used by library authors, which have to be more careful with API stability anyway, so it isn't a major issue.

Kotlin, unlike Java, was designed after the advent of build tools such as Gradle and Maven. These build tools impose a very strict definition of what a module is. Java's packages are older, and don't always play well with modules. Java 9's module system certainly is an attempt to fit the Java language into tooling, but it doesn't seem to have reached broad adoption by build tools as they provide relatively similar features. Kotlin's move from packages to modules as the unit of visibility is rooted in this mismatch: Kotlin lets us use the build tool to define the relationship with other modules (e.g. Gradle's `implementation` and `api` configurations) instead of trying to handle it within the language itself. It is thus more idiomatic to have a large project with many modules each having few packages, than having a large project with few modules each having a very large amount of packages, unlike old school Java applications. 

### Item 16. Use public accessors, not public fields

Public fields in Java can introduce a hefty maintenance burden. As users interact with the field directly, it isn't possible to intercept their attempts to read or write data. Regarding writing, this stops the class from enforcing its own invariants, which often causes these invariants to "leak" into the outside world, as users are forced to verify them themselves. Regarding reading, it means optimizations such as lazy loading (see [item 83](#item-83-use-lazy-initialization-judiciously)) cannot be added later on, or even simply transforming the field into a computed value from another field. API authors cannot make the field private without a breaking changes, public fields thus tend to "lock" a specific API surface.

Java code is advised to never expose public fields, instead keeping them private and exposing getters and setters:

```java
class Point {
	private double x;
	private double y;

	public Point(double x, double y) {
		this.x = x;
		this.y = y;
	}

	public double getX() {
		return x;
	}

	public double getY() {
		return y;
	}
}
```

Kotlin not only makes this the default, but makes this impossible to avoid. **Fields are _always_ private**. What the developer declares are _properties_, a grouping of a field (always private), a getter, and optionally a setter.

The previous example can thus be rewritten as:

```kotlin
class Point {
	val x: Double
	val y: Double

	constructor(x: Double, y: Double) {
		this.x = x
		this.y = y
	}
}
```
As we can see, the getters are entirely removed as they are implicit. The `public` keyword also disappeared on `x` and `y`: since the backing field is _always_ private, the implicitly-`public` accessibility is the getters'.

This example can be simplified further thanks to Kotlin's primary constructor feature: a trivial constructor can be declared directly at the class level, using `val` and `var` to declare properties of the class:
```kotlin
class Point(
	val x: Double,
	val y: Double
)
```
Note how the class doesn't have a body anymore (though it could). What is declared between the parenthesis is the [primary constructor](https://kotlinlang.org/docs/classes.html#constructors).

Here are a few examples of Kotlin properties in action:

```kotlin
class Foo {
	val a = 5 //(1)!

	var b = a + 1 //(2)!

	private val c = b++ //(3)!

	var d = c //(4)!
		private set

	val d2 //(5)!
		get() = d

	var e: Int //(6)!
		get() {
			return d + 1
		}
		set(value) {
			return value - 1
		}

	val f by lazy { 5 * 2 } //(7)!

	var g: Car by Delegates.notNull() //(8)!

	var h by Delegates.observable(0) { _, _, it -> //(9)!
		println("New value: $it")
	}
}
```

1. A simple read-only property. This declaration can be thought of as a Java `#!java final var` private field alongside a public getter.
2. A simple mutable property. This declaration is similar to a Java private field alongside a public getter and a public setter.
3. Fields in Kotlin are always private. By declaring a `private` property, we declare the visibility of its accompanying methods. Here, we have a `val`, which generates a getter but no setter, so we mean to generate a `private` getter.
4. A simple mutable property with a private field, a public getter, and a private setter.
5. A read-only property with a custom getter. Here, we start to understand why conceptualizing properties as "a private field accompanied by accessors" is incomplete: in this example, since the property's getter doesn't refer to the backing field at all, and the property doesn't have a setter, Kotlin will not generate a backing field at all. This declaration is thus equivalent in Java to a standalone getter with no accompanying setter nor field.
6. A mutable property which offers a custom getter and a custom setter. Like in the previous example, the backing field isn't used at all, so it won't be generated. This is a way to declare computed properties: to the external world, this appears to be a property, but in reality, no data is stored. A typical example of this is [`IntRange.endInclusive`](https://github.com/JetBrains/kotlin/blob/2.0.20/libraries/stdlib/src/kotlin/ranges/PrimitiveRanges.kt#L55) which is defined in terms of `IntRange.last`.
7. The `by` keyword allows [delegating](https://kotlinlang.org/docs/delegated-properties.html) a property's getter and setter to another object. This allows creating reusable property types. In this example, we use `lazy` from the standard library, which is a built-in way to declare a thread-safe lazily-computed value. For more information, see [item 83](#item-83-use-lazy-initialization-judiciously).
8. The `Delegates.notNull` delegate allows declaring a property without an initial value, without declaring it as nullable. [Learn more](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.properties/-delegates/not-null.html).
9. The `Delegates.observable` delegate allows declaring a property that triggers a callback each time its setter is called. [Learn more](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.properties/-delegates/observable.html).

Finally, the ability to use getters and setters as properties makes the vast majority of the code less cluttered.

Compare the following Java code:
```java
String firstName(User user) {
	return user.getProfile().getPersonalInformation().getNames().getFirstName();
}
```

with its equivalent Kotlin code:
```kotlin
fun firstName(user: User) =
	user.profile.perfornalInformation.names.firstName
```
This isn't a major change by itself, but it does eliminate conceptual complexity, which helps with understanding complex algorithms.

One downside is we lose on the purity of the contained information. For example, we know that writing to a Java reference is always atomic, but this isn't true for Kotlin properties, as they could have a custom writer that does something completely different. I think this is a small price to pay compared to the benefits of much easier API evolution, but I do sometimes miss it.

### Item 17. Minimize mutability

Kotlin encourages immutability for the same reasons Java does. However, Kotlin provides many additional tools to make this a reality.

The most basic tool is the simplified syntax for read-only fields. In Java, fields, parameters and other variables must be declared as `final` to opt-in:
```java
class Point {
	private final double x;
	private final double y;
	
	// …
}
```

In Kotlin, variables are declared using the `val` and `var` keywords:
```kotlin
class Point {
	private val x: Double
	private val y: Double
	
	// …
}
```
Because the read-only variant takes as much code to write as the mutable variant, many more people commit to using read-only variants by default.

Let's review the different steps recommended by Effective Java to make classes immutable:

1. **Don't provide methods that modify the object's state.**
2. **Ensure that the class can't be extended.** <br/> This is the default in Kotlin, so this point doesn't require to be mentioned.
3. **Make all fields final.** <br/> Many developers use `val` by default instead of `var`, meaning they already to this by default.
4. **Make all fields private.** <br/> As mentioned in [item 16](#item-16-use-public-accessors-not-public-fields), this is mandatory.
5. **Ensure exclusive access to any mutable components.** <br/> We will discuss this in future blog posts, but Kotlin's concurrency model (Coroutines) strongly encourages avoiding shared mutable state altogether.

Therefore, the typical Kotlin developer only needs to be careful with point 1.

As mentioned in [items 10 and 11](better-java-2.md#items-1012-overriding-equals-hashcode-and-tostring), immutability is required for proper behavior of containers like `Map` and `List`.

[//]: # (TODO: difference between List and MutableList)

### Item 18. Favor composition over inheritance

> _Inheritance is a powerful way to achieve code reuse, but it is not always the best tool for the job._

This sentence alone is probably one of the most important in the entire book, and one I often see even experienced Java developers forget. In my opinion, this is a symptom of the design of Java: inheritance is a first-class tool, other approaches aren't. When all you have is a hammer, everything is a nail.

According to the [open-closed principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle), it should be possible to extend functionality, but not modify existing functionality. In Kotlin, this is idiomatically done by providing an `interface` describing the contract, and one or more classes for the different implementations. Java also uses this approach, but also commonly uses abstract or open classes, which do not respect this principle.

Composition is the process of wrapping an existing implementation as an implementation detail of our own feature, to provide additional functionality based on that existing feature. In properly designed code, this happens when we want to implement an `interface` by modifying an existing implementation. In Java, the recommended approach is to create a _delegating class_, responsible for delegating to an existing implementation, and extending that one. We can thus override each method, and methods that aren't overridden automatically delegate to the existing implementation.

In this example, we want to create a `Set` instance that counts how many elements have been added. We are not providing a _new set implementation_, but rather are providing additional functionality on existing implementations. We could extend `HashSet`, but that would tie our counting set to `HashSet` specifically, and it makes our code very brittle for reasons explained in depth in the book. 

First, we create a delegating implementation of `Set`, which wraps an existing implementation (`upstream`) and delegates everything to it:
```java
public class DelegatingSet<E> implements Set<E> {
	private final Set<E> upstream;
	
	public DelegatingSet(Set<E> upstream) {
		this.upstream = upstream;
	}
	
	public void clear() { upstream.clear(); }
	public boolean contains(Object o) { return upstream.contains(o); }
	public boolean isEmpty() { return upstream.isEmpty(); }
	public int size() { return upstream.size(); }
	public Iterator<E> iterator() { return upstream.iterator(); }
	public boolean add(E e) { return upstream.add(e); }
	public boolean remove(Object o) { return upstream.remove(o); }
	public boolean containsAll(Collection<?> c) { return upstream.containsAll(c); }
	public boolean addAll(Collection<? extends E> c) { return upstream.addAll(c); }
	public boolean removeAll(Collection<?> c) { return upstream.removeAll(c); }
	public boolean retainAll(Collection<?> c) { return upstream.retainAll(c); }
	public Object[] toArray() { return upstream.toArray(); }
	public <T> T[] toArray(T[] a) { return upstream.toArray(a); }
	@Override public boolean equals(Object o) { return upstream.equals(o); }
	@Override public int hashCode() { return upstream.hashCode(); }
	@Override public String toString() { return upstream.toString(); }
}
```

Now that we have this delegation implementation, we can extend it and override the methods we want, continuing to delegate everything else:
```java
public class CountingSet<E> extends DelegatingSet<E> {
	private int addCount = 0;
	
	public CountingSet(Set<E> upstream) {
		super(upstream);
	}
	
	@Override 
	public boolean add(E e) {
		addCount++;
		return super.add(e);
	}
	
	@Override 
	public boolean addAll(Collection<? extends E> c) {
		addCount += c.size();
		return super.addAll(c);
	}
	
	public int getAddCount() {
		return addCount;
	}
}
```

This pattern is very powerful, but it is also quite verbose to implement. As always when that is the case, Kotlin provides a keyword to allow us to access the complex pattern in an easy way. Here, we use the `by` keyword, which represents [interface delegation](https://kotlinlang.org/docs/delegation.html). It essentially eliminates the need for writing the delegating class, as it is entirely boilerplate:
```kotlin hl_lines="3"
class CountingSet<E>(
	private val upstream: MutableSet<E>, //(1)!
) : MutableSet<E> by upstream {
	var addCount = 0 //(2)!
		private set
	
	override fun add(e: E): Boolean {
		addCount++
		return super.add(e)
	}
	
	override fun addAll(c: Collection<out E>): Boolean {
		addCount += c.size
		return super.addAll(c)
	}
}
```

1.  Properties declared in the parenthesis after the class definition are part of the primary constructor, a shorthand for Java's trivial assignment constructors. See [item 16](#item-16-use-public-accessors-not-public-fields).
2.  We use a mutable property with a private setter to avoid having both a private and a public API. See [item 16](#item-16-use-public-accessors-not-public-fields).

In this example, we can see many techniques we discussed previously come together to simplify the code.

The main downside of delegation is that the wrapped class doesn't know about the wrapper. If the wrapped class provides itself to the user (e.g. as a receiver in a callback), the receiver obtains the unwrapped instance, which doesn't have the added behavior.

### Item 19. Design and document for inheritance or else prohibit it

Class inheritance is a powerful tool that can easily have negative impacts when used in the wrong situations. In particular, inheriting from a class that wasn't designed specifically to allow it, and overriding some of its functionality, may create bugs and other unwanted behaviors.

In Java, it is advised to make it clear which classes and methods are designed for inheritance, by marking all others with `final`. Just like with fields' `final` keyword (see [item 17](#item-17-minimize-mutability)), Kotlin reverses the default: classes and methods are non-overridable by default, and the `open` keyword explicitly enables inheritance.

As mentioned in [item 18](#item-18-favor-composition-over-inheritance), providing an interface and a non-overridable class is often the superior pattern, as creating new implementations through delegation is less risky than accessing the internals of classes through overriding.

### Item 20. Prefer interfaces to abstract classes

Interfaces are much more flexible than abstract classes, and should thus be preferred in most situations.

- Just like abstract classes, interfaces can declare abstract and concrete methods.
- A class may implement as many interfaces as necessary, but may only implement one abstract class.
- Existing classes can be easily retrofitted to implement a new interface, but can usually not be retrofitted to implement an abstract class.
- Interfaces are ideal to provide additional functionality to existing objects: for example `Comparable` and `SequencedCollection`.
- Interfaces can represent non-hierarchical type frameworks.
- Interfaces simplify the creation of new implementations by combining existing ones, as mentioned in [item 18](#item-18-favor-composition-over-inheritance).

Although interfaces are much better than abstract classes at defining the implementation contract, they cannot declare state (properties with a backing field or constructors). Thus, abstract classes are still useful as _skeletal implementations_: a default implementation of an interface that provides additional utilities and checks invariants, that end-users can extend easily instead of having to verify all the interface's invariants themselves. This skeletal implementation often declares additional `protected` methods that serve as "hooks" for the final implementation. An example of this pattern can be found [here](https://gitlab.com/opensavvy/ktmongo/-/blob/1e9b0c623e54327f89a96950bd14bc2eb1306449/dsl/src/commonMain/kotlin/expr/common/Expression.kt#L148), where the interface `Expression` provides the contract, and the abstract class `AbstractExpression` handles implementing the invariants. Note also how `AbstractExpression` delegates the implementation of the `Node` interface to avoid repeating it. 

Interestingly, the creation of properties in Kotlin (see [item 16](#item-16-use-public-accessors-not-public-fields)) introduces an easy mistake that Java doesn't have: the duplication of backing fields due to inheritance. Open properties that possess a backing field do so for each layer in the hierarchy, which is much harder to do accidentally in Java. For example, let's consider the following Kotlin code:
```kotlin
abstract class A(
	open var foo: String,
)

open class B(
	override var foo: String,
) : A(foo)

class C(
	override var foo: String,
) : B(foo)
```
Do not be mistaken, there are 3 variables storing a string in this example: each property introduces its own backing field, that is almost entirely inaccessible because its getter is overridden by the child class. It may still be accessed through reflection, through the debugger, through serialization… and more worryingly, each backing field may drift: they are all initialized to the same value upon construction, but the setters are overridden, so only the leaf setter will run. The code above is similar to the following Java code, which is more visibly wasteful:
```java
abstract class A {
	private String foo;
	
	public A(String foo) {
		this.foo = foo;
	}
	
	public String getFoo() { return foo; }
	public void setFoo(String foo) { this.foo = foo; }
}

abstract class B {
	private String foo;

	public B(String foo) {
		this.foo = foo;
	}

	@Override public String getFoo() { return foo; }
	@Override public void setFoo(String foo) { this.foo = foo; }
}

final class C {
	private String foo;
	
	public C(String foo) {
		this.foo = foo;
	}

	@Override public String getFoo() { return foo; }
	@Override public void setFoo(String foo) { this.foo = foo; }
}
```

Instead, the following Java code is more often desirable:
```java
abstract class A {
	abstract String getFoo();
	abstract void setFoo(String foo);
}

abstract class B extends A {}

final class C extends B {
	private String foo;
	
	public C(String foo) {
		this.foo = foo;
	}
	
	@Override public String getFoo() { return foo; }
	@Override public void setFoo(String foo) { this.foo = foo; }
}
```
and can be produced with the following Kotlin code:
```kotlin
abstract class A {
	abstract var foo: String
}

abstract class B : A()

class C(
	override var foo: String
) : B()
```
or even more concisely using interfaces:
```kotlin
interface A {
	var foo: String
}

interface B : A

class C(
	override var foo: String
) : B
```

Finally, Kotlin introduces sealed interfaces and sealed classes, which are respectively variants of interfaces and abstract classes for which all direct implementations are known by the compiler.

### Item 21. Design interfaces for posterity

In both Java and Kotlin, interfaces can contain concrete methods. Library authors can thus add new functionality to existing type hierarchies without breaking all existing implementations. However, there are situations in which adding a concrete method in an interface still breaks an implementation, for example when that implementation already had a method with that signature but a different behavior, or when the implementation was a wrapper implementing additional behavior (which is thus not implemented for the new methods).

Although tips related to interface stability in Java also apply to Kotlin, Kotlin also provides additional tools to allow evolving APIs with minimal breakage. The most recent of them are [subclassing opt-in requirements](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/-subclass-opt-in-required/), which library authors can use to mark specific interfaces as dangerous to implement. When an end-user wants to implement them, they have to explicitly opt in to possible future breakage. Since end-users are in control of the risk they want to take, library authors have more freedom to evolve existing interfaces.

Another tool is to sidestep the issue entirely: Kotlin has extension functions, which can declare additional behavior built from the existing behavior. These new functions are invoked by end-users like default methods, but are not actually part of the interface and are not impacted by the way existing implementations are written. However, implementations cannot override them either, so they cannot customize their behavior. To learn more about extension-oriented design, read [this blog post](https://elizarov.medium.com/extension-oriented-design-13f4f27deaee).

### Item 24. Favor static member classes over nonstatic

Java has inner classes:
```java
class Outer {
	class Inner {}
}
```
and nested classes:
```java
class Outer {
	static class Nested {}
}
```

The latter is nothing more than a regular class placed within another class; the nested class is namespaced within the outer class, but is not otherwise different in any way from regular classes.

Inner classes are however quite different: they possess a hidden reference to a specific instance of the outer class, allowing them to access any of its fields as if they were their own. This hidden reference is often forgotten by developers, with sometimes surprising results. For example, in the Android world, creating an inner class in an `Activity` and keeping a reference to it blocks the garbage collector from freeing the `Activity` instance, which is a very large object (see [this blog post](https://pszklarska.medium.com/catch-leak-if-you-can-608a99537d8a)).

In general, it is recommended to Java developers to almost always use nested classes, only using inner classes when their special feature is needed. As always, the Kotlin team knew about the usage patterns and reversed the defaults: classes are nested by default, and a special keyword is used to mark them as inner classes.

Kotlin inner classes:
```kotlin
class Outer {
	inner class Inner
}
```
and nested classes:
```kotlin
class Outer {
	class Nested
}
```

Nested and inner classes otherwise retain the same functionality and usage patterns in both languages.

Since the book used this item to detail the use-cases for anonymous and local classes as well, they bring another difference between Java and Kotlin: Java doesn't allow anonymous classes that implement multiple interfaces or that extend a class and implement an interface. However, Kotlin can!
```kotlin
interface Foo
interface Bar
abstract class Baz

val o = object : Baz(), Foo, Bar {
	/* …abstract class body… */
}
```

### Item 25. Limit source files to a single top-level class

The main reason the book recommends avoiding multiple top-level classes in a single file is because of the behavior of `javac` when some files are not provided in the command-line. Modern projects always use a dedicated build tool (like Maven or Gradle) which will not make these mistakes, removing this risk.

Kotlin has many more types of declarations (classes, interfaces and abstract classes, enums, but also top-level properties and functions, type aliases, value classes, data classes…) which are often declared in a small number of lines of code, which makes more tempting to put multiple of them in a single file.

It is still recommended to limit one file to one _unit of understanding_, but Kotlin developers generally understand this as being potentially larger than a single class. For example, it is considered good practice to put extension functions that relate to a class at the end of the same file:
```kotlin
class Foo {
	// …
	
	fun add(message: String) {
		// …
	}
}

fun Foo.addAll(messages: Iterable<String>) {
	for (message in messages) {
		add(message)
	}
}
```

Note that when it comes to Java interoperability, the file in which a top-level function is placed is *not* innocent! Top-level functions and properties are compiled on the JVM to static members of a class named after the file, suffixed by `Kt`. In general, this means moving top-level functions and properties to another file in the same package is a binary breaking change (but a source-compatible one), which library authors should be careful with. Moving a top-level class or object to another file is fine, however.

As another example, when implementing the interface+abstract class pattern described in [item 20](#item-20-prefer-interfaces-to-abstract-classes), it is typical to put both in a single file, and each implementation in their own files;
```kotlin
interface Foo {
	// …
}

abstract class AbstractFoo {
	// …
}
```

Of course, the ability to put as many symbols as we want in a single file can make the opposite pattern possible, where a single file grows to unmanageable scales. Files in the Kotlin standard library can often contain 10–30 declarations, which I would not recommend in regular projects.

***

In this chapter, we went over the fundamentals of software architecture in Java and how Kotlin adapts and adopts the best practices. However, this is not nearly all that Kotlin has to offer! Kotlin provides type aliases, data classes, value classes, functions as a first-class citizen and the DSL pattern, which all fundamentally change the way software is modeled.

Going over everything that exists isn't possible in a single post, and the goal of this format was to compare what inspiration Kotlin took or didn't take from Java, so other strategies couldn't be integrated easily.

Next week's post will probably the most challenging to write for me: we will be talking about generics and variance. Have a great week!
