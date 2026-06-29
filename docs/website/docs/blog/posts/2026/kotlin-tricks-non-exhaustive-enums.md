---
date: 
  created: 2026-06-29
slug: kotlin-non-exhaustive-enum
tags:
  - Kotlin/Tricks
---

# Kotlin tricks: Non-exhaustive enums

Enums are an exhaustive set of values. But what if you suspect you may want to add new entries in the future?

<!-- more -->

## What's an enum?

An enum is a kind of class that has a compile-time fixed set of instances.

```kotlin
enum class Example {
	Foo,
	Bar,
	;
}
```

Because an enum is a class, it can have properties, have constructors, implement interfaces, etc.
However, it already extends the abstract class `kotlin.Enum`, so it cannot extend other classes.

```kotlin
enum class VectorType(val byte: Byte) {
	Bytes(0x03),
	Floats(0x27),
	Booleans(0x10),
	;
	
	fun parse(content: ByteArray): Vector { /* … */ }
}
```

Methods and properties are added to each instance of the enum:
```kotlin
VectorType.Booleans.parse(/* … */)
```

Each instance can even override members declared in the enum body:
```kotlin
enum class VectorType(val byte: Byte) {
	Bytes(0x03) {
		override fun parse(content: ByteArray): Vector { /* … */ }
	},
	Floats(0x27) {
		override fun parse(content: ByteArray): Vector { /* … */ }
	},
	Booleans(0x10) {
		override fun parse(content: ByteArray): Vector { /* … */ }
	},
	;

	abstract fun parse(content: ByteArray): Vector
}
```

In general, Kotlin enums are very powerful and can use almost all features of regular classes.

!!! note "A note on code style"
    Both of these examples are identical from the point of view of the compiler:
    ```kotlin
    enum class Example {
        Foo,
        Bar,
        ;
    }
    ```
    ```kotlin
    enum class Example {
        Foo,
        Bar
    }
    ```
    I use UpperCamelCase because enums entries are instances of the class, so they follow the rules of `object` declarations.
    This is allowed by the [official code style](https://kotlinlang.org/docs/coding-conventions.html#property-names), and the preferred style of many libraries, including Ktor and Compose.

    I prefer adding the [trailing comma](https://kotlinlang.org/docs/coding-conventions.html#trailing-commas) because it makes VCS diffs smaller and easier to understand: when a new element is added, the line `Bar` doesn't need to be modified to add a comma. This is recommended by the official code style.

    The semicolon (`;`) is used as a delimiter between the enum entries (instances) and the enum members (constructors, properties, methods, nested types…).
    Similarly to the trailing comma, I prefer to always write it to decrease diffs when a member is indeed added.

    Fun fact, when there are enum members, it is the only place in the entire Kotlin language where a semicolon is mandatory.

## The problem

Because the compiler knows the different entries, it can complain if the user failed to handle one:
```kotlin
enum class Example {
	Foo,
	Bar,
	Baz,
	;
}

val e = Foo

val a = when (e) {
	Foo -> 12
	Bar -> 13
}
```
!!! failure "Compilation error"
    ```text
    error: 'when' expression must be exhaustive. Add the 'Baz' branch or an 'else' branch.
    val a = when (e) {
            ^
    ```

Because it isn't possible to add new enum entries after compilation, the compiler knows all entries it knows of _are_ all the enum entries that exist, and can confirm that code handles all possible cases:
```kotlin
val a = when (e) {
	Foo -> 12
	Bar -> 13
	Baz -> 3
}
```
Here, the compiler is happy, because this code is valid no matter what.

However, sometimes, as library authors, we know that there is a chance that we may add new enum entries in the future.
At that time, that would be a major breaking change for all users: all their exhaustive conditions wouldn't be exhaustive anymore.

Ideally, we would be able to force users to handle the 'unknown' case, which corresponds to not-yet-added enum entries.

## The trick

If you read [my previous Kotlin Tricks post](kotlin-tricks-mandatory-varargs.md), which also revolved around making some things impossible for callers, you may expect that the solution is the same.

We can exploit deprecation errors to add a phantom enum entry that users cannot write. Because they cannot write it, but `when` must always be exhaustive, the compiler forces them to make it exhaustive somehow; the only way is to use `else`. We can even customize the error message to make it easy to understand to people who don't know the trick:
```kotlin
enum class HttpMethod {
    Get,
    Post,
    Patch,
    Delete,
    Head,

    @Deprecated(
        message = "This enum is non-exhaustive. Additional entries may be added in the future. Ensure you always use 'else ->' in any 'when' conditions to prepare for future additions.",
        level = DeprecationLevel.ERROR,
    )
    NonExhaustive,
    ;
}

val e = HttpMethod.Delete

val a = when (e) {
    HttpMethod.Get -> 1
    HttpMethod.Post -> 2
    HttpMethod.Patch -> 3
    HttpMethod.Delete -> 4
    HttpMethod.Head -> 5
    HttpMethod.NonExhaustive -> 6
}
```
!!! failure "Compilation error"
    ```text
    error: 'enum entry NonExhaustive: HttpMethod' is deprecated. This enum is non-exhaustive. Additional entries may be added in the future. Ensure you always use 'else ->' in any 'when' conditions to prepare for future additions.
    HttpMethod.NonExhaustive -> 6
               ^
    ```

Instead, the following code does compile:
```kotlin
val a = when (e) {
    HttpMethod.Get -> 1
    HttpMethod.Post -> 2
    HttpMethod.Patch -> 3
    HttpMethod.Delete -> 4
    HttpMethod.Head -> 5
    else -> 6
}
```

This way, [if a new HTTP method is added in 2026](https://www.rfc-editor.org/info/rfc10008/), you don't have to break all code from your existing users.

## Limitations & alternatives

This trick isn't perfect.

1. We are, in fact, adding a new enum entry. Even if it cannot appear in code, it can appear in other places, like in `HttpMethod.entries`.
   If you are writing an enum that users want to iterate over, it's probably best to avoid using this trick.
   Note that you may need more work to support serialization, which is usually handled by default for enums.
2. What will users do if they don't know what to do in the case of an unknown enum entry? They will probably through a runtime error.
   Is that really better than code not compiling when they upgrade the version? As a library author, you have to factor this into your decision.

One alternative is to create a manual enum, which implements its own `entries`:
```kotlin
abstract class HttpMethod private constructor() {
   data object Get : HttpMethod()
   data object Post : HttpMethod()
   data object Delete : HttpMethod()

   companion object {
       val entries = listOf(Get, Post, Delete)
   }
} 
```
However, it doesn't provide the full functionality of `enum class`. For example, `enum class` instances are `Comparable` (can be sorted, can be compared).
Also, it creates a new type for each entry, which you may want to avoid.

You can avoid creating a type for each entry by using anonymous objects:
```kotlin
abstract class HttpMethod private constructor() {
	companion object {
		val Get = object : HttpMethod() {
			override fun toString(): String = "Get"
		}
		val Post = object : HttpMethod() {
			override fun toString(): String = "Post"
		}
		val Delete = object : HttpMethod() {
			override fun toString(): String = "Delete"
		}

		val entries = listOf(Get, Post, Delete)
	}
}
```
You must now explicitly implement `toString()`, but you still have the ability to declare common members and override them differently for each entry.

If you don't need the ability to override members, you can simplify this further:
```kotlin
@JvmInline
value class HttpMethod(val name: String) {
	companion object {
		val Get = HttpMethod("GET")
		val Post = HttpMethod("POST")
		val Delete = HttpMethod("DELETE")
        
		val entries = listOf(Get, Post, Delete)
	}
}
```
This is the implementation [chosen by Ktor](https://github.com/ktorio/ktor/blob/3.5.1/ktor-http/common/src/io/ktor/http/HttpMethod.kt#L17).
!!! example "The future"
    With the future addition of the `companion` block, this entire example will have a single allocation: the list.

What did you think? Will you use this trick?
