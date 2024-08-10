---
date: 2024-08-12
slug: state-of-kotlin-js
---

# The state of Kotlin/JS in 2024

An overview of the Kotlin/JS ecosystem, and a plea to JetBrains.

<!-- more -->

## If you're new here 

In 2017, I started to convert from a Java developer to a Kotlin developer. I started learning Kotlin because it allowed me to write the code I was already writing, but simpler and more readable. But also, Kotlin had a dream: a _true_ write-once-run-everywhere language. See, Java once had that dream, but the world has changed a lot since then: mobile platforms either don't support it (iOS) or were so out-of-date that it may as well be another language (Android didn't have lambdas back then). And, of course, there's the web.

I don't appreciate it, but it's true: the age of desktop apps is over. Today, everything has to be mobile-first, and probably web-first too. And Kotlin had a dream: instead of having some transpiler layer like some languages have attempted, Kotlin would integrate the transpiler directly as first class target. Kotlin would compile to the JVM, of course, but also to JavaScript, and maybe to the Native platforms too!

In the seven years since then, many things have happened. Many of these dreams have come true. Many others haven't. Let's take a look.

I won't claim to be representative of the Kotlin/JS ecosystem, but I do think I've been following it closely enough to have a good idea of what's going on.

## What is it?

Kotlin is, and isn't, a single language. Kotlin is a language that can compile (or transpile) to multiple targets, including the JVM, JS, LLVM ("native") and WASM. When compiling to each of these platforms, the Kotlin language slightly adapts itself: on Kotlin/JVM, it gains "platform types" which make handling types with unknown nullability easier. On Kotlin/JS, it gains the `dynamic` type that represents dynamically-typed JS objects.

But despite these differences, Kotlin _is_ a single language. It has a single standard library, and a large part of the library ecosystem targets multiple platforms. Kotlin isn't a platform itself: unlike technologies like Xamarin, Flutter or React Native, that force you to buy into their entire model, Kotlin is truly native on all platforms—Kotlin code can always directly call platform-specific APIs.

We're always talking about Kotlin Multiplatform: the ability to share code between multiple platforms. But with Kotlin, you don't *have* to share everything. You could decide (and many people do) to share DTOs and network logic, and use a different UI on each platform, using that platform's native UI tooling. This sets Kotlin apart from other multiplatform technologies: Kotlin is less risky, because there's always an exit.

But today, I'm specifically talking about Kotlin/JS.

## Who is it for?

Let's address the elephant in the room first. Kotlin isn't TypeScript, not even close. They are two languages that have completely different approaches to _everything_.

TypeScript aims to bring type hints to JavaScript. It achieves this by making complex compile-time checks. After compilation, the resulting code is expected to be as close as possible to the equivalent JavaScript code.

Kotlin is, first and foremost, a single language that is compiled to multiple platforms. The Kotlin standard library is extensive and has many features that the JavaScript standard library lacks. Therefore, Kotlin cannot rely on the JavaScript standard library, and must bundle its own. Other libraries, most notably the Ktor client, similarly embed large amounts of code.

Unlike TypeScript, Kotlin _will_ insert runtime checks into the generated JS code. Kotlin developers prefer interacting with the JS ecosystem through wrapper libraries that add layers of safety and Kotlin-first syntax sugar, whereas TS developers usually interact with JS libraries directly. TS accepts type operations that make no sense (such as casting a type into another incompatible one) whereas Kotlin forces these operations to go through an intermediate `dynamic` cast. All these factors taken together make Kotlin code much less likely to encounter "rogue" values: variables that have a runtime value incompatible with their declared type. In fact, they happen so rarely that TS developers don't believe me when I mention it.

TypeScript feels like an alternative universe version of JavaScript. Kotlin feels like a different language entirely.

Some people prefer TypeScript's lightweight approach. Some people prefer Kotlin's uniform experience.

For me, that's the elephant in the room. If you love JavaScript or TypeScript, Kotlin works on a paradigm so different that it will feel alien. That's ok! If you're happy with your existing tools, that's great for you!

But—and I don't know if you will believe me on this one—__not everyone loves JavaScript and TypeScript__. Shocking, I know!

Developers of backend-heavy applications that are often built on the JVM, on C#, and mobile developers, often don't feel at home on the Web. The tooling is so different, everything is so… dynamic. This is who Kotlin/JS is for.

If you use any dependency on top of the standard library, don't expect a Kotlin/JS site to be less than ~500kiB (uncompressed). If you have a large codebase and use the Ktor client, you may quickly reach 1–2MiB. If you're reading this as a full web developer, this may sound like a lot, but in reality it really depends on your situation.

Most Kotlin/JS sites are used for heavy-duty single-page apps that a user spends a large part of their day in. Having a slightly longer loading time is irrelevant: what matters is that the entire experience after initial load is perfectly smooth. On that, Kotlin/JS delivers. 

I wouldn't recommend using Kotlin/JS for showroom websites where you want absolute peak load times (but that doesn't mean you can't do it). I would, however, recommend Kotlin/JS to anyone having a complex application with a complex backend, especially if it's a JVM one, who has to have a website, especially if that website will probably have to do anything non-trivial. And by non-trivial, I mean "having a JS framework": if your client is doing anything related to serialization, validation, authorization…: Kotlin/JS will simplify your life.

This is because of the under-appreciated power of _sharing code_. 

## What code can you share?

Yes.

More seriously, have you ever had a bug because someone changed a DTO server-side and forgot to change it client-side? Or, there was a subtle typo and no one noticed. Or, the backend validation was stricter than the frontend's, and users complained about the poor experience. Or, the frontend's validation was stricter than the backend's and your company lost millions after a major data breach.

In all these situations (and many more), the problem is caused by lack of coherence between duplicated features in different languages. By using a single language on frontend and backend, validation code, DTO code, but also tests, can be shared to ensure the behavior is the same. Developers can discover the joy of refactoring backend code and seeing the frontend code be updated automatically by the IDE. The IDE lets you navigate from one platform to another, detects unused endpoints, and all other features we usually expect IDEs to provide.

The only major language that allows doing this without entirely rewriting either backend or frontend is Kotlin. Sure, you could write both your frontend and backend in JavaScript, but first: that's a bad idea, and second, you will have to rewrite your backend from whatever tech it is currently written in. Kotlin is unique in that it allows you to start sharing code between the JVM, JS, C, ObjectiveC and Android _right now_ (and in the future, Swift and Python may join that list).

So, what code do I advise you share? If you're starting a new project, I recommend you attempt to share your entire domain and network layers: data representation, validation logic, state and error management, authorization, DTOs, serialization, etc.

If you're starting to introduce Kotlin/Multiplatform into an existing project: analyze which objects are duplicated between your platforms and often change. In past projects, these were often simple data types which had complex validation logic. Start by sharing these, and expand from there each time a function that you write for one platform would be beneficial to the other.

## Interoperability with JS/TS

## Compose HTML vs. Compose Multiplatform

[//]: # (TODO: Bitspittle's c4w article)

## UI frameworks

[//]: # (TODO: Kobweb)
[//]: # (TODO: KVision)
[//]: # (TODO: Fritz2)
[//]: # (TODO: Doodle)

## Tooling

[//]: # (TODO: it's slow)
[//]: # (TODO: bundles are large)
[//]: # (TODO: IDE support is bad: no debugger etc)

## Kotlin/JS vs. Kotlin/WASM

## A plea to JetBrains

[//]: # (TODO: JB's answer at KotlinConf)
[//]: # (TODO: what's holding KJS back is that people are afraid it will go away at any point)
[//]: # (TODO: My plea: communicate, give visibility to library authors, stop hiding KJS under the covers)
[//]: # (TODO: NOTHING will happen if you're scaring people away)
[//]: # (TODO: Empower the community. Trust it to thrive.)
