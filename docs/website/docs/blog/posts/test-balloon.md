---
date: 
  created: 2025-06-02
slug: test-balloon
tags:
  - Kotlin
---

# Lifting Kotlin testing

There's a lot that goes into testing, but we rarely think of the most basic aspect: syntax. 

<!-- more -->

## Testing in Kotlin

Today, I'd like to discuss how we write tests. Not their _contents_, but the tests _themselves_. How do we declare which tests exist? How do we organize tests? Can we improve them?

For the duration of this article, we'll write tests for the following domain:

```kotlin
typealias UserId = String //(1)!

interface UserService {

	fun isUsernameValid(username: String): Boolean //(2)!
	fun isPasswordValid(password: String): Boolean
	suspend fun create(username: String, password: String): UserId
}
```

1. In the real world, I would recommend using a value class instead for better type safety.
2. In the real world, I would probably use a `Username` value class instead of merging this concern into the user service.

We want to ensure that:

- Usernames and passwords cannot be empty.
- Passwords cannot be shorter than eight characters.
- It is not possible to create a user with an invalid username or password.
- It is not possible to create a user with a username that is already taken.

In the real world, the set of tests would probably be much larger, as we didn't take into account the specific characters that may be allowed in the username, etc. For now, this should suffice.

### Kotlin-test

Out of the box, the Kotlin standard library provides the [`kotlin-test` artifact](https://kotlinlang.org/api/core/kotlin-test/). Kotlin-test is heavily inspired by JUnit5, to the point of being almost perfectly identical—to be more precise, Kotlin-test is a subset of JUnit5.

Much like JUnit5, Kotlin-test requires a class that will hold the tests, which are defined as methods with an annotation:
```kotlin
class FooTest {
	
	@Test
	fun thisIsATest() {
		// Your test here
	}
}
```

Kotlin-test runs on almost all Kotlin platforms and is supported by default in IntelliJ. Because it is the default framework, I will use it as a baseline to compare other frameworks against.

Let's implement all of these tests.
```kotlin
class UserTest {
	
	lateinit var database: Database
	lateinit var users: UserService
	
	@BeforeTest
	fun setupUsers() = runTest {
		database = Database.connect()
		users = UserService(database)
	}
	
	@AfterTest
	fun dropUsers() = runTest {
		database.drop()
	}
	
	@Test
	fun usernameCannotBeEmpty() {
		assertFalse(users.isUsernameValid(""))
	}
	
	@Test
	fun validUsernames() {
		assertTrue(users.isUsernameValid("alice"))
		assertTrue(users.isUsernameValid("bob"))
	}

	@Test
	fun passwordCannotBeEmpty() {
		assertFalse(users.isPasswordValid(""))
	}

	@Test
	fun passwordMinimumSize() {
		assertFalse(users.isPasswordValid("1"))
		assertFalse(users.isPasswordValid("12"))
		assertFalse(users.isPasswordValid("1234567"))
	}
	
	@Test
	fun validPasswords() {
		assertTrue(users.isPasswordValid("12345678"))
		assertTrue(users.isPasswordValid("abrehuqdnjhkf"))
	}
	
	@Test
	fun cannotCreateUserIllegalData() = runTest {
		assertThrows { users.create("", "123456789") }
		assertThrows { users.create("alice", "") }
		assertThrows { users.create("bob", "123") }
		assertThrows { users.create("carol", "1234567") }
	}
	
	@Test
	fun canCreateUser() = runTest {
		users.create("alice", "123456789")
	}
	
	@Test
	fun cannotCreateDuplicateUser() = runTest {
		users.create("alice", "123456789")
		assertThrows { users.create("alice", "48948645346") }
	}
}
```

As you can see, this is a _lot_ of code for the very simple example we have in front of us, with only four business rules. Tests should help understand what the code does, and these are quite verbose. 

In addition, these also take a few shortcuts: for example, multiple cases are declared in a single test, because declaring a new test for each would be too verbose. While this works, it makes the reports less useful: if the first case of a test fails, we will not have any information on the other cases. In a larger codebase, this can quickly make understand complex failures harder. 

Using Kotlin-test, there is not much more we can do if we want to remain multiplatform-compatible. We cannot use `@BeforeClass` and `@AfterClass` on non-JVM platforms, and the backticks-notation for method names doesn't work on JS.

### JUnit5

Kotlin-test is based on JUnit5, and JUnit5 is much, much more complete. In this particular example, JUnit5 offers nesting to organize the tests, and parameterization to isolate each case into a single test.
```kotlin
class UserTest {
	
	lateinit var database: Database
	lateinit var users: UserService
	
	@BeforeEach
	fun setupUsers() = runTest {
		database = Database.connect()
		users = UserService(database)
	}
	
	@AfterEach
	fun dropUsers() = runTest {
		database.drop()
	}
	
	@Nested
	inner class DataValidation {

		@Test
		fun usernameCannotBeEmpty() {
			assertFalse(users.isUsernameValid(""))
		}

		@ParameterizedTest
		@ValueSource(strings = ["alice", "bob"])
		fun validUsernames(username: String) {
			assertTrue(users.isUsernameValid(username))
		}

		@Test
		fun passwordCannotBeEmpty() {
			assertFalse(users.isPasswordValid(""))
		}

		@ParameterizedTest
		@ValueSource(strings = ["1", "12", "1234567"])
		fun passwordMinimumSize(password: String) {
			assertFalse(users.isPasswordValid(password))
		}

		@ParameterizedTest
		@ValueSource(strings = ["12345678", "abrehuqdnjhkf"])
		fun validPasswords(password: String) {
			assertTrue(users.isPasswordValid(password))
		}
	}
	
	@Nested
	inner class Creation {

		@ParameterizedTest
		@CsvSource([",123456789", "alice,", "bob,123", "carol,1234567"])
		fun cannotCreateUserIllegalData(username: String, password: String) = runTest {
			assertThrows { users.create(username, password) }
		}

		@Test
		fun canCreateUser() = runTest {
			users.create("alice", "123456789")
		}

		@Test
		fun cannotCreateDuplicateUser() = runTest {
			users.create("alice", "123456789")
			assertThrows { users.create("alice", "48948645346") }
		}
	}
}
```

We've used `@Nested` to create groups of tests (typically called ‘suites’) and `@ParameterizedTest` to extract the different cases from tests. In theory, this is a much better way to declare tests: each test only tests a single thing, after all.

In practice, however… Do you find this example more readable than the previous one?

I don't.

For code to be readable, it needs, at the minimum, to make sense intuitively while reading it—but often, we expect much more: the ability to understand what it does through skimming it. We build our understanding of a codebase by focusing on what matters and knowing what the details are. We are trained to see annotations as metadata: annotations are details. What matters is the content of the tests, and we've made the content harder to read by removing the useful information.

Worse, this is actually hard to write. We've gone from needing to understand “tests are regular functions annotated with `@Test`” to an entire new ecosystem of annotations that provide slightly different behaviors. But annotations _are not code_. We cannot reason about them the way we reason about everything else.

But what if we could do better?

## What if we removed the magic?

We are using Kotlin, and Kotlin is all about making complex things simple, and verbose things short. Let's pull back a bit, and try to see what we could improve, bit by bit.

```kotlin
class UserTest {

	// …
}
```

First, why are we declaring a class? Because Java doesn't have top-level functions—but Kotlin does.

```kotlin
@Test
fun usernameCannotBeEmpty() {
	assertFalse(users.isUsernameValid(""))
}
```

Why are tests functions? Because they're the simpler way to declare code to execute in Java. But that syntax is almost entirely boilerplate. The parameters cannot be used, except in the case of test parameterization. But parameterization exists because the tests are functions.

[//]: # (TODO: top-level functions)
[//]: # (TODO: suspend and fixtures)
[//]: # (TODO: data-driven)

### Kotest

### Prepared

[//]: # (TODO: Arbitrary test names)
[//]: # (TODO: Test nesting - suites)
[//]: # (TODO: Test parametrization)
[//]: # (TODO: Reusing tests between multiple suites)
[//]: # (TODO: fixtures)

## Entering the stage

```kotlin
val UserTest by preparedSuite {
	
	
}
```

## Composition
