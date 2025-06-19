---
date: 
  created: 2025-06-17
  modified: 2025-06-19
slug: state-of-kotlin-tests
tags:
  - Kotlin
---

# Lifting Kotlin testing: From JUnit to TestBalloon

There's a lot that goes into testing, but we rarely think of the most basic aspect: syntax. 

<!-- more -->

## Testing in Kotlin

It is arguable that one of the main factors of whether good tests are written in a project is whether they are easy to write.
Making tests easier to maintain is known to improve their quality and how much effort team members will spend on maintaining them.

Still, like many things in Kotlin, we can guess the age of a library by looking at how much it mirrors Java's approach.
When the ecosystem started, it was safer to take a very strong inspiration from the Java ecosystem as it was known to be successful.
As time passes, however, Kotlin becomes more and more independent, and Kotlin-first solutions emerge.

Before we get to the main point of this article, let's take a quick overview of multiple test frameworks available in Kotlin, in chronological order.

!!! abstract "What is a test framework?"
    In this article, I differentiate between test frameworks and other test libraries. The test framework is responsible for handling the concept of tests: how we declare tests, how we organize them, how they are initialized.
    Other libraries, such as assertion libraries, will only be briefly mentioned in this article.
    If you're searching for a (short) comparison of assertion libraries, [see here](https://opensavvy.gitlab.io/groundwork/prepared/docs/tutorials/index.html#assertion-libraries).

##### JUnit5

[JUnit5](https://junit.org/junit5/) is the standard for test frameworks on the JVM. Built for Java, it supports everything you may expect from a test framework, and more.
Deeply integrated into Gradle and IntelliJ, JUnit5 is often the only way different tools can communicate.

Typically (but not always), tests are declared as `@Test`-annotated methods in a class suffixed by `Test`:
```kotlin
class FooTest {

	@Test
	fun thisIsATest() {
		assertEquals("foo", "foo")
	}
}
```

Beyond simple tests, JUnit5 supports test parameterization, dynamic test declaration, nesting, etc. 

##### Kotlin-test

[Kotlin-test](https://kotlinlang.org/api/core/kotlin-test/) is the official answer by the Kotlin team. Published alongside the Kotlin standard library, it is the default option for any Kotlin project.
Kotlin-test mirrors JUnit5 in most usages, only adding some metadata to help Kotlin usage (e.g. contracts in `assertNotNull` to power smart-casts).

Unsurprisingly, the Kotlin-test basic example is identical to the JUnit5 example:
```kotlin
class FooTest {

	@Test
	fun thisIsATest() {
		assertEquals("foo", "foo")
	}
}
```

Other than smart-casts, the main advantage of Kotlin-test is its support for all Kotlin platforms.
At the time of writing, it is the only test framework that is able to run on all platforms.

However, what Kotlin-test has in support for various platforms, it lacks in everything else. Kotlin-test doesn't have test nesting, parameterization, dynamic test declaration or anything else.

##### Kotest

[Kotest](https://kotest.io/) is probably the most well-known other test framework for Kotlin, and the main alternative to Kotlin-test.
Split into three subprojects which can be used independently (the framework, the assertion library, and utilities for property-based testing), Kotest offers everything you may need.

However, that size has a cost: Kotest is massive, which makes it slow to update. For example, the latest stable Kotest version at the time of writing (5.9.1) [doesn't support Kotlin Native since Kotlin 2.0.20](https://github.com/kotest/kotest/issues/4177) (August 2024, almost a year old).
Kotest has existed for a while, and many of its features integrate poorly with more recent developments, most notably KotlinX.Coroutines.Test (which provides [delay-skipping](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/)). For example, [Kotest doesn't support the coroutines test scheduler on Kotlin JS](https://github.com/kotest/kotest/issues/4077), [doesn't wait for foreground coroutines](https://github.com/kotest/kotest/issues/4058), [has a non-configurable maximum timeout of 1 minute](https://github.com/kotest/kotest/issues/3969), etc.

Still, Kotest's configurability and DSL-based approach has seduced many. Tests are declared in a class' constructor using [different available styles](https://kotest.io/docs/framework/testing-styles.html), for example `FunSpec`;
```kotlin
class FooTest : FunSpec({
	test("This is a test") {
		"foo" shouldBe "foo"
	}
})
```
or `ShouldSpec`:
```kotlin
class FooTest : ShouldSpec({
	should("be equal to foo") {
		"foo" shouldBe "foo"
	}
})
```

Each of these styles offers the same features, though the syntax is slightly different each time.

##### Prepared

[Prepared](https://opensavvy.gitlab.io/groundwork/prepared/docs/) is a magicless testing framework I maintain at [OpenSavvy](https://opensavvy.dev/). By magicless, I mean that all its features are based on the Kotlin language, not on magic. Its two main selling features are the simplicity when declaring complex tests (as will be demonstrated later in this article) and the way it handles fixtures.

Before creating Prepared, I was an avid Kotest user (and in a way I still am)—Prepared was created to implement a new way of managing fixtures, which Kotest could not implement without major changes, and as a way to cover for many of Kotest's issues, especially on Kotlin JS. For example, Kotest's suites are `suspend`, which is incompatible with the existing Kotlin JS toolchains. Instead, Prepared suites are not `suspend` (as fixtures encompass the same use-cases) and can therefore support nesting on all platforms.

However, Prepared isn't a _full_ test framework: it cannot work on isolation. Most notably, it doesn't have a test runner (the thing responsible for discovering and actually running the tests), instead relying on the other existing frameworks. For example, Prepared can run on top of JUnit5 (on the JVM), on top of Kotlin-test (on JS), and on top of Kotest (on all platforms supported by Kotest).

Depending on the test runner used, Prepared mimics how the runner describes its own tests. For example, when using Kotest, a typical declaration looks like:
```kotlin
class FooTest : PreparedSpec({
	test("This is a test") {
		check("foo" == "foo")
	}
})
```
Prepared offers many more features, which will be discussed later.

##### TestBalloon

[TestBalloon](https://github.com/infix-de/testBalloon/) is the youngest child in the group, and the reason I'm writing this article now. It is still very early on (with its first release only happening a few weeks ago during KotlinConf 2025) but I believe in the project, in part because I've been following its prototype for almost a year now. Oliver, the TestBalloon's author, started working on it after being frustrated with the difficulty of adding Wasm WASI support to Kotest.

Very similarly to Prepared, TestBalloon strives for simple concepts that combine elegantly, and a DSL-based approach. However, TestBalloon is able to run its own tests on most platforms thanks to its own compiler plugin. The typical test is declared as a top-level variable using the `testSuite` builder:
```kotlin
val FooTest by testSuite {
	test("This is a test") {
		assertEquals("foo", "foo")
	}
}
```

I do believe TestBalloon will make testing easier for everyone. For this reason, I added support for Prepared to run on top of TestBalloon less than 24 hours after TestBalloon's first release, and all Prepared examples in this article will use the TestBalloon runner.

Now that we have an idea of the frameworks that exist, let's go through some features they provide and the way they implement them.

## Organizing tests

As we've seen, the five test frameworks we're comparing can be grouped into two categories: annotation-based, and DSL-based.

Annotation-based frameworks represent tests as methods annotated with a special `@Test` annotation, whereas DSL-based frameworks provide methods to declare tests.

### Test names

One immediate advantage of DSL-based frameworks is that they trivially allow declaring tests with a nice-to-read name.
Compare:
```kotlin
@Test
fun usersCanCreatePosts() {
	// …
}
```
And:
```kotlin
test("Users can create posts") {
	// …
}
```

While this is quite a simple feature, it already makes everyday usage nicer, especially when reading test reports.

Kotlin-test has a similar feature via Kotlin's backticks notation:
```kotlin
@Test
fun `Users can create posts`() {
	// …
}
```
However, that notation doesn't work on all platforms.

### Nesting

It is often recommended that tests should be narrow in scope: a test should test a single thing. A consequence is that we create a vast number of tests.
To organize them better, different frameworks have different ways of organizing tests, of which the most common is nesting.

JUnit5 supports grouping with yet another annotation, `@Nested`:
```kotlin
class UserTest {

	@Test
	fun topLevelTest() {
		assertEquals("foo", "foo")
	}

	@Nested
	inner class PostTest {
		
		@Test
		fun nestedTest() {
			assertEquals("bar", "bar")
		}
	}
}
```

Kotlin-test doesn't support nesting.

DSL-based frameworks use regular methods to describe nesting, often called Context or Suites.

=== "Kotest"

    ```kotlin
    class UserTest : FunSpec({
        test("top-level test") {
            "foo" shouldBe "foo"
        }
    
        context("Posts") {
            test("Nested test") {
                "bar" shouldBe "bar"
            }
        }
    })
    ```

    This feature is not available on Kotlin JS.

=== "Prepared"

    ```kotlin
    val UserTest by preparedSuite {
        test("Top-level test") {
            check("foo" == "foo")
        }

        suite("Posts") {
            test("Nested test") {
                check("bar" == "bar")
            }
        }
    }
    ```

=== "TestBalloon"

    ```kotlin
    val UserTest by testSuite {
        test("Top-level test") {
            check("foo" == "foo")
        }

        testSuite("Posts") {
            test("Nested test") {
                check("bar" == "bar")
            }
        }
    }
    ```

The main advantage of the DSL approach, other than the better name given to the intermediate suite, is the ability to control the nesting programmatically. Because the nesting is a regular function call, we can use regular Kotlin tooling to control it: conditional statements, loops, etc.

### Tags

Another typical way of organizing tests is to mark them with some kind of user-defined tags. For example, to separate slow tests from fast tests to make it possible to only run the fast ones in the tight development loop.

Of course, JUnit5 supports this feature via an annotation:
```kotlin
class UserTest {
	
	@Test
	fun fastTest() {
		// …
	}
    
	@Test
	@Tag("slow")
	fun slowTest() {
		// …
	}
}
```

Kotlin-test doesn't support test tagging.

DSL-based frameworks use some kind of configuration declaration, which can usually be declared either at the test level or at the suite-level, in which case it applies to all contained tests.

=== "Kotest"

    ```kotlin
    val slow = Tag("slow") // Preferably shared between all usages

    class UserTest : FunSpec({
        test("Fast test") {
            // …
        }
    
        test("Slow test").config(tags = setOf(slow)) {
            // …
        }
    })
    ```

=== "Prepared"

    ```kotlin
    val Slow = Tag("slow") // Preferably shared between all usages

    val UserTest by preparedSuite {
        test("Fast test") {
            // …
        }

        test("Slow test", Slow) {
            // …
        }
    }
    ```

At the time of writing, I am not aware of this feature existing in TestBalloon.

### Skipping tests

It can sometimes be useful to mark tests as skipped (though it is often a risk that these become skipped forever and are never fixed).

Again, JUnit5 provides this feature as an annotation:
```kotlin
class UserTest {
	
	@Test
	fun active() {
		// …
	}
    
	@Test
	@Disabled("This test is disabled")
	fun disabled() {
		// …
	}
}
```

The DSL-based frameworks offer this feature similarly to tagging.

=== "Kotest"

    ```kotlin
    class UserTest : FunSpec({
        test("Active test") {
            // …
        }
    
        test("Disabled test").config(enabled = false) {
            // …
        }
    })
    ```

=== "Prepared"

    ```kotlin
    val UserTest by preparedSuite {
        test("Active test") {
            // …
        }

        test("Disabled test", Ignored) {
            // …
        }
    }
    ```

=== "TestBalloon"

    ```kotlin
    val UserTest by testSuite {
        test("Active test") {
            // …
        }

        test("Disabled test", TestConfig.disable()) {
            // …
        }
    }
    ```

Again, because these test frameworks use DSLs, it is possible to imperatively control in which cases tests are skipped.

## Dynamic tests

Being able to declare tests is one thing, but being able to concisely test multiple scenarii is even more helpful.

### Reusing tests

In Kotlin, it is quite common to use interfaces to describe a common contract. Let's say we wanted to test that all implementations of an interface follow a set of invariants.
Because all implementations must follow the same rules, we want to write the tests once and apply them to all implementations.

For the sake of the example, we will test implementations of an interface:
```kotlin
interface Reference<T> {
	fun getOrNull(): T?
	fun clear()
}
```

The contract is simple: once `clear()` has been called, `getOrNull()` should return `null`.

With JUnit5, we can create an abstract class that describes the test we want to execute:
```kotlin
abstract class ReferenceTest {
	
    abstract fun createTestReference(): Reference<String>
	
	@Test
	fun clearingShouldMakeTheValueUnavailable() {
		// Given
		val ref = createTestReference()
		assertNotNull(ref.getOrNull())
        
		// When
		ref.clear()
        
		// Then
		assertEquals(null, ref.getOrNull())
	}
}
```

Now, for a given implementation, we can create a specific implementation of the test class:
```kotlin
class DefaultReferenceTest : ReferenceTest() {
	
	override fun createTestReference() = TODO()
	
}
```

While this approach works, it has a few downsides:

- It is quite verbose.
- It sometimes can be a bit fickle. In some situations, I have had to add an empty `@Test`-annotated method in the child class for it to be picked up.

The same approach works with Kotlin-test.

For DSL-based frameworks, we can simply use Kotlin extension functions on the test scope.
With Kotest:
```kotlin
fun FunSpec.commonReferenceTests(createTestReference: () -> Reference<String>) {
	test("Clearing should make the value unavailable") {
		// Given
		val ref = createTestReference()
        ref.getOrNull() shouldNotBe null
        
        // When
        ref.clear()
        
        // Then
        ref.getOrNull() shouldBe null
    }
}

class DefaultReferenceTest : FunSpec({
	commonReferenceTests { TODO() }
})
```

A downside of this approach with Kotest is that the created function is specific to a given testing style, and cannot easily be used within a test class declared with another style.

The same approach can be followed with TestBalloon. I'll leave writing it to the reader.

Prepared is built around the concept of fixtures: the way a test can get outside input. The equivalent of the Kotest and TestBalloon approach can be written essentially in the same way, but Prepared also has a dedicated feature, the eponymous prepared type:
```kotlin
fun TestDsl.commonReferenceTests(reference: Prepared<Reference<String>>) {
	test("Clearing should make the value unavailable") {
		// Given
		checkNotNull(reference().getOrNull())
        
        // When
        reference().clear()
        
        // Then
        check(reference().getOrNull() == null)
    }
}

val DefaultReferenceTest by preparedSuite {
	commonReferenceTests { TODO() }
}
```
The prepared type represents a [_prepared value_](https://opensavvy.gitlab.io/groundwork/prepared/docs/features/prepared-values.html): a special value that is prepared once a test needs it. On the first usage, it is computed, and then the same value is returned for the entire duration of a test.
We will mention it in more details in the fixtures section.

### Parameterization

Test parameterization refers to running the same test with different data. These tests may also be called data-driven. In this style of testing, an elementary invariant is tested for a large number of values.
In the simplest case, we may want to run the test for a few specific values. In more complex situations, we may want to execute the test for a complex combinaison of cases.

In Junit5, parameterization is handled using annotations.
```kotlin
class UserTest {
    
	@ParameterizedTest
	@ValueSource(strings = ["", "a", "+", "+foo"])
	fun forbiddenUsername(username: String) {
		assertThrows<IllegalArgumentException> {
			Username(username)
		}
	}
}
```
Notice how `@Test` became `@ParameterizedTest`, an annotation `@ValueSource` is required to declare the values, and that annotation has a parameter for each supported argument type—which are most primitive types as well as `String` and `Class`.
`null` is supported through the additional annotation `@NullSource`. 
Enums are supported via `@EnumSource`, which allows specifying a subset of elements using a stringly-API or using regular expressions.
Arbitrary values are supported through `@MethodSource`, which requires declaring a `Stream`-returning method which will generate the different parameters, and takes an argument with the string name of that function.
Similarly, a static field can be referred to using the `@FieldSource` annotation.
And even with all of this, we cannot declare _two_ arguments without yet another system:
```kotlin
class UserTest {
    
	@ParameterizedTest
	@MethodSource("my.package.UserTest#provideParameters")
	fun forbiddenUsername(username: String, role: User.Role) {
		assertThrows<IllegalArgumentException> {
			Username(username, role)
		}
	}
    
	companion object {
		@JvmStatic
		fun provideParameters() = Stream.of(
			Arguments.of("", User.Role.Guest),
			Arguments.of("a", User.Role.Guest),
			Arguments.of("+", User.Role.Admin),
			Arguments.of("+foo", User.Role.Employee),
		)
	} 
}
```
And even then, there is no built-in way to declare a test with the cartesian product of two parameters.

Kotlin-test does not support test parameterization.

Once again, DSL-based tests show how simple everything can be once we rely on a proper programming language instead of creating new meta-languages with annotations.
After all, the tests are declared as code, and code can do all of these things.
For simplicity's sake, I will only provide the Prepared examples for this section, but the exact same technique works for the three DSL-based frameworks.

First, parameterizing a test with a single parameter:
```kotlin
val UserTest by preparedSuite {
	for (username in listOf("", "a", "+", "+foo")) {
		test("The username '$username' is invalid") {
			shouldThrow<IllegalArgumentException> {
				Username(username)
			}
		}
	}
}
```
There isn't a need for learning a new meta-language anymore: the existing programming language is already capable of handling loops.

We can also trivially rewrite the example with two parameters:
```kotlin
val UserTest by preparedSuite {
	val args = listOf(
		"" to User.Role.Guest,
		"a" to User.Role.Guest,
		"+" to User.Role.Admin,
		"+foo" to User.Role.Employee,
	)
	
	for ((username, role) in args) {
		test("The username '$username' is invalid") {
			shouldThrow<IllegalArgumentException> {
				Username(username, role)
			}
		}
	}
}
```

If we prefer a cartesian product, we can simply nest the loops:
```kotlin
val UserTest by preparedSuite {
	for (username in listOf("", "a", "+", "+foo")) {
		for (role in User.Role.entries) {
			test("The username '$username' is invalid with role $role") {
				shouldThrow<IllegalArgumentException> {
					Username(username, role)
				}
			}
		}
	}
}
```

Note that this trivially supports declaring cartesian products of parameters that depend on each other, which is never easy to do otherwise.

Techniques such as test parameterization are very powerful to trivially write and maintain tests that verify a large number of edge cases. Yet they are seldom used in the JUnit5 world.
I suppose the need to learn a new meta-language with its own different rules is the main culprit.

Although DSL-based frameworks can handle parameterization without any dedicated features, this doesn't mean it's not possible to do even better for complex cases.
For complex cases, my preferred library is [Parameterized](https://github.com/BenWoodworth/Parameterize):

```kotlin
val UserTest by preparedSuite {
    parameterized {
        val username by parameterOf("", "a", "+", "+foo")
        val role by parameter(User.Role.entries)

        test("The username '$username' is invalid with role '$role'") {
            shouldThrow<IllegalArgumentException> {
                Username(username, role)
            }
        }
    }
}
```

Even better, because DSL-based frameworks are declared as regular code, they compose with the library ecosystem much better.
The Parameterized library isn't made _for_ Prepared, or for any other framework.
It just works with all DSL-based frameworks, even the ones that haven't been written yet.

This ability to be extended outside the framework is a meaningful strength for long-term maintainability and evolution into new paradigms as fashion shifts.

## Fixtures

It is generally accepted that tests should contain the least possible amount of code, to clearly differentiate what is being tested versus what is setup.
The setup is often shared between multiple tests.

This setup is called a fixture: something that isn't really part of the test, yet is required by the test.

### The classical approach

In the most simple case, we may have a service that must be usable within your tests.
```kotlin
class UserTest {
	private lateinit var connection: Connection
	private lateinit var userService: UserService
	
	@BeforeEach
    fun setUp() {
		connection = DriverManager.getCollection("jdbc:test:db")
        userService = UserService(connection)
	}
    
    @Test
    fun createUser() {
		userService.create(User("foo", "bar"))
	}
    
    @AfterEach
    fun cleanUp() {
		connection.rollback()
        connection.close()
	}
}
```

Note how fixtures are split into four parts:

- The declaration,
- The initialization,
- The destruction,
- The usage.

Both JUnit5 and Kotlin-test, being annotation-based frameworks, use additional methods with a specific annotation to declare the fixtures. Because Kotlin fields must be initialized, the keyword `lateinit` is required to avoid compilation error—but that has the consequence that the compiler will not complain if a variable is never initialized. 

Kotest offers a similar feature, adapted to its DSL:

```kotlin
class UserTest : StringSpec({
    lateinit var connection: Connection
    lateinit var userService: UserService
    beforeTest {
        connection = DriverManager.getCollection("jdbc:test:db")
        userService = UserService(connection)
    }
    "Create a user" {
        userService.create(User("foo", "bar"))
    }
    afterTest {
        connection.rollback()
        connection.close()
    }
})
```
Still, `lateinit` is required to declare the variables, and the management is spread out over the entire file.

### Dedicated fixture support

Prepared and TestBalloon both offer a dedicated concept of test fixture; although they have similar syntax, they behave quite differently.
The TestBalloon fixture is the most similar to traditional fixtures found in other frameworks:
```kotlin
val UserTest by testSuite {
	val connection = testFixture {
		DriverManager.getCollection("jdbc:test:db")
	} closeWith {
        connection.rollback()
        connection.close()
	}
    
    val userService = testFixture {
		UserService(connection())
	}
    
    test("Create a user") {
        userService().create(User("foo", "bar"))
    }
}
```
Notice how having a first-class concept of fixtures allows co-locating all parts of the fixture into a single place.
TestBalloon fixtures are lazy: that is, they must explicitly be referred to within a test to be initialized.
Since they can depend on each other, in practice, this doesn't increase the amount of code by much.
Each test fixture gets a single value shared between all tests that refer to it.

The Prepared framework was initially created because of its new way of handling fixtures, which are split into two types: [prepared values](https://opensavvy.gitlab.io/groundwork/prepared/docs/features/prepared-values.html) and [shared values](https://opensavvy.gitlab.io/groundwork/prepared/docs/features/shared-values.html). Both are declared similarly to TestBalloon's fixtures:
```kotlin
val UserTest by preparedSuite {
	val connection by prepared {
		val connection = DriverManager.getCollection("jdbc:test:db")
        cleanUp("Close the connection") {
			connection.rollback()
            connection.close()
        }
        connection
	}
    
    val userService by prepared {
        UserService(connection())
    }
    
    test("Create a user") {
        userService().create(User("foo", "bar"))
    }
}
```
While the syntax and features are similar, including the fact that prepared values are lazy, they have the difference that each test gets its own value.
This greatly removes the amount of shared state between tests, which often causes flakiness.
Because of this, prepared values are recommended in most situations—but sometimes, setting up a new fixture for each test can be expensive.
In these cases, Prepared offers shared values, which have the same feature set (except that [they don't currently support finalizers](https://gitlab.com/opensavvy/groundwork/prepared/-/issues/83)).
Shared values have the same syntax but have a single value reused during all tests, like TestBalloon's fixtures.

Additionally, Prepared's fixtures can be used anywhere, not just within suites. This makes them convenient to share logic reused in many parts of the tests.

Here is a short summary:

| Feature                    | TestBalloon          | Prepared's prepared values | Prepared's shared values |
|----------------------------|----------------------|----------------------------|--------------------------|
| Initialized lazily         | ✓                    | ✓                          | ✓                        |
| Can depend on each other   | ✓                    | ✓                          | ✓                        |
| Are isolated between tests | No                   | ✓                          | No                       |
| Can be declared anywhere   | No (only in a suite) | ✓                          | ✓                        |
| Supports finalizers        | ✓                    | ✓                          | No (planned)             |

## Integrations

While everything we've discussed so far is great for testing simple systems, in practice, we often work with complex structures that require additional features.
In this section, let's review a few of these additional complexities.

### Coroutines

Coroutines are a Kotlin feature specialized for asynchronous programming. Asynchronous functions are declared using the `suspend` keyword, which colors them (`suspend` functions can only be called within other `suspend` functions).
To write tests which use coroutines, a special conversion is required to be able to enter the `suspend` world. If the framework doesn't provide it, the user must do so themselves.

JUnit5 doesn't support coroutines. Users can use `runBlocking` (from `kotlinx-coroutines-core`) or `runTest` (from `kotlinx-coroutines-test`) to enter the coroutines world:
```kotlin
class UserTest {
    private lateinit var connection: Connection
    private lateinit var userService: UserService

    @BeforeEach
    fun setUp() = runBlocking {
        connection = DriverManager.getCollection("jdbc:test:db")
        userService = UserService(connection)
    }

    @Test
    fun createUser() = runBlocking {
        userService.create(User("foo", "bar"))
    }

    @AfterEach
    fun cleanUp() = runBlocking {
        connection.rollback()
        connection.close()
    }
}
```
Note that because the fixtures get their own coroutine root, there is no possible cancellation or context sharing between fixtures.

Kotlin-test supports coroutines similarly, however multiplatform support has a few limitations:

- `runBlocking` is not supported in multiplatform,
- Only the syntax `#!kotlin fun foo() = runTest { … }` is supported; `#!kotlin fun foo() { runTest { … } }` isn't.

Once again, DSL-based frameworks shine because they can hide the initialization from users. In fact, all DSL-based frameworks we examined offer coroutines support without any additional configuration or syntax.

`runTest` has another peculiarity: within it, coroutines are modified to make them easier to test. Coroutines execute in a single thread (to facilitate deterministic ordering) and delays are skipped, making it easy to test algorithms that wait for some time before performing an action, or depend on timeouts, without actually having to wait for that time. Additionally, a background scope is available to run tasks that may have a longer lifetime than the test itself: the test will wait until all foreground coroutines to terminate, but will not wait for background coroutines.

- Kotest has partial support that is disabled by default (enable with `TestConfig(coroutineTestScope = true)`).
- Prepared supports these features and they are enabled by default. 
- TestBalloon supports these features and they are enabled by default.

### Time and randomness

A common source of non-determinism in tests is the passage of time, or the usage of randomness.
We can abstract these uncontrollable events through generators.
This way, when testing a service, we can use a deterministic generator we control instead of the real one.

Here are a few examples of generators:

- For the current time: `Clock`
- For measuring elapsed time: `TimeSource`
- For random values: `Random`

Typically, the generators are provided as interfaces in their respective library, allowing users to create their own controlled implementations.
It is therefore not crucial if a test framework doesn't have a dedicated feature, but their presence can still be a quality of life.

Junit5 and Kotlin-test don't provide features related to this.

Kotest provides an additional module to get a controllable `Clock` on which the `clock.plus(6.minutes)` operator is available to advance the time.
However, that clock is not multiplatform.

Prepared offers [time control](https://opensavvy.gitlab.io/groundwork/prepared/docs/features/time.html) and [randomness control](https://opensavvy.gitlab.io/groundwork/prepared/docs/features/random.html) through the special `time` and `random` variables which can be accessed within tests and fixtures:
```kotlin
val UserTest by preparedSuite {
    test("When logging in, we artificially wait for one second to avoid side-channel attacks") {
        val testUser = userService.createUser("foo.${random.nextInt()}@mail.com", "123456789")
		
		val time = time.source.measureTime {
			userService.logIn(testUser.username, "123456789")
        }
        
        check(time >= 1.seconds)
    }
}
```
Thanks to delay-skipping provided by KotlinX.Coroutines, this test will finish immediately, but will in fact wait for a second in a production system. Additionally, we can trivially test whether the system behaves correctly at specific times, for example during the new year transition, by inserting `time.set("2024-12-31T23:59:59Z")` at the start of the test. Finally, because the test uses a random value, it will pick a random seed and print it to the standard output. To reproduce a test that executed on another machine, you can simply insert `random.setSeed(123456)` at the start of the test.

TestBalloon doesn't provide such features for now.

### The filesystem

Some test frameworks provide features to interact the filesystem. Typically, two different kinds of features are supported:

- Reading test data from the filesystem (e.g. complex inputs, expected output…)
- Working with temporary files and directories.

Kotest offers utilities for creating temporary files:
```kotlin
class MySpec : FunSpec({
	val file = tempfile()
    test("A test") {
		file.writeText("foo")
    }
})
```
This feature is not available on all platforms. The same file may be used by multiple tests in a single suite. The file is automatically deleted at the end of the suite.

Prepared offers similar utilities:
```kotlin
val MyTest by preparedSuite {
	val file by createRandomFile()
    test("A test") {
		file().writeText("foo")
    }
}
```
This feature is not available on all platforms. Each test gets a different file. The file is automatically deleted at the end of the test—except if the test fails. If the test fails, the path of the file is printed to the standard output and the file is kept, so you can analyze it yourself.

Prepared offers the function `resource` to access a JVM resource associated to a given class:
```kotlin
val MyTest by preparedSuite {
	val test by resource<TargetClass>("test.txt").read()
    
    test("A test") {
		println(test())
    }
}
```

TestBalloon doesn't provide such features for now.

### Frameworks and libraries

Finally, test frameworks can provide helpers to production frameworks.
Here, Kotest wins by far thanks to its longevity.
For brevity, this list isn't exhaustive and not detailed.

| Framework      |   JUnit5    | Kotlin-Test | Kotest | Prepared | TestBalloon |
|----------------|:-----------:|:-----------:|:------:|:--------:|:-----------:|
| Spring         |      ✓      |             |   ✓    |          |             |
| Ktor           |             |             |   ✓    |    ✓     |             |
| TestContainers | Third-party |             |   ✓    |          |             |
| Kafka          | Third-party |             |   ✓    |          |             |
| MockServer     | Third-party |             |   ✓    |          |             |
| Koin           |             |             |   ✓    |          |             |
| Arrow          |             |             |   ✓    |    ✓     |             |
| Gradle         |             |             |        |    ✓     |             |

## Conclusion

With the recent birth of TestBalloon, we can see that there is still a lot of innovating in the space of test frameworks. Even after JUnit's dominance in the JVM ecosystem, we can still make tests easier to write and maintain. Kotlin makes possible patterns that we couldn't dream of in the Java world.

Nowadays, the main difficulty around creating test frameworks is the large amount of work needed to interoperate with the other tools, especially Gradle, the Kotlin compiler and IntelliJ. For the past year or so, the TestBalloon author and I have been discussing ways this could improve in the future; we hope that what we have each been able to achieve will demonstrate that a better experience is possible in this space.

Overall, test frameworks enormously affect the facility of writing tests and which schools of testing are made possible. We shouldn't underestimate how much simpler and expressive they can make tests, and in turn how easy it becomes to verify tricky conditions in our production systems.

Each test framework has its own approach and its own priorities, providing support for the most basic cases, but adding extended features for its own idioms. By understanding what their goals are, we can profit from them—and maybe get inspired for new ways of tackling existing problems when it becomes time to create yet another framework.

Did you discover a feature that could help your life? Give the projects a star!

- [JUnit5](https://github.com/junit-team/junit5)
- [Kotlin-test](https://github.com/JetBrains/kotlin)
- [Kotest](https://github.com/kotest/kotest)
- [Prepared](https://gitlab.com/opensavvy/groundwork/prepared)
- [TestBalloon](https://github.com/infix-de/testBalloon/)
