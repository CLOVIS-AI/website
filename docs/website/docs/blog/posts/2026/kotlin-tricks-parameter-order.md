---
date: 
  created: 2026-06-08
slug: kotlin-parameter-order
tags:
  - Kotlin
---

# In which order should Kotlin parameters be declared?

There are so many kinds of parameters! Is there an order in which they should be declared?

<!-- more -->

## Parameters in Kotlin

The Kotlin programming language itself doesn't give strong rules on the order in which parameters should be declared. Yet, there is an order that is better than the others.

Here are the different kinds of parameters that exist:

- Regular (mandatory) parameters.
- Parameters with a default value.
- `vararg` parameters.
- Lambda parameters.

Let's review each of them and why their order matters.

## Positional and named arguments

Kotlin offers two ways to specify arguments:
```kotlin
fun foo(
	a: Int,
	b: String,
)

// Positional
foo(2, "Bob")

// Named
foo(b = "Bob", a = 2)
```

When using positional syntax, the order of the arguments must match the order of the parameters.
However, when using named syntax, the order does not matter.

It is allowed to mix both syntaxes in a single call as long as the order of the arguments matches the order of the parameters.
```kotlin
fun foo(
	a: Int,
	b: String,
	c: Boolean,
)

foo(1, "2", true)

foo(1, "2", c = true)

foo(1, b = "2", true)
```
As soon as a named argument doesn't follow the order of the parameters, positional arguments become forbidden.
```kotlin
foo(1, c = true, "2")
```
!!! failure
    Compilation error: mixing named and positional arguments is not allowed unless the order of the arguments matches the order of the parameters.
    ```kotlin
    foo(1, c = true, "2")
                     ^
    ```

In that case, all remaining arguments must be named.
```kotlin
foo(1, c = true, b = "2")
```

Therefore: **parameters intended to be used with named syntax should be placed after parameters intended to be used with positional syntax**.

## Parameters with a default value

A parameter can have a default value. In that case, it is optional on the call site:
```kotlin
fun foo(
	a: Int,
	b: String = "default",
	c: Boolean = true,
)

foo(a)
```

If a method provides optional parameters, users may want to specify some but not others:
```kotlin
foo(a, "bob")
```
Because positional arguments must be ordered, it is not possible to positionally specify `c` but not `b`.

Thus, default parameters are often written with named syntax:
```kotlin
foo(a, c = false)
```

Alternatively, if a mandatory parameter is placed after a default parameter, a user who wants to use positional syntax is forced to specify the default parameter too:
```kotlin
fun foo(
	a: Int,
	b: String = "default",
	c: Boolean = true,
)

foo(a, "default", false)
// I cannot specify 'c' positionally without specifying 'b'
```

Therefore: **optional parameters should be placed after mandatory parameters**.

## Variadic arguments

The `vararg` keyword declares a parameter that can be specified multiple times:
```kotlin
fun display(
	vararg items: Any,
) {
	for (item in items) {
		println(item)
	}
}

display("123", 5, 7)
```

On the call site, the vararg is not delimited. Because of this, a function can only have a single `vararg` parameter. 

Additionally, positional arguments are forbidden after a `vararg` argument:
```kotlin
fun mandatoryAfterVararg(
	vararg items: Int,
	mandatory: String,
) {
	// …
}

mandatoryAfterVararg(5, 7, 9, "foo")
```
!!! failure
    Compilation error: no value passed for parameter '`mandatory`'.
    ```
    mandatoryAfterVararg(5, 7, 9, "foo")
                                       ^
    ```

This is because all positional parameters that appear after the `vararg` parameter are considered part of the `vararg`, even if the type is different.

Therefore: **all positional parameters must appear before the `vararg` parameter**.

## Lambda parameters

Kotlin has the capability of declaring functions that look like operators. For example:
```kotlin
repeat(5) {
	println("Hello!")
}
```
`repeat` is not a Kotlin keyword, it is a regular function in the standard library.
This code is identical (but preferred) to the explicit variant:
```kotlin
repeat(5, {
	println("Hello!")
})
```

In Kotlin, a lambda parameter that is the very last parameter can be specified outside the parentheses, which highlights that lambda as the primary action of the method.

For example, in the following method, `update` is the primary action, and `options` is a non-primary configuration:
```kotlin
users.updateMany(
	options = { limit(10) },
) {
	User::isAdult set false
}
```

To be able to highlight the primary action using the trailing lambda syntax: **the primary action of a method must be placed as the very last parameter**.

Other non-primary actions are placed like regular mandatory or optional parameters.

## Compose content and slots

In addition to regular lambdas, Compose adds special `@Composable` lambdas, which represent nested components. These components are displayed within the method.

When a composable method has a primary content, that lambda should be treated like the primary action:
```kotlin
Button(action = { println("Clicked") }) {
	Text("Hello world!")
}
```

This choice makes it easier to read the code, as the `{}` brackets have the same structure as the final UI. Compare the following unidiomatic code which doesn't follow this rule:
```kotlin
Button(content = { Text("Hello world!") }) {
	println("Clicked")
}
```
If written this way, the UI as a whole becomes noticeably harder to read.

Therefore: **if a composable method has a primary content, it should be placed at the very last parameter**.

A composable method may accept multiple such lambdas, called "slots", in different places of its UI. For example:
```kotlin
@Composable
fun TwoColumns(
	left: @Composable () -> Unit,
	right: @Composable () -> Unit,
)
```
In the case where each slot has equivalent importance, we want to force users to use named syntax:
```kotlin
// Readable
TwoColumns(
	left = {
		// …
	},
	right = {
		// …
	}
)

// Not as readable
TwoColumns({
	// …
}) {
	// …
}
```
The trailing lambda highlights the `right` column against the `left` column. Since both are supposed to be equivalent, this is misleading.

We can forbid users to write this code by ensuring that none of the lambda parameters are placed as the last parameter.
Therefore: **non-primary lambdas should be placed before at least one other parameter**.

<hr>
Together, these rules bring us to a single order.

## The best order

A function should always declare parameters in the following order:

```kotlin
fun FriendsList(
```

1. Mandatory parameters.
```kotlin
currentUser: User,
```
2. Mandatory lambda parameters, including Compose mandatory slots.
```kotlin
filterFriend: (User) -> Boolean,
befriend: @Composable (User) -> Unit,
```
3. The vararg.
```kotlin
vararg friends: User,
```
4. Optional lambda parameters, including Compose optional slots.
```kotlin
key: (User) -> Any = { it },
```
5. Optional parameters.
```kotlin
maxNumberOfItems: Int = 10,
```
6. The main lambda parameter, or the Compose content.
```kotlin
content: @Composable () -> Unit,
```

```kotlin
)
```

I'm sure there are cases where this order is not perfect, but I don't think I have encountered one yet. If you have, please tell me!
