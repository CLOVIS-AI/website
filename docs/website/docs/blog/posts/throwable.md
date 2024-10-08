---
date: 2024-10-07
slug: flee-throwable
tags:
  - JVM
  - Kotlin
---

# Don't touch Throwable

`Throwable` is the supertype of all things that can be thrown. It often appears in codebases as an upper bound for error handling. Here's why you shouldn't do it.

<!-- more -->

!!! note ""
    The examples in this article are written in Kotlin. However, everything mentioned applies in exactly the same way to Java and other JVM languages if they use the JVM's exceptions.

## What is Throwable?

`Throwable` is the parent type that enables the usage of the `throw` keyword. Any subtype of `Throwable` can be thrown, any other type cannot.

```kotlin hl_lines="3"
fun divide(dividend: Double, divisor: Double): Double {
	if (divisor == +0.0 || divisor == -0.0)
		throw ArithmeticException("Cannot divide $dividend by zero")
	
	return dividend / divisor
}
```

A value being thrown interrupts the execution of the current function. The call stack is rewound until a `try` block is found. There can be arbitrarily many methods between the place where the throwable is thrown and the `try` block—all these methods are interrupted without any explicit marking.

??? note "About Java's checked exceptions"
    Java has checked exceptions (subtypes of `Exception` but not of `RuntimeException`), which are marked by the `throws` keyword in intermediate functions. This makes little difference in regard to my points in this post, so they won't be discussed. 

```kotlin
fun a() {
	try {
		b()
	} catch (e: Throwable) {
		println("Something went wrong! $e")
	}
}

fun b() {
	c()
}

fun c() {
	println("Before!")
	d()
	println("After!")
}

fun d() {
	5 / 0
}
```
```text title="Standard output"
Before!
Something went wrong! java.lang.ArithmeticException: Cannot divide 5 by zero
```

As we can see, the `After!` line is never printed, because the thrown exception has rewound the call stack until the `a` function's `try` block. The `catch` block is executed, after which execution continues at the end of the `try` block, so the function `a` returns normally.

`Throwable` often appears in user code as part of the `catch` block, to perform an operation in situation of arbitrary failure.

**This usage should be stopped. Throwable isn't meant for recovery. Even worse, no meaningful recovery can be made using Throwable.**

Quite a bold claim, eh? Let me explain.

## Error, Exception, and the rest

`Throwable` is meant to expose a language feature. By itself, it doesn't hold any semantic meaning: in particular, it does not hold any guarantees on the current situation. `Throwable` has two subclasses which do expose semantic meaning: `Error` and `Exception`.

`Exception` represents exceptional situations which need to be handled by the program. These situations may be more or less unexpected, but all represent situations in which the JVM is able to continue normal execution. For example, `ArithmeticException` is thrown when division by zero occurs (or other arithmetic mistakes): the computation cannot continue, but the process as a whole is not threatened. `IOException` is thrown when the program attempts to read or write to a file, but the system refuses: again, the process is not threatened.

Opposite to them, `Error` represents situations in which the process *is* threatened. Depending on the thrown error, the process may be in progress of shutting down, or some of its capabilities may be unavailable. Because of this, it is not possible to assume anything in the general case.

By catching `Throwable`, we also catch `Error`. But, _by definition of what an error is_, catching a generic `Error` is not safe.

Many common recovery patterns fail in subtle ways, or even introduce new bugs, when faced with instances of `Error`. Let's review some of them.

## Recovery patterns

### Logging

Logging usually consists of sending some information to an external data storage. The simplest option is to use `System.out`, but any logging framework or displaying the error to the user is an equivalent solution.
```kotlin
try {
	someOperation()
} catch (e: Throwable) {
	log.error("Something went wrong: $e")
}
```

In this situation, we continue the execution no matter what the situation is. Here are a few situations in which this example exhibits a bug:

- `OutOfMemoryError` is thrown when the process is out of memory. In this situation, *any object instantiation* is likely to fail, and will itself throw an `OutOfMemoryError`. Since we do not control the internals of the logging system, and it is likely that it will instantiate some objects at some point, logging may not be possible. Since we are not performing any recovery specific to memory congestion, it is very likely that the system is still out-of-memory: it will be thrown again somewhere else.
- `VirtualMachineError` is thrown when the JVM is dying for any unknown reason. Nothing is guaranteed, any further operation may fail. There is nothing a program can do to attempt to salvage this. The logging likely won't succeed, and the error will likely be thrown again very soon.
- `ThreadDeath` is executed when a thread is killed by another thread. It is paramount that `ThreadDeath` is always rethrown: otherwise, the thread will continue executing even though the process thinks of it as dead. This situation is called a zombie thread, and it can lead to subtle data corruption bugs or simply resource exhaustion. 

In extreme situations of incoming process death (`OutOfMemoryError`, `VirtualMachineError`…), we cannot trust the process itself to log what is going on, and it is likely that the logs will never reach us. Instead, we should use an out-of-process logging utility (also called crash handlers). For example, desktop applications for the KDE desktop environment are run by `drkonqi`, which will notice abnormal process death and will generate a bug report. Depending on the situation, that external crash handler may decide to restart the process anew.

In the case of `ThreadDeath`, we introduce an even worse bug: we have interrupted the death of a thread, and it will continue doing whatever the rest of our code does, even though the rest of the system believes it won't. 

Some errors may however be safely dealt with this approach. For example, a `StackOverflowError` doesn't cause major issues to the rest of the system. If it is caught and whatever it was doing was not critical, the process may simply log it and continue to do something else. If this is your situation, you are better off using `#!kotlin catch (e: StackOverflowError)` which will make it clear to readers and avoid creating zombie threads.

Overall, arbitrarily logging throwable and ignoring them is simply a mistake. Even using this approach with `Exception` is too dangerous, as multiple well-known libraries use some specific exceptions as control flow, which will be broken by this approach.

### Rethrowing

A somewhat safer approach than logging and ignoring is to log and rethrow.
```kotlin
try {
	someOperation()
} catch (e: Throwable) {
	log.error("Something went wrong: $e")
	throw e
}
```

This approach is already much better as it will not swallow control-flow throwable like `ThreadDeath`. 

If the process is dying, it still suffers from the same problems as the logging approach. However, the logging approach is often used as a last line of defense (which, as explained, it cannot be), whereas the rethrowing approach is used when we expect something else to be the last line of defense (otherwise, why rethrow?). Since we assume the existence of another line of defense, it isn't a major issue that we attempt a best-effort log. However, we should remain aware that whichever logging operation we attempt is likely to itself fail due to the same condition that failed the process.

Since we are not obscuring the failure, this solution is much less dangerous than the other showcased. However, since the failure is continuing to spread, it is arguably not a recovery solution. 

### Swallowing

Swallowing a failure corresponds to ignoring it, plain and simple. 
```kotlin
try {
	someOperation()
} catch (e: Throwable) {}
```

Swallowing a throwable obviously leads the exact same problems as the logging and ignoring solution, with the added downside that even if the system was able to log the information, we still won't know what went wrong.

Sometimes it may be tempting to think we are swallowing an exception for a good reason. But this is actually almost impossible to ascertain for any well-known throwable: a library may use one of its subclasses as control flow (for example, KotlinX.Coroutines infamously uses `IllegalStateException` for control-flow).

If you decide to swallow an exception, always follow these rules:

1. Catch a specific throwable, not the `Throwable` or `Exception` supertypes. Be careful to choose one that cannot be sub-typed by code that isn't yours, so you know no-one else uses it differently. Ideally, ensure only your code can instantiate it.
2. Always put a comment in the `catch` block that explains why you are catching it and why you think it is safe. List your assumptions so readers can immediately understand why their situation is different.

### Using a default value

Another common recovery attempt is to have some backup code that can obtain a meaningful result even when the main operation failed. There are many ways to write this, but they all suffer from the same problems we already mentioned: if the process is dying, the operation will likely not succeed, and it may break control flow.

Again, the only sane approach in the rare cases where this is actually meaningful is to catch a specific throwable that we know is safe to ignore this way.

## The key takeaways

1. Don't assume that the process can continue after you `catch` something.
2. Always catch the strictness possible type you can.
3. Never catch `Throwable` or `Error`.
4. If you must catch `Exception`, always rethrow it.
5. Rely on external crash handlers for truly critical solutions. Android will automatically restart an app, Kubernetes and Docker Swarm can be configured to automatically restart a server.
