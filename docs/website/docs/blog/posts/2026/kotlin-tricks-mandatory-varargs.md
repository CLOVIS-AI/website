---
date: 
  created: 2026-03-09
slug: kotlin-mandatory-varargs
tags:
  - Kotlin/Tricks
---

# Kotlin tricks: Mandatory varargs

Did you know there are a few different ways to make Kotlin varargs mandatory?

<!-- more -->

I usually write long-form articles, but that makes it hard to stick to the weekly schedule.
With this post, I want to experiment with shorter articles that focus on a single Kotlin trick.
Let me know what you think!

## What's a vararg?

Kotlin has _variadic arguments_, written `vararg`.

!!! tip "Pedantry corner"
    A **parameter** is the thing we declare in a function signature.
    ```kotlin
    fun foo(
        a: Int,  // It's a parameter!
    ) = TODO()
    ```

    An **argument** is the thing we pass to a function call.
    ```kotlin
    foo(1)  // It's an argument!
    ```

    This is why we say "named arguments" but "parameters with a default value".

Kotlin, like many other languages, has special parameters, marked `vararg`, which can be specified multiple times when calling a function.

```kotlin
fun foo(vararg names: String) {
	for (name in names) {
		println(name)
	}
}
```

On the call-site:
```kotlin
foo(
	"Alice",
	"Bob",
	"Charlie",
)
```

Here, we can specify as many—or as few—arguments as we want.

This includes specifying no arguments at all:
```kotlin
foo()
```

Sometimes, however, we don't want to allow specifying no arguments at all.

For example, what should this method do if no arguments are specified?
```kotlin
fun <T> List<T>.containsAll(vararg elements: T): Boolean
```

[Vacuous truthers](https://en.wikipedia.org/wiki/Vacuous_truth) will demonstrate it should return `true`.

!!! tip "Why should it return `true`?"
    If we call `listOf(1, 2, 3).containsAll()`, all the specified elements (of which there are none) are in the list.

This reasoning is useful when we can't detect at compile-time whether the call makes sense. At runtime, we need to behave in _some_ way, 
and throwing an exception may be more confusing than following mathematics.

In the case of `containsAll`, however, we _can_ detect at compile-time whether the call is vacuous: if there are no arguments, it probably doesn't make sense.
Therefore, it is legitimate to want to forbid it.

## Using a regular parameter, followed by a vararg

When faced with this problem, most developers will add a mandatory vararg parameter:

```kotlin
fun <T> List<T>.containsAll(element: T, vararg elements: T): Boolean
```

At the call-site, this does work:
```kotlin
listOf(1, 2, 3).containsAll()      // Doesn't compile!
listOf(1, 2, 3).containsAll(1)     // Compiles
listOf(1, 2, 3).containsAll(1, 2)  // Compiles
```

There is, however, a downside. Because the first element is out of the `vararg`, we cannot treat all elements as a single list.

We can either treat it as its own specific case, which has the downside that our code becomes more complex: 
```kotlin
fun <T> List<T>.containsAll(element: T, vararg elements: T): Boolean {
	return this.contains(element) &&
		elements.all { this.contains(it) }
}
```
Or, we can reconstruct the list ourselves:
```kotlin
fun <T> List<T>.containsAll(element: T, vararg elements: T): Boolean {
	val allElements = listOf(element) + elements
	return allElements.all { this.contains(it) }
}
```
However, this has a big performance cost: `listOf(element) + elements` creates a copy of the `elements` array.

If the function is only called occasionally, this may be acceptable, but a function like `containsAll` may be called within hot loops.

Another downside of this approach is that we cannot use the spread operator, though there are other good reasons to avoid it.

## The real trick

In Kotlin, we can often combine language features to solve other problems. Here, our goal is to forbid a specific call-site. There is already a tool for this!

We can create an overload that takes no parameters, and forbid its usage.

```kotlin
@Deprecated(
	message = "List<T>.containsAll() requires at least one argument.",
	level = DeprecationLevel.ERROR,
)
fun <T> List<T>.containsAll(): Nothing =
	throw UnsupportedOperationException("containsAll() with no arguments is not allowed")

fun <T> List<T>.containsAll(vararg elements: T): Boolean =
	elements.all { this.contains(it) }
```

The advantages are:

- The main function can be implemented idiomatically.
- The no-argument overload is clearly identified, verified at compile-time by the compiler, and we can customize the compiler error message to make it explicit to callers.

Note that a user could still explicitly spread an empty array. In this case, the principles of vacuous truth naturally take over.

It works because:

- When the compiler sees a call with no arguments, it always resolves it to the no-argument overload, even if calling it is forbidden. The compiler will not try to fall back to the `vararg` overload.
- `level = DeprecationLevel.ERROR` changes the default of `WARNING` to force a compiler error.

If you want to create a function that takes a minimum of two arguments, you can create a forbidden zero-argument overload as well as a forbidden one-argument overload.
The same principle extends to any number of arguments, though I've never had the case for more than two.

What do you think?

Do you recognize this as a legitimate language feature or as a hack?

***

Interestingly, I've personally come around to changing my mind on this. When I requested mandatory varargs ([KT-55890](https://youtrack.jetbrains.com/issue/KT-55890)), I thought this was a weird hack, and a dedicated language feature would have been a better solution.

Now that I follow the process of designing Kotlin closer, I find myself having the opposite opinion. As Kotlin becomes more complex, I often prefer slightly changing existing language features so they can be combined to solve more problems, rather than adding new dedicated language features that must find their own place among all others.

Next week I will write about the canonical parameter order for Kotlin. Subscribe with any of the buttons in the margin to not miss it!
