---
date: 
  created: 2026-05-04
slug: errors-without-flatmap
tags:
  - Kotlin
---

# Arrow: Error handling without flatMap

The Arrow library is quite poorly known. As Kotlin 2.4.0 is around the corner, bringing stable context parameters, it is time to brush a few misconceptions.

<!-- more -->

## Arrow isn't (only) about errors

Arrow isn't an error management library. Arrow is a collection of libraries, one of which is about errors-as-values.

`arrow-fx-coroutines` is about facilitating high-level usage of coroutines. It brings methods such as [`parMap`](https://apidocs.arrow-kt.io/arrow-fx-coroutines/arrow.fx.coroutines/par-map.html) to easily write concurrent mapping operations, [`parZip`](https://apidocs.arrow-kt.io/arrow-fx-coroutines/arrow.fx.coroutines/par-zip.html) to concurrently join multiple values into one, [`raceN`](https://apidocs.arrow-kt.io/arrow-fx-coroutines/arrow.fx.coroutines/race-n.html) to select the fastest of multiple operations, [resources](https://arrow-kt.io/learn/coroutines/resource-safety/) to ensure your database connection is closed only after your server stops receiving requests, etc. 

Using `arrow-fx-coroutines` is a no-brainer, and multiple of its features will not be added to KotlinX.Coroutines _because_ they are already present in Arrow.

`arrow-resilience` is about dealing with unexpected temporary errors, like a network failure between microservices. It provides methods for retrying operations and implements lightweight distributed transactions (with distributed rollback).

`arrow-optics` is about transforming deep immutable data structures.

Together, the Arrow collection complements many aspects of the Kotlin standard ecosystem.

## Arrow isn't about functional programming

Arrow is not about the functional programming you're thinking of, at least.

When building a web server, do you organize your server code in a domain layer which consists of stateless service classes that contain methods that transform immutable data classes by copying them? That's not object-oriented programming! There are no interior-mutable objects passing around messages. This is functional programming.

In that sense, almost the entire Kotlin standard library, and large parts of the ecosystem, is about functional programming. Arrow definitely follows the same trend. But the functional programming Arrow advertises isn't some obscure mathematics concept, it's the code most Kotlin developers are already writing.

The Arrow team used to have the goal of bringing over concepts from Haskell and Scala to Kotlin, like monads, type classes, applicatives, etc. Since around Arrow 1.0 (2021!), all of these types have been removed from the Arrow library. This is not because these types are bad concepts, it's because Kotlin can represent them in simpler ways. [Even the Scala ecosystem is taking inspiration on the way Arrow represents errors](https://github.com/rcardin/raise4s).

Today, the Arrow libraries follows the same guidelines as KotlinX libraries, especially KotlinX.Coroutines.

## Arrow isn't about flatMap hell

Functional error handling is often demonstrated by focusing on errors-as-values: instead of throwing exceptions, a method returns a special wrapper type that can contain either a success or a failed value.

```kotlin title="Arrow (Kotlin)"
fun findUser(id: String): Either<UserNotFound, User> {
	val user = repository.findUser(id)
	
	return 
		if (user != null) user.right()
		else UserNotFound(id).left()
}
```

```javascript title="Effect (JS)"
function findUser(id) {
	const user = repository.findUser(id);
	
	return user ? 
		Either.right(user) : 
		Either.left(new UserNotFound(id));
}
```

```java title="Vavr (Java)"
static Either<UserNotFound, User> findUser(String id) {
	final User user = repository.findUser(id);
	
	return user != null ? 
		Either.right(user) : 
		Either.left(new UserNotFound(id));
}
```

As soon as we introduce wrapped types, we need to consider how to declare operations that use wrapped values. This is where the infamous `flatMap` hell arrives.

Let's imagine a simple example:
```kotlin title="Kotlin, without errors-as-values"
fun getLocation(id: String): Location =
	TODO()

fun getWeatherAt(location: Location): Weather =
	TODO()

fun Weather.toConditions(): Conditions =
	TODO()

fun printConditionsAt(locationId: String) {
	val location = getLocation(locationId)
	val weather = getWeatherAt(location)
	val conditions = weather.toConditions()

	println("Weather at $location: $weather - $conditions")
}
```

Now, let's add error management and see how the example changes.

```kotlin title="Result4K (Kotlin)"
fun getLocation(id: String): Result<Location, LocationError> =
	TODO()

fun getWeatherAt(location: Location): Result<Weather, WeatherError> =
	TODO()

fun Weather.toConditions(): Result<Conditions, ConditionsError> =
	TODO()

fun printConditionsAt(locationId: String) {
	val message = getLocation(locationId)
		.mapFailure { "Could not get location by ID: $it" }
		.flatMap { location ->
			getWeatherAt(location)
				.mapFailure { "Could not get the weather at location $location: $it" }
				.flatMap { weather ->
					weather.toConditions()
						.mapFailure { "Could not convert weather to conditions: $it" }
						.flatMap { conditions ->
							"Weather at $location: $weather - $conditions"
						}
				}
		}
		.get()
	
	println(message)
}
```

A lot of code needs to be added to deal with the wrapped value. This isn't a critique of Result4K specifically, this is how most errors-as-values libraries in imperative programming languages solve this problem. This is, in fact, the state of the art in imperative languages.

**However, this is not how this code would be written in a true functional language.**

Here is the same function without error handling, in Haskell:
```haskell title="Haskell, without error handling"
printConditionsAt :: String -> IO ()
printConditionsAt locationId = do
    let location   = getLocation locationId
    let weather    = getWeather location
    let conditions = toConditions weather
    
    let message = "Weather at " ++ show location ++ ": " ++ show weather ++ " - " ++ show conditions
    putStrLn message
```
And here is how it changes when error handling is introduced:
```haskell title="Haskell, with error handling"
printConditionsAt :: String -> IO ()
printConditionsAt locationId = do
    let result = do
            location <- mapLeft (\e -> "Could not get location by ID: " ++ show e) $ 
                        getLocation locationId
            
            weather <- mapLeft (\e -> "Could not get the weather at location " ++ show location ++ ": " ++ show e) $ 
                       getWeather location
            
            conditions <- mapLeft (\e -> "Could not convert weather to conditions: " ++ show e) $ 
                          toConditions weather
            
            return $ "Weather at " ++ show location ++ ": " ++ show weather ++ " - " ++ show conditions

    case result of
        Left err  -> putStrLn err
        Right msg -> putStrLn msg
```

> Try it yourself: [without error handling](https://play.haskell.org/saved/agy8FQt8) • [with error handling](https://play.haskell.org/saved/vqD71EwM)

Of course, the code with error handling is more verbose. After all, it does more. The crucial difference is that it can unwrap values without introducing nesting. Failures immediately short-circuit, and the code can continue as if there was no wrapping.

Scala's `for` comprehensions, Rust's `?` operator and F#'s `let!` all provide the same ability to deal with wrapped values without dealing with the wrappers. Nested `flatMap` isn't something that happens in functional languages, so why should it happen in Kotlin?

The runtime behavior is the same as the `flatMap` version, this is purely syntax sugar.

Arrow brings the same capability to Kotlin, without needing a compiler plugin or any other transformations, because Kotlin already provides everything required to implement this syntax out of the box.

!!! info
    These examples use [context parameters](https://kotlinlang.org/docs/context-parameters.html), a new stable language feature in Kotlin 2.4.0. I have already [written about them](../2025/context-parameters.md).
    
    Arrow can be used without context parameters (replacing them with extension receivers), but the code can be slightly less idiomatic because there can only be one extension receiver per method.

Here is the Kotlin code without error handling, identical as previously:
```kotlin title="Kotlin, without errors-as-values"
fun getLocation(id: String): Location =
	TODO()

fun getWeatherAt(location: Location): Weather =
	TODO()

fun Weather.toConditions(): Conditions =
	TODO()

fun printConditionsAt(locationId: String) {
	val location = getLocation(locationId)
	val weather = getWeatherAt(location)
	val conditions = weather.toConditions()

	println("Weather at $location: $weather - $conditions")
}
```

Here is how it transforms when using Arrow Typed Errors:
```kotlin title="Kotlin, with Arrow Typed Errors"
context(_: Raise<LocationError>)
fun getLocation(id: String): Location =
	TODO()

context(_: Raise<WeatherError>)
fun getWeatherAt(location: Location): Weather =
	TODO()

context(_: Raise<ConditionsError>)
fun Weather.toConditions(): Conditions =
	TODO()

fun printConditionsAt(locationId: String) {
	val message = merge {
		val location = withError({ "Could not get location by ID: $it" }) {
			getLocation(locationId)
		}

		val weather = withError({ "Could not get the weather at location $location: $it" }) {
			getWeatherAt(location)
		}

		val conditions = withError({ "Could not convert weather to conditions: $it" }) {
			weather.toConditions()
		}

		"Weather at $location: $weather - $conditions"
	}

	println(message)
}
```

Just like in the Haskell example, we can deal with errors by short-circuiting, without having to deal with wrapper types. No matter the number of operations, our code stays flat and doesn't need extra indentation.

If we want to use functional programming ideas, let's not also adopt the boilerplate they learned to avoid decades ago!

***

This example was taken as one of the worst case scenarii: each method returns a different error, requiring different handling.

We can simplify it by making all methods fail with the same type.

```kotlin title="Result4K (Kotlin)"
fun getLocation(id: String): Result<Location, WeatherError> =
	TODO()

fun getWeatherAt(location: Location): Result<Weather, WeatherError> =
	TODO()

fun Weather.toConditions(): Result<Conditions, WeatherError> =
	TODO()

fun printConditionsAt(locationId: String) {
	val message = getLocation(locationId)
		.flatMap { location ->
			getWeatherAt(location)
				.flatMap { weather ->
					weather.toConditions()
						.flatMap { conditions ->
							"Weather at $location: $weather - $conditions"
						}
				}
		}
		.mapFailure { "Error: $it" }
		.get()

	println(message)
}
```

```haskell title="Haskell"
printConditionsAt :: String -> IO ()
printConditionsAt locationId = do
    let result = do
            location   <- getLocation locationId
            weather    <- getWeatherAt location
            conditions <- toConditions weather
            return $ "Weather at " ++ show location ++ ": " ++ show weather ++ " - " ++ show conditions

    case result of
        Left err  -> putStrLn $ "Error: " ++ show err
        Right msg -> putStrLn msg
```
As you can see, the Haskell version almost completely hides the error handling. Compared to the error-less version, the only difference is the `=` operator replaced by the `<-` operator.

Thanks to context parameters, Kotlin can go even further, and have _no difference at all_.

```kotlin title="Kotlin, with Arrow Typed Errors"
context(_: Raise<WeatherError>)
fun getLocation(id: String): Location =
	TODO()

context(_: Raise<WeatherError>)
fun getWeatherAt(location: Location): Weather =
	TODO()

context(_: Raise<WeatherError>)
fun Weather.toConditions(): Conditions =
	TODO()

fun printConditionsAt(locationId: String) {
	val result = recover(
		block = {
			val location = getLocation(locationId)
			val weather = getWeatherAt(location)
			val conditions = weather.toConditions()
			"Weather at $location: $weather - $conditions"
		},
		recover = { "Error: $it" }
	)

	println(result)
}
```

Except for the `recover` call which describes the error handling strategy, the code of the method itself is completely unchanged from the initial error-less implementation.

***

**The best code should be the simplest one.**

Functional-style error handling is often presented as a complex mess of wrapper types, but this is a constraint we impose on ourselves. Functional programming languages have solved this problem, and that same solution has been available in Kotlin for multiple years.

If we want to write robust code, handling errors shouldn't be a chore. Code that handles errors should be as simple to read as code that doesn't.
