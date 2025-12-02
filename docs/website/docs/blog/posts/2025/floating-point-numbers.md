---
date:
  created: 2025-11-10
slug: ieee754
tags:
  - Kotlin
  - Back to basics
---

# What is the smallest number in Kotlin?

We use floating-point numbers all the time: `Float` and `Double`. Both of these types have a few specific cases that you may not know about. Let's review them.

<!-- more -->

## Numbers and IEEE754

In Kotlin, we have a few different numeric types, that all inherit from `kotlin.Number`:

- Signed integer types: `Byte`, `Short`, `Int` and `Long`
- Unsigned integer types: `UByte`, `UShort`, `UInt` and `ULong`
- Floating-point types: `Float` and `Double`

### Integers

Integer types are simple: each different bit pattern is a different value, and each value is a different pattern. All positive values are encoded as the value itself in binary:

- 0: `00000000 00000000 00000000 00000000`
- 1: `00000000 00000000 00000000 00000001`
- 2: `00000000 00000000 00000000 00000010`
- 3: `00000000 00000000 00000000 00000011`
- and so on.

Negative values are encoded in [two's-complement](https://en.wikipedia.org/wiki/Two%27s_complement), which is slightly more complex. In everyday life as Kotlin developers, this doesn't matter, as we don't tend to access bit patterns directly.

The only important thing to keep in mind with integers is that they have one more negative value than they have positive values. This is because 0 is considered a positive value, and there are -2,147,483,648 values of each sign:

- Positive: `0..2147483647`
- Negative: `-2147483648..-1`

Just be aware that `-2147483648 * -1` is… itself. In general, mathematics break down when we're near the edge, and it's rare to need numbers that big anyway. If you're worried that your value may approach 2 billion, switch to a `UInt` (4 billion) or a `Long` (18 quintillion).

### Floating-point numbers

Floating-point numbers, however, are more tricky. It's more difficult to avoid their edge cases because they may happen in other places than simply the edge.

Floating-point numbers are standardized in IEEE754 and should behave the same in all programming languages, though of course some implementations take some liberties. This article uses Kotlin examples, but all the concepts should apply the same elsewhere.

As a quick refresher, floating-point numbers are encoded as an exponent and a significand. They are computed as `significand × 2^exponent`. The final number always has the same number of significative digits (in base 2, not necessarily in base 10), but the exponent allows shifting the total value towards larger or smaller values, similarly to scientific notation.

Therefore, small exponent values allow declaring very precise numbers, whereas large exponent values allow representing immense numbers at the cost of precision.

## Equality

When a value cannot be represented, it is rounded to the closest representable one. As a consequence, an operation that looks trivial may not be representable and may round differently than a value specified by the user.

The most well-known case is the sum of `0.1` and `0.2`, which does not exactly return `0.3`:

<iframe src="https://pl.kotl.in/Agl8UHFkR?from=2&to=7&theme=darcula" style="width: 100%; min-height: 250px" frameborder=0 scrolling="no" loading="lazy"></iframe>

From this result we find an important rule: we should almost never compare floating-point numbers by equality. Instead, compare with ranges.

## It's all about precision

Because large numbers have low precision, combining them with values of low precision does not change their value:

<iframe src="https://pl.kotl.in/4CdbZEM_h?from=2&to=11&theme=darcula" style="width: 100%; min-height: 320px" frameborder=0 scrolling="no" loading="lazy"></iframe>

As we can see, the addition of small values does nothing, as each sum rounds to the current value.

However, if we do the exact same operation the other way around, the small values have a chance to compound and thus make a difference.

<iframe src="https://pl.kotl.in/hE5xJpP9U?from=2&to=11&theme=darcula" style="width: 100%; min-height: 320px" frameborder=0 scrolling="no" loading="lazy"></iframe>

Although we've done the exact same mathematical operation, the result is different based on the order of terms. Over the lifetime of a complex program, this can lead to numbers being quite different from expected. Therefore, when you can, prefer computing small values together before combining them with larger values.

## The greatest number

Now that we understand how floating-point numbers use precision, we can try to sort them. Starting from 0, let's look at all the positive numbers.

- The smallest positive number is 0.
- As we've seen, floating-point numbers are most precise when they're near zero and less precise when they're far from zero. The smallest positive number greater than 0 is thus the most precise number, which is called `MIN_VALUE`. For `Float` it's 1.4E-45, for `Double` it's 4.9E-324. 
    - It is incorrect to call this value the precision: it is the _maximum precision_, but any value further from zero will be much less precise.
    - Of course, it is also incorrect to call this value the minimum: floating-point numbers can be negative, and even zero is lesser.
- After that, we find all the typical numbers we expect: 0.1, 7.5, etc. Again, they may be rounded to the closest possible value.
- An important value is 2⁵³-1 (9,007,199,254,740,991). It is the largest integer that can be represented in a `Double`. After that, the precision becomes so low that some integers are rounded to other values. JavaScript only has double-precision numbers, so it is the largest integer representable in JavaScript.
    - Because this value is greater than 2³¹-1 and 2³²-1, the maximum values of `Int` and `UInt`, both of these types are represented as native numbers when compiling to JavaScript; as are all smaller integer types.
    - The Kotlin standard library uses more complex types to represent `Long` and `ULong`, which may both exceed that size, with respective maximum values of 2⁶³-1 and 2⁶⁴-1.
- After that, numbers continue normally up to the largest possible "proper" number: `MAX_VALUE`. For `Float`, it's 3.4028235E38, and for `Double` it's 1.7976931348623157E308.

That's already quite a lot of information!

However, much like 0 is smaller than `MIN_VALUE`, there is a special value that is greater than `MAX_VALUE`: ∞. ∞ is greater than all other floating-point numbers.

There are multiple bit patterns that represent ∞, but the standard library doesn't attach a significance to them, so we can treat them all as one value, called `POSITIVE_INFINITY` and displayed as `Infinity`.

A typical operation that returns ∞ is the division of a positive number by zero:

<iframe src="https://pl.kotl.in/OVdAeJM5M?from=2&theme=darcula" style="width: 100%; min-height: 170px" frameborder=0 scrolling="no" loading="lazy"></iframe>

Note that division by zero of **integers** throws an exception; division by zero of floating-point numbers returns ∞.

## Negative numbers

Unlike integers, floating-point numbers are symmetric around zero. For any positive floating-point number, there is a symmetric negative floating-point number of the same absolute value.

The greatest negative floating-point number is therefore… −0.

Per the specification, −0 has an identical value to 0 (they are both `==`). However, the specification also says that −0 should always be sorted as lesser than 0.

This brings us to our first strange Kotlin edge case: the language must ensure that −0 is equal to 0, but also that they are sorted in the correct order. However, these two things are done with the same concept: the `compareTo` operator!

Well, did you know Kotlin's comparison operators (`>`, `>=`, `<=`, `<`) don't actually map to the `compareTo` method, unlike what's written in the operator overloading documentation? Well, they _almost_ do. In fact, this is the only case I know where they don't. See for yourself:

<iframe src="https://pl.kotl.in/tFB9hkAS3?from=2&to=5&theme=darcula" style="width: 100%; min-height: 200px" frameborder=0 scrolling="no" loading="lazy"></iframe>

The rule in Kotlin is that `compareTo` must return 0 if the two numbers are equal, a negative number if the first is lesser, and a positive number if the first is greater. Here, it returns −1, meaning that the first number is lesser. However, the `<` operator returns `false`.

This is, of course, not a bug—it's documented [in the Kotlin specification](https://kotlinlang.org/spec/kotlin-spec.html#comparison-expressions)—but it's a very specific edge case that almost no one is aware of.

The other negative floating-point numbers behave as expected, symmetrically to their positive equivalent.

−∞, written `NEGATIVE_INFINITY`, is the smallest possible floating-point number. Typically, it is returned when dividing a negative number by zero.

## It's a matter of definition

At this point, you may think that we've answered the question: the floating-point numbers, in descending order, are: 

1. `Double.POSITIVE_INFINITY`
2. `Double.MAX_VALUE`
3. All other positive numbers
4. `Double.MIN_VALUE`
5. `0.0`
6. `-0.0` (which is equal to `0.0` but still sorted as lesser)
7. `-Double.MIN_VALUE`
8. All other negative numbers
9. `-Double.MAX_VALUE`
10. `Double.NEGATIVE_INFINITY`.

And, depending on your definition of a number, then that's right!

As you may have guessed, the problem is the last special value introduced by IEEE754: `Double.NaN`, or "Not a number". `NaN` is an error mechanism, like `null` is for references. Mathematically, it isn't a number at all, just like `null` isn't a `String`. But it is a possible value of `Float` and `Double`, which can be sorted, so it must be placed _somewhere_ when we sort a list.

`NaN` is weird in many ways. Most importantly, `NaN` is _not equal to itself_. If you want to check whether a value is `NaN`, we can use the `isNaN()` function.

The typical operation that returns `NaN` would be the square root of a negative number.

Since `NaN` is not a number, it is _unordered_ when compared to all other numbers:

<iframe src="https://pl.kotl.in/oStv7Tg4p?from=2&to=6&theme=darcula" style="width: 100%; min-height: 250px" frameborder=0 scrolling="no" loading="lazy"></iframe>

As you can see, all operators always return `false`. This is the case even comparing `NaN` with itself:

<iframe src="https://pl.kotl.in/tG6ifiBWY?from=2&to=6&theme=darcula" style="width: 100%; min-height: 250px" frameborder=0 scrolling="no" loading="lazy"></iframe>

But, how is this possible? We've learned that the comparison operators call the method `compareTo` to know the order. `compareTo` returns an integer, and the rules are based on its sign and zeroness. There are only three possible cases: the returned integer is strictly positive, strictly negative, or zero. Therefore, at least one of the comparison operators should return `true`.

Well, just like with −0, the Kotlin specification has a different behavior for the comparison operators and the `compareTo` method when one of the operands is `NaN`.

So, how can we sort lists that contain a `NaN`? Well, just like with −0, the specification gives a different behavior to the comparison operators and to the `compareTo` method in this case.

<iframe src="https://pl.kotl.in/mMtxSENhs?from=2&to=10&theme=darcula" style="width: 100%; min-height: 400px" frameborder=0 scrolling="no" loading="lazy"></iframe>

When comparing `NaN` with any number, `compareTo` returns 1, meaning that the other number is greater. Therefore, `NaN` will always be sorted as the smallest possible value when sorting a list. But does it really count as the smallest _number_?

Note that `NaN` compared to `NaN` returns 0, which should mean that they are equal, but as we've seen before, they are not. This is necessary to ensure consistent sorting orders.

Of course, it would be too simple if that was all there was to it. IEEE754 has `+NaN` (greater than all possible numbers) and `-NaN` (lesser than all possible numbers). In Kotlin, I'm not aware of any distinction between them: the standard library only has `Double.NaN`, which evidently represents `-NaN`. I guess you could create a `Double` with the bit pattern of a `+NaN` and I suppose it would be sorted as mandated by IEEE754, but I doubt that you'll encounter this in a real program.

It still doesn't stop there, however. There are actually multiple different encoding patterns that are all read as `NaN`, grouped in two categories: quiet `NaN` (often written `qNaN`) and signaling `NaN` (often written `sNaN`). As per IEEE754, from greater to lesser:

1. `+qNaN`
2. `+sNaN`
3. All non-`NaN` values, in the order we've seen before
4. `-sNaN`
5. `-qNaN`

Mathematical operations that return `NaN` most often return `qNaN`. Signaling `NaN` is meant as a special value to set your uninitialized values to, so you can detect that you've read uninitialized data. However, this doesn't seem relevant to Kotlin.

- The decision of which `NaN` are `qNaN` or `sNaN` seems to be CPU-dependent.
- The JVM [doesn't have any specific behavior depending on which `NaN` it is](https://docs.oracle.com/javase/8/docs/api/java/lang/Double.html#longBitsToDouble-long-), though you could decode the bit encoding yourself and implement whatever you wanted. On some CPU architectures, `sNaN` may even be silently converted to `qNaN` at any point during the program.
- JavaScript implementations also do change one `NaN` into another silently.
- Native [keeps the distinction between `qNaN` and `sNaN` on most architectures](https://llvm.org/docs/LangRef.html#behavior-of-floating-point-nan-values) with a few caveats.
- Wasm [doesn't have `sNaN` at all](https://github.com/WebAssembly/design/issues/1463#issuecomment-1304424091).

## Conclusion

All of this to say;

- For all intents and purposes, we do not, and should not, care about the different `NaN` in Kotlin.
- If your definition of a number includes `NaN`, then `NaN` is the smallest possible number in Kotlin.
- If your definition of a number doesn't include `NaN`, then `NEGATIVE_INFINITY` is the smallest possible number in Kotlin.
