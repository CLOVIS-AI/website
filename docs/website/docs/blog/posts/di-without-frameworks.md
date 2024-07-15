---
date: 2024-07-15
slug: di-without-frameworks
tags:
  - Software architecture
---

# Dependency Injection without a framework

Dependency Injection is more popular than ever. As with all popular things, the original idea is becoming more and more diluted down, to the point that discussing it is becoming difficult. Let's go back to the roots, and see what DI really is.

<!-- more -->

## Understanding the problem

Every design pattern starts by attempting to solve a problem. Usually, the solution aims at providing flexibility at the cost of a slightly increased amount of code.

This time, the problem is about hardcoded functionality. As an example, let's say we have some external collection of objects, and we want to export them to another service that will generate some statistics and send them to someone. However, there may be many more objects that can fit into memory. Also, this process may be quite time-sensitive, so we want to process objects in parallel.

If you had to implement this, you may end up with something like:
```kotlin
fun publishStatistics() {
	val objectStream = objectSource
		.queryBuilder()
		.streamAll()
	
	try {
		objectStream.forEach { it -> StatisticsPublisher.publish(it) }
	} finally {
		objectStream.close()
	}
}
```
There's nothing obviously wrong with this function, it does what it says.

Later, we may want to add a new feature to dump objects to a CSV file. Again, we may not be able to fit all objects into memory, so we want to keep the streaming behavior. Let's write it.
```kotlin
fun dumpToCsv() {
	val objectStream = objectSource
		.queryBuilder()
		.streamAll()
	
	try {
		objectStream.forEach { it -> CsvExporter.dump(it) }
	} finally {
		objectStream.close()
	}
}
```

Ah, but this one is awfully similar to the first one, isn't it? If tomorrow we wanted to add parallel processing to the requests, we would need to edit both of them. As code scales, and more and more variants are created, the situation worsens, and the likelihood of incoherent behavior increases.

What can we do? We can recognize that in these two functions, there are actually _three_ algorithms:

- "Stream objects and execute an arbitrary operation on each of them",
- "Publish all objects one by one",
- "Dump all objects one by one".

The last two are already implemented elsewhere in our codebase: the `StatisticsPublisher.publish` and the `CsvExporter.dump` methods. What we really did is reimplement the first algorithm twice. Instead, we could extract it to its own function that is parameterized by which action it applies to streamed elements.

```kotlin hl_lines="1 7"
fun onEachObject(action: (MyObject) -> Unit) {
	val objectStream = objectSource
		.queryBuilder()
		.streamBuilder()
	
	try {
		objectStream.forEach(action)
	} finally {
		objectStream.close()
	}
}
```

Now, both other functions are trivially created on top of this first algorithm:
```kotlin
fun publishStatistics() {
	onEachObject { it -> StatisticsPublisher.publish(it) }
}

fun dumpToCsv() {
	onEachObject { it -> CsvExporter.dump(it) }
}
```

This is what Dependency Injection is: extracting an algorithm that is deep within another one, so the wrapping algorithm is instead parameterized with the wrapped one. You may be more familiar with the object-oriented version of the pattern, where the algorithm is represented by an interface:
```kotlin hl_lines="6 12"
interface ObjectAction {
	fun act(it: Object)
}

class ObjectService(
	val exporter: ObjectAction
) {
	// …
	
	fun onEachObject() {
		// …same code…
		objectStream.forEach { it -> exporter.act(it) }
	}
}
```

Whether you use the function-oriented or object-oriented approaches, the result is the same: it is now possible to replace the inner algorithm by another implementation without editing (or duplicating) the outer algorithm.

Now, this example was quite simple, and I won't claim that dependency injection was the best tool to use here. Instead, let's move to some situations where the benefits are more obvious.

## Let's inject some dependencies

### 1. Time-sensitive algorithms

We have all had to work with a time-sensitive algorithm at some point, and we probably all know how painful they are to test and debug. Usually, this is because the algorithm measures the time itself (e.g. through methods like `Instant.now()`). Using DI, we instead inject an object that can generate measurements (usually called `Clock` or `TimeSource`). During test or debugging, we inject another instance, that we can control to trigger the specific cases we are interested in.

This way, we can stop the time, control how fast it goes, skip time until the next interesting event…  The same technique applies to many other data generators, including random number generators.

### 2. Higher-level functions

You may have noticed that the initial examples of this article boiled down to instantiating a stream and  performing operations on it. Stream APIs with higher-level functions (`filter`, `map`, `flatMap`, `forEach`…) are examples of injecting an algorithm within another one to avoid having to rewrite the boring loops that make it work.

Among them, the most visible examples are the `sort` variants. Usually, these functions either take no parameters (and use the natural sorting order of the underlying values) or take some kind of `Comparator` and ask it to decide, out of two values, which should go first, parameterizing a sorting algorithm that may be several orders of magnitude more complex.

### 3. Any other dependency, really

Services usually trigger events or send data to other parts of the system. However, these dependencies make the code harder to update when these systems change, and harder to test. By extracting them from our main algorithm, our main algorithm becomes "purer", in a sense: its code only contains what is itself does, and it is parameterized by what other systems should do as a reaction.

If you want to dive deeper into this specific use-case and how to architecture complex systems around it, I cannot recommend Code Aesthetic enough:
<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/J1f5b4vcxCQ?si=91qbGRfrIJYo6aTV" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

---

You're a new developer on a project. You've heard of dependency injection being a popular way to organize code; you may have heard some of its benefits. You want to use it in your project, but you're not really an expert, and you don't want to reinvent the wheel. You see that there are a few Dependency Injection frameworks, so you start using one.

But wait—throughout this entire article, we never mentioned them once, and yet we've used Dependency Injection the entire time.

## So, what do Dependency Injection frameworks do?

Well, most of the time, they don't do dependency injection. Dependency injection is decoupling algorithms from each other, meaning changing the structure of the code as it is written. A library cannot do that.

From what I can gather, this is largely a misnomer of another pattern, the service locator. Since service locators are often used to help with dependency injection, they end up having some kind of `@Inject` annotation or method, leading to the confusion.

A service locator is a component that is responsible for linking services together with their dependencies. Usually, they have some way of registering services, including which other services they depend on. When a service locator is asked to provide a specific service, it recursively initializes all its dependencies and returns it.

!!! note
    In the rest of this article, I will use [Koin](https://insert-koin.io/) as an example, because it's the one I'm most familiar with, but the benefits and drawbacks I will mention apply to most service locators.

Here's how services are registered and accessed with Koin:
```kotlin
// Declaring the services and their dependencies as parameters
interface UserRepository
class UserMongoRepository(val database: MongoDatabase) { … }

interface DataRepository
class DataMongoRepository(val database: MongoDatabase) { … }

class UserService(val repository: UserRepository)
class DataService(val repository: DataRepository, val users: UserService)

// Registering multiple similar services into a 'module'
val mongoRepositories = module {
	singleOf(::MongoDatabase)
	singleOf(::UserMongoRepository)
	singleOf(::DataMongoRepository)
}

val businessRules = module {
	singleOf(::UserService)
	singleOf(::DataService)
}

fun main() {
	startKoin {
		modules(mongoRepositories)
		modules(businessRules)
	}
    
    val dataService by inject<DataService>()
}
```
Koin knows which arguments are required by which constructors, and can build the final service by linking the others. In this example, all declared services are instantiated, because of the dependency tree:
```text
DataService is requested by the user; its constructors depend on:
    DataRepository
        DataMongoRepository is the only known implementation, so it is chosen; it itself depends on:
            MongoDatabase
    UserService is registered; its constructors depend on:
        UserRepository
            UserMongoRepository is the only known implementation, so it is chosen; it itself depends on:
                MongoDatabase which is declared as a singleton and has already been initialized, so the same instance is reused 
```

Service locators can help organize linking many services together, ensuring they aren't built multiple times.

## Are you benefitting from using Dependency Injection frameworks?

Service locators often claim the benefits of dependency injection as their own—but as we have seen, these are two different patterns. Let's visit a few of the possible benefits and drawbacks of service locators when used for dependency injection.

First, they allow categorizing the services into "modules" that can be substituted entirely out of the system. For example, if you use the same structure as the previous example, with the repositories in one module and the business in another, you can simply create a:
```kotlin
val testRepositories = module {
	singleOf(::FakeUserRepository)
	singleOf(::FakeDataRepository)
}
```
in which the fake repositories are hardcoded or in-memory. Then, simply adding the `businessRules` module along with this one generates a running version of the entire app, with the database removed.

Interestingly, this approach makes mocking frameworks somewhat redundant. The main goal of a mocking framework is to use magic to edit the internals of objects to control what they do—but with dependency injection, especially when a service locator handles the substitution for you, that is already something you can do very easily. I fail to see a good reason to use both a service locator and a mocking framework in the same project.

Some people argue that having this instantiation logic located in a single place is beneficial, but I'm not convinced that service locators help in any meaningful way there. After all, the above example can be rewritten to:
```kotlin
// the interface/class definitions are the same

fun main() {
	val database = MongoDatabase()
	val userRepository = UserMongoRepository(database)
	val dataRepository = DataMongoRepository(database)
	val userService = UserService(userRepository)
	val dataService = DataService(dataRepository, userService)
}
```
Honestly, I prefer this variant. Sure, the arguments all have to be spelt out, but the order of initialization is extremely clear, and this is way faster than the fastest service locators can claim to be (they often claim to be fast, because previous iterations were _very_ slow). This variant also has a major benefit in that the IDE actually understands the initialization usage, so data flow analysis and other navigation features work well (try them in with your service locator framework: each constructor becomes an opaque boundary).

Of course, this variant only works because the initial example only used `singleOf`. Your service locator may have many other powerful initialization strategies that may be harder to replicate without it. If you need them, that's great! But always think of whether your code would be simpler and more maintainable if you removed the framework.

I also think that service locators would be _more_ useful if they were used as part of each module. Instead of exposing all classes in a module, classes would all be internal, only exposing the module itself. This way, other modules couldn't refer to any specific implementations provided by the module, ensuring that class renames, deletions, destructurations could be performed with no external changes whatsoever. 
Many teams avoid doing this because it taints each module with the service locator framework, which I certainly agree would be a bad idea. I think each module could instead provide a helper module with its encapsulated features, in the same manner as test fixtures are exposed. Gradle makes this a bit complex, maybe this can be a feature for a future build tool.

I'm curious how service locators are used in other projects.

## A small note about naming

But really, this article was about a pet peeve of mine: as ideas become popular, they are mutated but often keep their initial name, even when they represent something completely different. Design patterns were meant to be recognizable pieces of architecture, their name itself being representative of a broader idea that we could use to communicate intent.

I don't really mind if you use service locators or not. If you think they make your code easier to maintain, great! I don't find them terribly useful myself, but I won't make a big deal of working on a project that uses them, either. I also haven't yet seen a bad bug that was caused by their usage.

But I do care that junior developers are more and more confused by the term "dependency injection", associating it with large reflection-heavy frameworks, when it refers to one of the most basic software architecture principles.

I hope this article was helpful to clear some confusions and understand some broader usages of the pattern.
Hopefully, we can all come to an agreement to rename dependency injection frameworks to something else—but I don't see it happening.
