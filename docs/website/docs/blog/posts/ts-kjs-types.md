---
date: 2024-09-05
slug: ts-kjs-types
tags:
  - JS/TS
  - Kotlin
---

# Typing the web: TypeScript & KotlinJS

When I talk about KotlinJS, people often ask me “Why not just TypeScript?”. This article compares the type system and standard libraries to understand how different the languages are.

<!-- more -->

## A bit of context

**JavaScript** is, and will stay, the primary language of the web. JavaScript is famously dynamic, permissive, and quite often surprising. Over the years, many languages have attempted to reign it in a bit. I think it would be fair to say that TypeScript is the most successful attempt so far.

**TypeScript** is a language developed by Microsoft that adds an optional type-system on top of JavaScript. TypeScript code is otherwise identical to JavaScript. TypeScript is compiled to JavaScript by simply removing all type information.

**Kotlin** is a language developed by JetBrains. Originally popular for its interoperability with Java in the desktop and backend worlds, it has grown massively in the Android ecosystem and became its official language. Since Kotlin 1.2 (late 2017), Kotlin supports transpiling to JavaScript and generating TypeScript definitions (stable since Kotlin 1.8, 2023).

There are many differences that we could discuss: build tooling, ecosystem, interoperability… Today, though, I want to focus on the language itself: the type system and the standard library.

## A difference in philosophy

At the root of it all, there is a different vision. TypeScript wants to be a general replacement of JavaScript: any JavaScript code should be portable to TypeScript with minimal changes and minimal downsides. Kotlin wants to be a single, unified language that supports multiple platforms, one of which being the web.

This difference in philosophy is the source of almost all differences. TypeScript must always retain JavaScript's runtime behavior, whereas Kotlin cares more about being coherent with other platforms. Let's look at a simple example: array concatenation.

### Array concatenation

**Array concatenation** is the process of combining two arrays into a larger array which contains all elements of both arrays. Concatenation is a functionality available to most “container” types: arrays, strings, etc.

In JavaScript, TypeScript and Kotlin, string concatenation is written with the `+` operator:
```javascript
"foo " + "bar" // "foo bar"
```

In Kotlin, array concatenation is written very similarly:
```kotlin title="Kotlin"
listOf(1, 2) + listOf(3, 4) // [1, 2, 3, 4]
```

??? note "Why `listOf`?"
    Kotlin makes a difference between arrays and lists: arrays are low-level primitives which may not be resized, and lists are high-level primitives which have various implementations that may or may not be mutable: array-based, linked lists…

    This difference will be familiar to Java or C++ developers. In everyday life, users interact with lists and very rarely with arrays, which is the reason this article uses lists. Though for this specific example, using arrays would be identical, since arrays can also be concatenated with the `+` operator.

However, in TypeScript, the equivalent code is a compilation error:
```typescript
[1, 2] + [3, 4]
```

Instead, the expected code is the following:
```typescript title="TypeScript"
[...[1, 2], ...[3, 4]] // [1, 2, 3, 4]
```

It may be surprising to many users that `+` is not allowed there. After all, JavaScript is notorious for allowing operators to be called with many different data types, so why wouldn't `+` work on arrays?
```javascript title="JavaScript"
{} + {}   // NaN
[] + []   // ''
[] + {}   // [object Object]
{} + []   // 0
```

Well, the reason is more or less under our nose. `+` in JavaScript never means array concatenation. In fact, it already means something different:
```javascript title="JavaScript"
[1, 2] + [3, 4]  // '1,23,4'
```

TypeScript wants to be a superset of JavaScript, and thus cannot change this behavior, so instead it forbids it.

This example is a bit contrived, but it does illustrate the major difference between TypeScript and KotlinJS: TypeScript retains the behavior of JavaScript, even it must forbid the use of a very convenient operator. KotlinJS only cares about being self-coherent, and doesn't inherit any of JavaScript's quirkiness.

### Number types

Similarly, TypeScript follows JavaScript with a single number type, `number`, that represents floating-point values. Similarly to most dynamically-typed languages, this makes JavaScript unfit for storing precise values (e.g. money amounts) because of [floating-point imprecision](https://en.wikipedia.org/wiki/Floating-point_error_mitigation). TypeScript, however, is fairly unique at being a statically-typed language suffering from this problem.

Surprisingly enough, JavaScript does have a [`BigInt` type](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt), but no `BigDecimal`.

Kotlin supports all the standard JVM number types: `Byte`, `Short`, `Int` and `Long` (integers, respectively 8, 16, 32 and 64 bits), `Float` and `Double` (floating-point numbers, respectively 32 and 64 bits). Kotlin also supports unsigned equivalents: `UByte`, `UShort`, `UInt` and `ULong`.

At the time of writing this article, Kotlin doesn't have its own `BigInt` or `BigDecimal` types, though it can of course use JavaScript's `BigInt`.

You may wonder how these types are implemented by KotlinJS. JavaScript's `number` is safe for integer storage (as long as not a single floating-point operation is ever executed) until 2^53^-1, meaning all integer types except `Long`/`ULong`, as well as all floating-point types, can be implemented with simple bound checks. The `Long` and `ULong` types are emulated by a class that is part of the KotlinJS standard library, which does make them slower but is an acceptable cost towards multiplatform coherence.

## Trusting the user

Another major difference is the trust given to the user. In TypeScript, the compiler _assumes that the user is right_, whereas Kotlin _assumes that the user doesn't know_. The most striking behavior is the implementation of the `as` operator. In both languages, `as` is the cast operator, which allows changing its compile-time type without impacting its run-time value.

```typescript title="TypeScript"
const a = foo as Car
```

```kotlin title="Kotlin"
val a = foo as Car
```

In both languages, the variable `a` will have the compile-time type `Car`. The run-time behavior if `foo` indeed stores a `Car` (or any of its subtypes) is identical in both languages. However, the opposite situation is very different.

If `foo` contains something other than a `Car`, TypeScript will continue executing the code as if nothing happened, as no run-time check is actually performed. Again, TypeScript wants to behave like JavaScript, so `as` cannot be anything more than be a compiler hint.

In contrast, Kotlin will actually insert a run-time check, and will throw a `ClassCastException` if the run-time value is not a subtype of `Car`. Kotlin treats written code as the user's _goal_, and will insert checks to ensure the behavior of the program matches this goal. In practice, this means bugs are caught much earlier, since exceptions are thrown as soon as the values stop making sense.

You may say this example is a bit unfair: `as` in TypeScript is intended to be a very different concept, and is only intended to be used when we are absolutely sure that it is true. However, I don't find the real-life usage to match this description. The best example may be that [TypeScript's documentation](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-assertions) itself lists the very first usage example as:
```typescript title="TypeScript"
const myCanvas = document.getElementById("main_canvas") as HTMLCanvasElement;
```
In this example, if the element isn't a `canvas`, or if no element is thrown, an error will only be thrown once we try to access a field that doesn't exist, later on the program:
```text
Cannot read properties of undefined
```
In contrast, KotlinJS will throw a `ClassCastException` if the returned element is `null`, `undefined` or another type than `HTMLCanvasElement`:
```kotlin title="Kotlin"
val myCanvas = document.getElementById("main_canvas") as HTMLCanvasElement
```

For the curious, we can look at the JavaScript code generated by the KotlinJS compiler:
```javascript title="JavaScript"
var tmp = document.getElementById('main_canvas');
if (!(tmp instanceof HTMLCanvasElement)) 
	THROW_CCE();
```

### Incoherent casts

Both languages will warn the user at compile-time on incoherent casts, such as:
```typescript title="TypeScript"
foo as number as string
```
```kotlin title="Kotlin"
foo as Int as String
```
No value in either language can be both an integer and a string, so this code is necessarily incorrect.

### Unchecked casts

Kotlin type parameters are erased at compile-time: that is, at run-time, `List<String>` is completely indistinguishable from a `List<Any>` that just so happens to only contain `String`s at that particular moment.

In practice, this means that casts on type parameters are unchecked: the compiler cannot insert run-time checks to verify them. For example:
```kotlin
fun <T> foo(elements: List<T>) {
	elements as List<String> //(1)!
	// …
}
```

1.  Unchecked cast: `List<T>` to `List<String>`

In Kotlin, this is a compile-time warning. At run-time, any value will pass through the test, and may throw an exception later on usage. In this particular case, Kotlin behaves exactly the same as TypeScript, with the addition of a compile-time warning.

### An escape hatch

Sometimes, unsafe casts are indeed required. For these situations, both languages have a special type to force the compiler to accept everything:
```typescript title="TypeScript"
const a = 2 as unknown as string;
```

```kotlin title="Kotlin"
val a = 2 as dynamic as String
```

In practice, this is often used when interacting with JavaScript libraries that may not declare a type at all.

## The language

Both TypeScript and KotlinJS, at the end of the day, are running on the JavaScript engine. And the JavaScript engine can be very wild. However, since Kotlin brings its own standard library, it can follow its own paradigm. TypeScript uses JavaScript's, and thus cannot. As a result, most code written in TypeScript interacts with JavaScript directly, as most of the ecosystem is written in JavaScript. Most KotlinJS code interacts with standard Kotlin types, only calling true JavaScript at the edge.

A direct consequence is that the quality of the bindings matter a lot of more for TypeScript than it does for KotlinJS. Since a high proportion of code called by TypeScript isn't actually written in TypeScript, it's very easy for the called code not to respect TypeScript's invariants. Since KotlinJS code mostly calls Kotlin code, the invariants are guaranteed.

### Null everywhere is a pain

In a language that forces compile-time safe access to nullable values, it's very easy to go too in-depth into making everything potentially dangerous nullable. If you do, writing code becomes painful, as everything needs to be manually checked by the user. One of these situations is array access: should an array element be nullable?

If you make array elements nullable, you end up with code similar to this:
```kotlin
for (i in 0..array.size) {
	val element = array[i]
	if (element != null) {
		println(element)
	}
}
```

This type of checks is clearly redundant: since we're iterating over the size, we can't access out-of-bounds elements _by construction_, so why should we check for nullability on each access? It also creates confusion if the array elements are nullable, as we can't distinguish between "we're out of bounds" and "the element itself is `null`".

Kotlin makes a decision based on usage patterns: sequential arrays (`List`) throw an exception on out-of-bounds (because you most likely know whether you're in-bounds or not), whereas associative arrays (`Map`) returns `null` if the element is absent:
```kotlin title="Kotlin"
val list = listOf(1, 2, 3)
val map = mapOf("foo" to 1, "bar" to 2)

val a: Int = list[1]       // 2
val b: Int? = map["bar"]   // 2
val c: Int = list[4]       // ArrayOutOfBoundsException
val d: Int? = map["baz"]   // null
```

Everywhere in this article, each time I show a difference between approaches, I try to find the rationale for why it was done this way, so we can all learn and not just point at "ahah different is weird". For this very specific case though, I haven't been able to find it, so sorry in advance. If you know the reason, please get in touch.

What TypeScript does to avoid this array element nullability issue is… lie. TypeScript tells you that the value can't be nullable:
```typescript title="TypeScript"
const arr = [1, 2, 3]

const e = arr[2]
```
In this example, the inferred type of `e` is `number` (which is the non-nullable version, nullable would be `number | undefined` or `number | null`). But that's… simply not true. If you access an element out of bounds, `undefined` will be returned, _even though the type says it's not possible_.
```typescript title="TypeScript"
const arr = [1, 2, 3]
const e = arr[7]  // The type is 'number' (non-nullable), but the value is 'undefined'
```

This feels like a clash between JavaScript and TypeScript: TypeScript wants to behave as if array access is always safe, but it does so at the cost of removing null-safety, even though that was one of the main advantages of the language. As a result, it's hard to make this code safe: what if you really want to rely on the returned `undefined` value, but the tooling will warn that it's an impossible situation (even though it clearly isn't). But if you're not careful, you can have a `undefined` creeping up in places you did not expect, since the language itself tells you it can't happen.

The fact that the standard library itself isn't `null`-safe is one of the reasons why many JavaScript developers simply don't trust TypeScript, even more so if they are also accustomed to actually `null`-safe languages, like Kotlin, Rust, Swift, Scala, etc.

### Evolution of the standard library

TypeScript relies on JavaScript's standard library. JavaScript's standard library is embedded into browsers, and can only evolve when all browser vendors agree to change it (or when Chrome does something and everyone has to follow). Even when a function finally makes it to the JavaScript standard, it may take some time until it reaches TypeScript.

One good example of this is `groupBy`: although [widely available](https://caniuse.com/?search=groupBy), it is still not yet part of the TypeScript standard library.

In contrast, Kotlin bundles its standard library (similar to what JavaScript users would do with something like [babel](https://babeljs.io/)), meaning that all new functions are always available right away, even when running on old devices, and they behave the same on every browser. This also gives Kotlin the power to actually deprecate and remove functions that end up being footguns, which makes it much more likely that the language will continue to be nice to use in the future.

***

Overall, TypeScript does fill a niche with being slightly safer than JavaScript while avoiding runtime costs like increased bundle size. KotlinJS isn't far behind, and there are many existing websites that use it without major performance differences, especially when compared to the average website. 

My main gripe with TypeScript is that I just don't trust it. I've been bitten too many times by declared types that just don't match what the application does at runtime, that I've started to take a habit of actually testing edge cases myself just to see if the function actually behaves like it says it does. And more than once, I found that it doesn't.

At the same time, I am spoiled by the Kotlin standard library, which is incredibly vast and offers almost everything you could ever want from a programming language, while still growing year after year.

I suspect many developers of larger web applications, especially those from the backend or mobile worlds, would like KotlinJS. Once you've passed the tooling hurdle, everything is just so much easier.
