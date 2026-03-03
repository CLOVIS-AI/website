---
date: 
  created: 2026-03-02
slug: gradle-vocabulary
tags:
  - Gradle
---

# Gradle vocabulary: projects, builds, artifacts…

To understand discussions about Gradle, it's necessary to understand how things are named—and that can be quite confusing.

<!-- more -->

This is a short article to summarize the various concepts and list tips and tricks I've learned over the years.

## Build structure

Let's start with the way Gradle structures what we're working with.

### Build

The root entity is the _build_. Usually, a Git repository contains one or more builds.

A build is configured in the `settings.gradle` (Groovy DSL) or `settings.gradle.kts` (Kotlin DSL) file.
The build is responsible for the overall configuration of the entire repository.

The minimum configuration is specifying the name of the root [project](#project):
```kotlin title="settings.gradle.kts"
rootProject.name = "my-great-project"
```

When opening a Gradle repository in IntelliJ, you should select the `settings.gradle.kts` file or the directory that contains it. In some versions of IntelliJ, opening one of the `build.gradle.kts` files can lead to a broken import where the paths are different for the different contributors.

!!! info "IntelliJ equivalent"
    In IntelliJ, a build is called “project”.

!!! tip "IntelliJ configuration"
    Each time a Gradle file is changed, IntelliJ must ask Gradle what changed to update its internal model. You can trigger this manually with ++shift+shift+"Sync all external projects"++.

    Alternatively, you can configure IntelliJ to resynchronize automatically when files change. By default, IntelliJ resynchronizes when files are changed by external programs (for example: switching branches with Git), but not when you modify files inside IntelliJ. This is because IntelliJ cannot provide auto-complete during synchronization. If you rarely modify Gradle configuration, you may prefer that IntelliJ always resynchronizes so you don't have to do it manually after your own changes: select "Any changes" in [File | Settings | Build, Execution, Deployment | Build Tools](jetbrains://idea/settings?name=Build%2C+Execution%2C+Deployment--Build+Tools).

A build typically has the structure:
```shell
foo/ #(1)!
    .gradle/ #(2)!
    .kotlin/ #(3)!
    gradle/ #(4)!
        wrapper/
            gradle-wrapper.jar #(5)!
            gradle-wrapper.properties #(6)!
        gradle-daemon-jvm.properties #(7)!
        libs.versions.toml #(8)!
    # …one or more projects… (9)
    gradle.properties #(10)!
    gradlew #(11)!
    gradlew.bat #(12)!
    settings.gradle.kts #(13)!
```

1. The root build directory.
2. The directory in which Gradle stores local caches and temporary files. This directory should not be checked into version control.
3. The directory in which the Kotlin plugin stores its temporary files. This directory should not be checked into version control. If you don't develop Kotlin applications or libraries, you probably will not have this folder.
4. The directory in which Gradle configuration is placed. **This directory should be checked into version control.**
5. The [Gradle Wrapper](https://docs.gradle.org/current/userguide/gradle_wrapper.html) is a small JAR that contains code to download JVMs and the Gradle Daemon proper. It should be checked into version control to ensure all contributors use the exact same version of Gradle.
6. A simple configuration file for the Gradle Wrapper: for example, it contains the version of Gradle that should be used. To update the Gradle version, use `./gradlew wrapper --gradle-version=X.XX` and then **run `./gradlew wrapper` a second time**.
7. A configuration file that specifies which JVM should be used to run Gradle itself. See [toolchains](#installing-the-right-version-of-java).
8. The list of dependencies used in the build, with the allowed versions.
9. See [Project](#project).
10. A configuration file to enable or disable Gradle features. This file is read by the Gradle Wrapper before the Gradle Daemon itself starts. This is where experimental features are typically controlled, as well as configuration options such as the Gradle Daemon JVM heap size.
11. A script to run the Gradle Wrapper on UNIX derivatives. This file should be checked into version control.
12. A script to run the Gradle Wrapper on Windows. This file should be checked into version control.
13. The settings script file, which configures the overall build.

### Included build

If you work with multiple builds, for example developing an application alongside an open source library, you should know that Gradle can be made aware of this. If you add:
```kotlin title="settings.gradle.kts"
includeBuild("../path-to-the-library")
```
then Gradle will understand the relationship between both builds. When you modify the library and run the application, Gradle will automatically recompile the library just like if it was in the same build. IntelliJ will pick up this configuration as well, and show both repositories in the same window, allow cross-navigation with ++ctrl+left-button++, cross-editing, etc.

This is a much better developer experience than what most people do: publish the library to Maven Local, configure Maven Local in the application, use a snapshot version number, and republish after each change.

### Project

A project is the primary Gradle entity. A project is configured in a `build.gradle` (Groovy DSL) or `build.gradle.kts` (Kotlin DSL) file.

Projects must be registered in the [build](#build) by calling `include` with the path to the directory which contains the `build.gradle` or `build.gradle.kts` file.
```kotlin title="settings.gradle.kts"
include("path/to/project")
```

!!! info "IntelliJ equivalent"
    In IntelliJ, a project is called “module”.

The list of known projects can be displayed with `./gradlew projects`.

Gradle automatically creates a _root project_ called `:`, even if there is no root [build script](#build-script).

Other projects are named after the path from the root. For example, the project in directory `path/to/project` is called `:path:to:project`. Note that you can change the name.

Only the name of the project (not its full path) is used during [dependency resolution](#dependency-management). Therefore, the projects `:foo:entities` and `:bar:entities` will collide when publishing.

### Settings script

The `settings.gradle` or `settings.gradle.kts` file that configures a [build](#build).

### Build script

The `build.gradle` or `build.gradle.kts` file that configures a [project](#project).

Despite its name, it does not configure a [build](#build).

## Installing the right version of Java

Gradle needs a JVM to run, and your application may need one as well.

### Daemon toolchain

To configure which JVM Gradle uses to run itself, you can use `./gradlew updateDaemonJvm --jvm-version=25 --jvm-vendor=adoptium`. To see all available options, run `./gradlew help --task=updateDaemonJvm`. This creates the file `gradle/gradle-daemon-jvm.properties`.

Gradle will scan the system to find a JVM that corresponds to the specified requirements. By default, if no JVM is found, Gradle will crash. Instead, you can teach Gradle how to download JVMs by itself:

```kotlin title="settings.gradle.kts"
plugins {
    id("org.gradle.toolchains.foojay-resolver-convention") version "1.0.0" //(1)!
}
```

1. [Project repository](https://github.com/gradle/foojay-toolchains) • [Version list](https://github.com/gradle/foojay-toolchains/tags).

You can find which toolchain Gradle is running with in the output of `./gradlew version`.

You can list all toolchains Gradle knows about with `./gradlew javaToolchains`.

It is currently only possible to specify an exact JVM version (e.g. Java 25). To make it possible to specify a _minimum_ version, please vote for [Gradle#35148](https://github.com/gradle/gradle/issues/35148).

### Project toolchain

For each JVM project, you can specify which JVM will be used to compile and run. You can use different toolchains in the same project, for example to run the tests with different Java versions.

=== "Java"

    ```kotlin title="build.gradle.kts"
    java {
        toolchain {
            languageVersion.set(JavaLanguageVersion.of(25))
        }
    }
    ```

=== "Kotlin"

    ```kotlin title="build.gradle.kts"
    kotlin {
        jvmToolchain(25)
    }
    ```

The Java and Kotlin configuration options are the same underlying value. If you use both Java and Kotlin in the same [project](#project), there is no need to configure the toolchain twice.

You can configure some tasks to use a different toolchain than the project default. For example, with the `Test` task:
```kotlin title="build.gradle.kts"
tasks.test {
    javaLauncher = javaToolchains.launcherFor {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

Project-level toolchains are a great way to ensure contributors use the exact JVM you intended. If your goal is to ensure that your project is compatible with a specific JVM version, the [Tapmoc plugin](https://github.com/GradleUp/Tapmoc) can provide stronger guarantees without needing to download new JVMs.

## Dependency management

One of Gradle's main features, after all, is handling your dependencies for you.

### Module

A module is the unit of what Gradle publishes. Each module gets its own Maven coordinates: a group and an artifact ID. Whereas the structure of a Maven artifact is represented in a `pom.xml` file, Gradle represents it in a `.module` file.

For example, [here is the `.module` file](https://repo1.maven.org/maven2/dev/opensavvy/pedestal/backbone-jvm/3.2.0/backbone-jvm-3.2.0.module) for one of my libraries, Pedestal Backbone.

A module is written in JSON, and stores all information Gradle needs to understand how to resolve its dependencies: its coordinates, the files it contains, their hashes, the module's dependencies, and more.

Modules are often confused with [projects](#project), because that's where we declare the coordinates and dependencies. However, a single project can contain multiple modules. This is the case, for example, with Kotlin Multiplatform projects, which contain one module per platform.

### Repository

A repository is a place where Gradle can find modules. The most famous, of course, is Maven Central.

Gradle supports different kinds of repositories:

- Maven-style repositories, like `mavenCentral()`, `mavenLocal()`, `google()`, `gradlePluginPortal()`,…
- Ivy-style repositories,
- A local directory, that contains JARs, with `flatDir {}`.

Repositories can be declared at the [project](#project) level:
```kotlin title="build.gradle.kts"
repositories {
	mavenCentral()
	google()
}
```

When Gradle searches for a specific module, it will look in each repository, in the order they are declared in.

**Declaring a repository too early is dangerous!** This repository will be taken as the source of truth, and will be able to replace any of your dependencies by another.

However, if you know that a specific library is only available in a specific repository, declaring it at the end will make dependency download slow, as Gradle tests all other repositories first. In that case, you can declare the repository first, but with an allowlist:
```kotlin
repositories {
	maven {
		name = "opensavvy-gradle-conventions"
		url = uri("https://gitlab.com/api/v4/projects/51233470/packages/maven")
        
		content {
			includeGroupAndSubgroups("dev.opensavvy")
		}
	}
	mavenCentral()
}
```

This way, dependencies in the `dev.opensavvy` group will be looked up first from our GitLab repository (since the Gradle Conventions aren't published to Maven Central), but any other dependency will be directly looked up from Maven Central.

If you use the same repositories in your entire [build](#build), it can be inconvenient to repeat them in each [project](#project). To avoid this, you can declare them directly in your `settings.gradle.kts`:
```kotlin title="settings.gradle.kts"
dependencyResolutionManagement {
	repositories {
		mavenCentral()
	}
}
```

### Configuration

A configuration is a group of dependencies. A module often has multiple configurations, which represent the ‘scope’ in which each dependency is used.

The Java plugin creates (among others):

`api`

:   The dependency will be made available to all users of the module, as if they declared it themselves. 

:   This is best used for libraries that you use in your own API (hence the name). If a type of the library appears in one of your public symbols as a parameter or return type, for example. If users don't have the dependency, they can't call your function!

`implementation`

:    The dependency will be made available to all users of the module, _but only at runtime_. The library will end up in the final JAR/WAR/other, but it won't be visible to their compiler, and won't appear in their auto-complete.

:    This is best used for libraries that you use internally in your own code. For example, if you use a math library, your users never need to call its functions or see it in their auto-complete.

`runtimeOnly`

:    The dependency will be made available to all users of the module, _but only at runtime_. Whereas `implementation` dependencies are available at compile-time to the module itself, `runtimeOnly` dependencies are not. From the point of view of a user, they are identical.

:    This is mainly useful for libraries that are service-loaded, like `slf4j-simple`. You never need to call functions from the library yourself, so you don't want to pollute your auto-complete.

`compileOnly`

:    The dependency will only be available during the compilation of the module itself. It will **not** be available to users of the module.

:    This is mainly useful for things that are removed after compilation, like annotations.

`annotationProcessor`

:    The dependency will not be used by the module itself at all. Instead, it will be registered as an annotation processor, which will be run during the compilation of the module.

The plugin also creates duplicates with the `test` prefix, which are used for unit tests.

However, these configurations are only there for grouping. Gradle uses different ones for resolving dependencies:

`compileClasspath`

:    All the dependencies that should be known to the compiler. These are also the dependencies that are available in the IDE, for example in auto-complete.

:    All `api`, `implementation`, `compileOnly` dependencies are added here. Additionally, all `api` dependencies of the dependencies you use are added as well.

`runtimeClasspath`

:    All the dependencies that are placed in the final JAR/WAR/other, and are known to the JVM during execution.

:    All `api`, `implementation`, `runtimeOnly` dependencies are added here. Additionally, all `api`, `implementation` and `runtimeOnly` dependencies of the dependencies you use are added as well.

Again, tests get their own configurations. 

Dependencies are declared in the `dependencies` block. Each configuration gets its own function name:
```kotlin title="build.gradle.kts"
dependencies {
	implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")
	testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine")
}
```

This does mean that it is possible to use a different version of a given dependency during compilation and when running your application. While this is useful in some situations, for example to test your application against different versions of your dependencies, it usually happens by accident. You can forbid it with:
```kotlin title="build.gradle.kts"
java {
    consistentResolution {
        useCompileClasspathVersions()
    }
}
```
This will enforce that all runtime configurations use the same versions as the compilation used.

You can observe the dependencies of a project with the `:dependencies` task:
```shell
./gradlew :<your project>:dependencies
```
Because projects often have many configurations, you can restrict for a specific configuration. For example, if you want to know the exact list of dependencies used at runtime, you can use:
```shell
./gradlew :<your project>:dependencies --configuration=runtimeClasspath
```
```text hl_lines="1 2 5"
+--- org.jetbrains.kotlin:kotlin-stdlib:2.3.0
|    +--- org.jetbrains:annotations:13.0 -> 26.0.2-1
|    +--- org.jetbrains.kotlin:kotlin-stdlib-jdk8:1.8.0 -> 1.8.10 (c)
|    \--- org.jetbrains.kotlin:kotlin-stdlib-jdk7:1.8.0 -> 1.8.10 (c)
+--- project :core
|    +--- org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.2
|    |    \--- org.jetbrains.kotlinx:kotlinx-coroutines-core-jvm:1.10.2
|    |         +--- org.jetbrains:annotations:23.0.0 -> 26.0.2-1
|    |         +--- org.jetbrains.kotlinx:kotlinx-coroutines-bom:1.10.2
|    |         |    +--- org.jetbrains.kotlinx:kotlinx-coroutines-core-jvm:1.10.2 (c)
|    |         |    +--- org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.2 (c)
|    |         |    +--- org.jetbrains.kotlinx:kotlinx-coroutines-reactive:1.10.2 (c)
|    |         |    \--- org.jetbrains.kotlinx:kotlinx-coroutines-slf4j:1.10.2 (c)
|    |         \--- org.jetbrains.kotlin:kotlin-stdlib:2.1.0 -> 2.3.0 (*)
|    \--- org.jetbrains.kotlin:kotlin-stdlib:2.3.0 (*)
…
```

As we can see, the project depends on the `org.jetbrains.kotlin:kotlin-stdlib:2.3.0` library, which itself depends on `org.jetbrains:annotations:13.0`. However, there is apparently another dependency that requires a more recent version of `org.jetbrains:annotations`, so Gradle opted to use version `26.0.2-1` instead.

It also depends on another project, `:core`, with all its dependencies.

If, instead, you're interested in a specific dependency, you can search the other way around. The `:dependencyInsight` task tells you _why_ a dependency was selected:
```shell
./gradlew :<your project>:dependencyInsight --configuration=runtimeClasspath --dependency=org.jetbrains:annotations
```
```text
org.jetbrains:annotations:26.0.2-1
\--- dev.opensavvy.ktmongo:bson-jvm:0.26.3
     \--- dev.opensavvy.ktmongo:bson:0.26.3
          \--- dev.opensavvy.ktmongo:bson-official-jvm:0.26.3
               \--- dev.opensavvy.ktmongo:bson-official:0.26.3
                    +--- dev.opensavvy.ktmongo:driver-shared-official-jvm:0.26.3
                         \--- dev.opensavvy.ktmongo:driver-shared-official:0.26.3
                              \--- dev.opensavvy.ktmongo:driver-coroutines-jvm:0.26.3
                                   \--- dev.opensavvy.ktmongo:driver-coroutines:0.26.3
                                        \--- project :integration-mongodb
                                             \--- jvmRuntimeClasspath

org.jetbrains:annotations:13.0 -> 26.0.2-1
\--- org.jetbrains.kotlin:kotlin-stdlib:2.3.0
     +--- jvmRuntimeClasspath
```
We learn that Gradle considered the versions `13.0` and `26.0.2-1`, but it chose the latter (because it is the most recent version).

The version `13.0` was considered because it is used by the `org.jetbrains.kotlin:kotlin-stdlib:2.3.0` library.
The version `26.0.2-1` was considered because it is used by the `dev.opensavvy.ktmongo:bson-jvm:0.26.3` library, which is transitively used by the local project `:integration-mongodb`.

This command is particularly useful to answer questions such as:

- Why am I using this library?
- Which version of this library am I using?
- Do I use the version `X` of this library, which is known to have a vulnerability?

### Variant

So far, we have considered that a [module](#module) is a single "thing". A module is a JAR file with some metadata.

A module can actually contain multiple "things", which are used in different situations. For example, the Pedestal Backbone library [has three JARs](https://repo1.maven.org/maven2/dev/opensavvy/pedestal/backbone-jvm/3.2.0/): a `.jar`, a `-sources.jar`, and a `-javadoc.jar`.

The Maven ecosystem has conventions that the `-sources` classifier contains the source code, and the `-javadoc` classifier contains the Javadoc documentation in HTML format (this is how sites like [javadoc.io](https://javadoc.io/doc/dev.opensavvy.prepared/suite) work).

Instead of relying on conventions, Gradle declares each of these in the [module](#module), as separate _variants_.

Each variant is an alternative file, with its own dependencies, for the module. Each user of a module selects one variant for each of their goals. This is why Kotlin Multiplatform users can declare the common code as a dependency, and each platform will download its own artifact: the common module declares one variant per platform, which each depends on the platform-specific module. You can see it in action in the [Pedestal Backbone common module](https://repo1.maven.org/maven2/dev/opensavvy/pedestal/backbone/3.2.0/backbone-3.2.0.module). Each platform gets a variant for compile-time, for runtime, and sources.

Kotlin Multiplatform creates a new module, with its own coordinate, for each platform. Strictly-speaking, that is unnecessary, as the same system of variants could be used to store all files in a single module, and Gradle does support that, though Kotlin doesn't. I'm not sure why modules were split by platform like this: it does slow down download time quite a bit, because each platform has its `.module` file etc that need to be downloaded. I would guess it's for interoperability with non-Gradle build tools.

You can list the variants that a [project](#project) will generate with the task `:outgoingVariants`:
```shell
./gradlew :<your project>:outgoingVariants
```

### Attribute

If a single [module](#module) can contain many variants, how does Gradle know which one to use?

The variant names aren't used. Instead, each [variant](#variant) declares attributes. Each [configuration](#configuration) also declares attributes. When the attributes of a configuration and a variant match, Gradle knows it should use that variant.

For example, the Pedestal Backbone library has the following variant:
```json
{
  "name": "jsRuntimeElements-published",
  "attributes": {
    "org.gradle.category": "library",
    "org.gradle.jvm.environment": "non-jvm",
    "org.gradle.usage": "kotlin-runtime",
    "org.jetbrains.kotlin.platform.type": "js",
    "org.jetbrains.kotlin.js.compiler": "ir"
  },
  "available-at": {
    "url": "../../backbone-js/3.2.0/backbone-js-3.2.0.module",
    "group": "dev.opensavvy.pedestal",
    "module": "backbone-js",
    "version": "3.2.0"
  }
}
```

The attributes tell Gradle that:

- `"org.gradle.category": "library"`: This is a library.
- `"org.gradle.jvm.environment": "non-jvm"`: It isn't meant for the JVM.
- `"org.gradle.usage": "kotlin-runtime"`: It is meant to be used at runtime.
- `"org.jetbrains.kotlin.platform.type": "js"`: It is meant to be used by KotlinJS.
- `"org.jetbrains.kotlin.js.compiler": "ir"`: It is meant to be used with the new KotlinJS compiler.

In this example, this variant will only be used when compiling the JavaScript bundle. During other compilation, Gradle will use other variants.

A community-maintained list of all attributes is available [here](https://github.com/liutikas/gmm-wiki).

In theory, variants provide a very flexible way to tune dependency resolution. For example, a library could provide an implementation that uses `ThreadLocal` on current JVMs and switches to `ScopedValue` only for users with a JVM recent enough to support it (using the attribute `org.gradle.jvm.version`). In practice, I'm not aware of any libraries that do this. In the Kotlin ecosystem, this is even harder because the Kotlin Gradle plugin is hostile to users modifying attributes, since it needs them for platforms. It is still _possible_, but [I wouldn't recommend it](https://gitlab.com/opensavvy/automation/kotlin-js-resources/-/blob/051357ed56d703bd15c0274e004646054feef90d/producer/src/main/kotlin/Kotlin.kt#L76-97).

### Transform

Sometimes, the attributes don't match exactly, but Gradle still knows how to handle the situation. Typically, this happens when Gradle needs a file, but the repository contains a ZIP.

A plugin can register a transform, which tells Gradle how to convert from one attribute to another. Transforms are implicitly called by Gradle during dependency resolution.

Transforms look and feel like [tasks](#task), but they're a completely different system: they are declared differently, are cached differently, do not appear in the task graph, etc.

## Tasks & artifacts

Gradle is a build tool, but it is organized as a task execution engine. The building is actually done by plugins, which declare tasks, configurations, etc.

### Task

A task is a unit of work that can be triggered by the user. Each task does one thing. Complex projects are built by combining many tasks.

Tasks can depend on each other in three different ways:
```kotlin
tasks.foo {
	dependsOn(tasks.other)
	mustRunAfter(tasks.other)
	shouldRunAfter(tasks.other)
}
```

`dependsOn()`

:   `dependsOn()` is **a true dependency**: the task `:foo` can only be executed after the task `:other` has finished. This is most likely because `:foo` uses files that are created by `:other`.

:    If the user executes `./gradlew :foo`, `:other` will be executed, and then `:foo`.

:    If the `:other` task reruns, `:foo` will also rerun.

`mustRunAfter()`

:   `mustRunAfter()` is **an ordering constraint**: the task `:foo` is not allowed to run until after `:other` has finished. Both tasks cannot run concurrently.

:   If the user executes `./gradlew :foo`, `:other` will not be executed, and `:foo` will be executed immediately. However, if the user executes `./gradlew :foo :other`, then `:other` will be executed first, followed by `:foo`.

:   If the `:other` task reruns, `:foo` will not necessarily rerun.

`shouldRunAfter()`

:    `shouldRunAfter()` is **a soft ordering constraint**: we prefer if `:foo` runs after `:other` has finished. However, Gradle is allowed to run both concurrently.

:   If the user executes `./gradlew :foo`, `:other` will not be executed, and `:foo` will be executed immediately. However, if the user executes `./gradlew :foo :other`, then `:other` will be executed first, followed by `:foo`.

:   If the `:other` task reruns, `:foo` will not necessarily rerun.

:   This is useful to separate high-feedback and low-feedback tasks. For example, if you have unit tests (which run fast and catch the majority of mistakes) and integration tests (which are slow and more thorough), specifying `integrationTest.shouldRunAfter(test)` will ensure that unit tests run first when the machine is overloaded. If Gradle can run both concurrently, it will.

The `base` plugin, which is almost always applied, specifies three default lifecycle tasks:

- `assemble`: Generate the production binaries for this [project](#project).
- `check`: Verify the quality of this [project](#project).
- `build`: Do both `assemble` and `check`.

When creating a task, it is good practice to register it as a dependency of either `assemble` or `check`, or none of them if it doesn't fit. This helps new developers by ensuring they can easily verify their changes, no matter what plugins or technologies are used by the project. Here are a few examples and where they fit:

- Building a docker container that contains the project: `assemble`
- Running unit tests: `check` (with the Java or Kotlin plugin, more specifically as a dependency of `test`)
- Running integration tests: `check` (with the Java or Kotlin plugin, still as a dependency of `check` and **NOT** as a dependency of `test`)
- Linting the code, static analysis: `check`
- Publishing the project: none of them
- Deploying the project: none of them

It is bad practice to create a task and define it as a direct dependency of `build`. `build` should never behave differently than explicitly specifying `assemble check`.

To assign one your tasks to a lifecycle task, use `dependsOn()`:
```kotlin
val lint by tasks.registering {
	// …
}

tasks.check {
	dependsOn(lint)
}
```

Additionally, the `base` plugin creates the `clean` task, which does nothing by default. Gradle dynamically generates cleaning tasks for each of your tasks, and you can register them for the main `clean` task similarly:
```kotlin
val lint by tasks.registering {
	// …
	// Let's assume this task creates an HTML report
}

tasks.clean {
	dependsOn(":cleanLint")
}
```

If your tasks are declared correctly, you should never have to write "delete the file x" in the `clean` task. Always use one of the generated cleaning tasks. If they do not do what you expect, it's because you didn't declare your [outputs](#input--output) correctly, which will cause many other issues.

!!! bug "`./gradlew clean build`"
    **Running `clean build` is always an antipattern**. `clean` means "delete files in the project directory", it does **not** mean "execute all tasks again".

    Running `./gradlew clean build` is exactly the same as running `./gradlew build`, because Gradle stores the files in many places outside the project directory (for example, the build cache, which is not necessarily even on your machine). If these two commands somehow give different results, there is something _very_ wrong with your [inputs and outputs](#input--output), and you should fix it before you end up having your CI compile something else than you expect.

    Note that there are multiple types of caches, with new ones being added over time, so even `./gradlew clean; ./gradlew build --no-build-cache` doesn't guarantee that everything will really be recompiled.

    The correct way to tell Gradle to recompile everything is to use `./gradlew build --rerun-tasks`.
    If you want to reexecute a single task, use `./gradlew :<your project>:<task> --rerun`.

As we mentioned in the [project](#project) section, the root project is called `:`. Therefore, these two commands are different:
```shell
./gradlew test
./gradlew :test
```
The first one means "Run `test` in all projects that have a task named `test`", the second one means "Run the `test` task in the root project only". 

Some tasks are already aggregating (e.g. Dokka's `:dokkaHtml`, or Kover's `:koverReport`), so you only need to run them in the root project. Running them in all projects can be much slower.

All tasks that have a description are listed with `./gradlew :<project>:tasks`. Alternatively, you can get more information about a specific task (like its implementation type, or its command-line arguments) with `./gradlew :<project>:help --task <task>`:

```shell
./gradlew :backend:help --task runJvm
```
```text
Paths
     :backend:runJvm
     :gradle:templates:template-app:runJvm

Type
     JavaExec (org.gradle.api.tasks.JavaExec)

Options
     --args     Command line arguments passed to the main class.

     --debug-jvm     Enable debugging for the process. The process is started suspended and listening on port 5005.

     --no-debug-jvm     Disables option --debug-jvm.

     --rerun     Causes the task to be re-run even if up-to-date.

Description
     Run Kotlin jvmMain as a JVM application.

Group
     application
```

### Input & output

For each task, Gradle knows its inputs and outputs. Gradle uses that information to decide when to execute and when not to execute a task.

The single most important precept of a build tool is: the build tool that does less, will be faster.

If the inputs and outputs are declared incorrectly, Gradle will either run the task too often, or will reuse results from a prior execution when it shouldn't.

Outputs are the easiest of the two: they are the files that are created, or modified, by the task. Almost all tasks have outputs: for example, even a test execution produces a test report. The only tasks that do not have outputs are tasks that run something purely for running it, like running the main function of your program.

Inputs are more insidious because it is easy to miss important ones. Inputs can be files, but don't have to be. An example of an often forgotten input is the version of the project, when a task embeds it into the binary. This is easy to fix:
```kotlin
val embedVersion by tasks.registering {
	inputs.property("version", project.version)
	
	// …
}
```
Without this, Gradle may reuse old versions if the only thing that changed was the version number.

A common mistake when creating a task that executes integration tests against your running application is to forget to declare the running application's version as an input. If you modify the application and run it again, the integration tests will not rerun if they do not have an input that corresponds to that information.

It can be difficult to find all inputs and outputs of a task, but they are easy to declare:
```kotlin
val embedVersion by tasks.registering {
    inputs.property("version", project.version)
    inputs.file("version.txt")
    inputs.files("foo/bar.txt", "foo/baz.txt")
    outputs.file("marker")
    // …
}
```

If you want to force a task to always re-execute, no matter what, give it an input that will never be the same value twice:
```kotlin
val alwaysRun by tasks.registering {
	inputs.property("random", Instant.now().toEpochMilli())
}
```

### Artifact

An artifact is a specific version of a specific [variant](#variant) of a [module](#module).

<hr/>

Have you learned something? Share this article with your colleagues!

I may extend this article in the future, to cover the different ways a build can be configured.
