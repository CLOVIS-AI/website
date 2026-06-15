---
date: 
  created: 2026-06-15
slug: measure-time
tags:
  - Back to basics
---

# Don't measure elapsed time by subtracting timestamps

The most common way of measuring elapsed time is flawed.

<!-- more -->

## How it's taught

Let's say you want to know how long a computation takes. If you search online, you'll invariably get an answer that captures a first timestamp, then a second one, and subtracts them.

Here are a few examples. Each is the first result I get on Google for “&lt;language&gt; measure time”.

=== "Python"

	```python
	import time

	start = time.time()
	print("hello")
	end = time.time()
	print(end - start)
	```
	Found on [StackOverflow](https://stackoverflow.com/a/7370824).

=== "Rust"

	```rust
	let start = SystemTime::now().duration_since(UNIX_EPOCH).unwrap();
	//do my stuff
	let end = SystemTime::now().duration_since(UNIX_EPOCH).unwrap();
	```
	Found on the [Rust forum](https://users.rust-lang.org/t/what-is-a-proper-way-to-measure-time-execution/91680).

=== "C"

	```c
	clock_t start = clock();
	/*Do something*/
	clock_t end = clock();
	float seconds = (float)(end - start) / CLOCKS_PER_SEC;
	```
	Found on [StackOverflow](https://stackoverflow.com/a/3557272).

=== "C++"

	```cpp
	//***C++11 Style:***
	#include <chrono>
	
	std::chrono::steady_clock::time_point begin = std::chrono::steady_clock::now();
	std::chrono::steady_clock::time_point end = std::chrono::steady_clock::now();
	
	std::cout << "Time difference = " << std::chrono::duration_cast<std::chrono::microseconds>(end - begin).count() << "[µs]" << std::endl;
	std::cout << "Time difference = " << std::chrono::duration_cast<std::chrono::nanoseconds> (end - begin).count() << "[ns]" << std::endl;
	```
	Found on [StackOverflow](https://stackoverflow.com/a/27739925).

=== "Haskell"

	Haskell requires [an external library](https://github.com/chrissound/FuckItTimer):
	```haskell
	start' <- start
	timerc start' "begin"
	print "hello"
	timerc start' "after printing hello"
	benchmark
	timerc start' "end"
	end <- getVals start'
	forM_ (timert end) putStrLn
	```
	Found on [StackOverflow](https://stackoverflow.com/a/64543032): it's only the third answer!

=== "JS"

	```js
	console.time('doSomething')

	doSomething()   // <---- The function you're measuring time for
	
	console.timeEnd('doSomething')
	```
	Found on [StackOverflow](https://stackoverflow.com/a/1975103).

=== "PHP"

	```php
	$start = microtime(true);
	while (...) {
	
	}
	$time_elapsed_secs = microtime(true) - $start;
	```
	Found on [StackOverflow](https://stackoverflow.com/a/6245978).

=== "Lua"

	```lua
	start_time = os.time()
	<some code>
	end_time = os.time()
	elapsed_time = os.difftime(end_time, start_time)
	```
	Found on [StackOverflow](https://stackoverflow.com/a/15666840).

***

Interestingly, the first result for these languages correctly points to the problem and proposes a solution:

- [Java](https://stackoverflow.com/a/1776053)
- [Kotlin](https://kotlinlang.org/docs/time-measurement.html#measure-time)
- [C#](https://dev.to/francis04j/the-efficient-way-to-measure-time-in-net-1i1k)

The first result for these languages does not have the bug, but doesn't explain it either:

- [JS](https://stackoverflow.com/a/1975103) (which gives both the correct and incorrect answers, but doesn't say which is which)
- [Go](https://coderwall.com/p/cp5fya/measuring-execution-time-in-go)

## Where it goes wrong

All these examples are based on four different steps:

1. Measure the current time once.
2. Perform an operation.
3. Measure the current time a second time.
4. Diff the two times.

The output is meaningless because there are many situations in which it doesn't measure the time the operation took.

Here is a non-exhaustive list:

- **The user's device reconnected to the internet and offset clock drift.**
  This is a common bug in devices that are used for a long time without internet connection (e.g. on a trip).
  A low-range laptop can drift up to one minute every week; time measurements can instantly jump by that much forwards or backwards when connectivity comes back.
- **The device went to sleep during the operation.**
  I have noticed that Claude Code has this bug: if the device goes to sleep, it reports hours of execution time even though the operation was paused almost immediately.
- **The user could configure the system time.**
  It's rare, but it can happen.
  If you set time back, the `ping` command [waits until the time catches back to what it expects](https://blog.habets.se/2010/09/gettimeofday-should-never-be-used-to-measure-time.html#ping) before sending a new message. 
- **There was a leap second and the clock needs to be corrected.**
  Yes, [leap seconds are a thing](https://en.wikipedia.org/wiki/Leap_second).
  In 2009, [many Oracle databases rebooted](https://community.oracle.com/mosc/discussion/3730329/solaris-ntp-leap-second-event-can-cause-oracle-clusterware-node-reboot) due to the database thinking the time was incoherent.
  In 2017, [Cloudflare crashed](https://blog.cloudflare.com/how-and-why-the-leap-second-affected-cloudflare-dns/) for the same reason.
- **It's not precise!**
  In most hardware, the system clock has a precision of 1 millisecond. To measure a short operation, this may lead to inaccurate results.

To avoid these problems, systems have two physical clocks. 

- The system clock measures a timestamp (also called ‘instant’), often a number of seconds or milliseconds since Jan 1st, 1970 at 00:00 UTC.
- The monotonic clock has an arbitrary always-increasing value.

While the system clock is great for knowing what time it is, it often only has millisecond precision and is affected by clock drift and clock jumps when the clock synchronizes. That makes it a poor candidate for measuring how long operations take.

The monotonic clock is typically more precise and never has to resynchronize, since it doesn't represent any specific value.

- The user's device reconnected to the internet: nothing happens to the monotonic clock.
- The device went to sleep during the operation: the monotonic clock also goes to sleep, so the final measured time corresponds to how long the device was awake for.
- The user could configure the system time: nothing happens to the monotonic clock.

The way to access the monotonic clock is different for every language. Here are a few solutions.

## How to fix it

=== "Java"

	```java
	long startTime = System.nanoTime();
	// …the code being measured…
	long elapsedNanoSeconds = System.nanoTime() - startTime;
	```
	This is only correct because Java explicitly specifies that [`System.nanoTime()` uses the monotonic clock](https://docs.oracle.com/javase/8/docs/api/java/lang/System.html#nanoTime--).
	Using `System.currentTimeMillis()` would use the system clock and be incorrect.

	I find Java interesting because it's _clearly_ the worst way to communicate this distinction to users, as the name implies it's only a precision difference, but the two functions have a completely different meaning.
	Yet, it's also the only language where developer reliably knows about this problem, so it must be working somehow.

=== "Kotlin"

	```kotlin
	val elapsed = measureTime {
	    // …
	}
	```
	The [`measureTime`](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.time/measure-time.html) function uses the monotonic clock.
	
	If the two bounds you want to measure are not in the same function call, you can use the more verbose equivalent:
	```kotlin
	val startTime = TimeSource.Monotonic.markNow()
	// …
	val elapsed = startTime.elapsedNow()
	```
	Kotlin's `Duration` class is compiled down to a `long`, so this code performs no allocations.

=== "C#"

	```c#
	Stopwatch stopwatch = new Stopwatch();
	stopwatch.Start();
	// Your code: Doing something e.g Thread.Sleep(3000);
	stopwatch.Stop();
	TimeSpan elapsedTime = stopwatch.Elapsed;
	```
	To avoid allocations, the following equivalent can be used:
	```c#
	long startTime = Stopwatch.GetTimestamp();
	// Your code: Doing something e.g Thread.Sleep(3000);
	TimeSpan elapsedTime = Stopwatch.GetElapsedTime(startTime);
	```

	Found on [dev.to](https://dev.to/francis04j/the-efficient-way-to-measure-time-in-net-1i1k).

=== "JS"

	```js
	const startTime = performance.now()
	
	doSomething()   // <---- measured code goes between startTime and endTime
	
	const endTime = performance.now()
	
	console.log(`Call to doSomething took ${endTime - startTime} milliseconds`)
	```
	[`performance.now()`](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now) uses the monotonic clock and has better precision than `Date.now()`.

=== "Go"

	```go
	start := time.Now()
	
	r := new(big.Int)
	fmt.Println(r.Binomial(1000, 10))
	
	elapsed := time.Since(start)
	```
	Go's [`time.Now()`](https://pkg.go.dev/time) measures _both_ the system clock and the monotonic clock, and uses either depending on the subsequent operation.
	`time.Since` uses the measurement from the monotonic clock, making this idiom safe.

=== "Python"

	Python doesn't have a direct built-in solution that corresponds to our criteria.

	`time.perf_counter()` has high precision but includes sleep time:
	```python
	t = time.perf_counter()
	#do some stuff
	elapsed_time = time.perf_counter() - t
	```

	Alternatively, `time.process_time()` does not include time sleeping, but also does not include time spent in I/O:
	```python
	t = time.process_time()
	#do some stuff
	elapsed_time = time.process_time() - t
	```

=== "Rust"

	Rust's `Instant` does not specify whether sleep time is included, but it does otherwise use a monotonic clock.
	```rust
	let now = Instant::now();
	
	// we sleep for 2 seconds
	sleep(Duration::new(2, 0));
	// it prints '2'
	println!("{}", now.elapsed().as_secs());
	```

=== "C"

	```c
	struct timespec ts_start;
	struct timespec ts_end;

	clock_gettime(CLOCK_MONOTONIC, &ts_start);

	// …operation…

	clock_gettime(CLOCK_MONOTONIC, &ts_end);
	
	struct timespec result;
	result.tv_sec = ts_end.tv_sec - ts_start.tv_sec;
	result.tv_nsec = ts_end.tv_nsec - ts_start.tv_nsec;
	while (result.tv_nsec > 1000000000) {
	    result.tv_sec++;
	    result.tv_nsec -= 1000000000;
	}
	while (result.tv_nsec < 0) {
	    result.tv_sec--;
	    result.tv_nsec += 1000000000;
	}
	```
	Found on the [Habets blog](https://blog.habets.se/2010/09/gettimeofday-should-never-be-used-to-measure-time.html), which is about this same problem and provided multiple of the examples mentioned above.

=== "C++"

	```cpp
	auto start = std::chrono::steady_clock::now();

	// …operation…

	auto end = std::chrono::steady_clock::now();

	auto elapsed_ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
	auto elapsed_us = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
	
	// 5. Output the results
	std::cout << "Execution time: " << elapsed_ms.count() << " ms\n";
	std::cout << "Execution time: " << elapsed_us.count() << " us (\u03bcs)\n";
	```
	[`steady_clock`](https://en.cppreference.com/cpp/chrono/steady_clock) is specified to be monotonic.

=== "PHP"

	```php
	$time = hrtime(true);
	// …operation…
	$time = hrtime(true) - start;
	```
	[`hrtime`](https://www.php.net/manual/en/function.hrtime.php) is specified to be monotonic.

***

A few additional remarks:

- **Java is the winner at “the highest number of developers using this language know about this problem”.** This is surprising to me because it has the worst syntactical differentiation between the two clocks. Because people are often confused, they tend to learn about the difference.
- **Kotlin is the winner at “the simplest code is the correct one”.** The `measureTime` method perfectly explains what it does with no boilerplate.
- **Go is the winner at “making code correct”.** It's actually difficult to create this bug in Go, even on purpose, because all comparisons use the monotonic clock no matter what. However, this is implemented at the cost of every call to `time.Now()` making two measurements, one of which is always ignored. Each measurement is cheap, so this is unlikely to cause performance issues in most situations.

If you have an example in another language, please send it to me!

Dates and times are a common source of bugs due to edge cases. However, most languages provide ways to avoid these bugs without needing to know about each of them. We just need to know which method to use when. Hopefully, this article clarifies the use-case of measuring the passage of time.
