---
date:
  created: 2025-08-18
slug: gradle-1st-task
tags:
  - Gradle
---

# Your first Gradle task in 2025

Best practices in Gradle have changed quite a few times over the years, but the online tutorials are rarely updated.
Let's review what is recommended as of Gradle 8, in 2025.

<!-- more -->

## Who is this guide for?

In Gradle parlance, there are two types of users: application developers and build engineers.

Application developers are people who use Gradle to build their projects. Application developers want to focus on their projects, they don't want to spend time learning the intricacies of the build tool. They want to specify what they are doing, let the build tool do its job, add dependencies, remove dependencies, but nothing too complex. Typically, application developers are expected to be familiar with `build.gradle.kts` files.

Build engineers are specialists who understand the build tool. Their role is to ensure everything is reliable, upgrade to new major versions, optimize the build and avoid duplication. Typically, build engineers are expected to be familiar with the creation of custom plugins, to understand how to publish artifacts to a registry, etc.

This guide is aimed at application developers who want to get a bit more out of Gradle.

## What can this guide be used for?

Gradle build scripts (`build.gradle.kts` files) should overall not contain logic. Especially in larger projects, any build logic should be enclosed in custom plugins which are then applied in build scripts. Build scripts should be purely declarative to ensure they are easy to configure for application developers.

If you're working with any kind of large build, you're probably used to having multiple modules in the same repository that are very similar to each other. For example, if you're using some kind of [hexagonal architecture](hexagonal.md), you probably have a few modules that contain database logic. Let's take a look at such an example:

```kotlin
plugins {
	kotlin("jvm") version "2.1.20"
	kotlin("plugin.power-assert") version "2.1.20"
	id("io.kotest.multiplatform") version "6.0.0.M3"
	id("io.github.arturbosh.detekt") version "1.23.8"
}

kotlin {
	jvmToolchain(21)
}

powerAssert {
	functions = listOf(
		"kotlin.check",
		"kotlin.checkNotNull",
		"kotlin.assert",
		"kotlin.test.assertTrue",
		"kotlin.test.assertEquals",
		"kotlin.test.assertNull"
	)
}

repositories {
	mavenCentral()
}

dependencies {
	implementation(projects.base.common)
	implementation(projects.base.database)
	implementation(libs.your.database.driver)
	implementation(libs.something.specific)
}

publishing {
	repositories {
		mavenCentral()
	}
}
```

You've probably seen projects where build scripts are quite long like this, and there is a lot of duplication between different build scripts.
This is not how you're supposed to use Gradle! Much like how regular code should be refactored to avoid duplication rot (when one copy becomes different than the others over time), build scripts should as well. The modern way to do this duplication is with convention plugins and **_not_ with `:buildSrc`**, but that's a story for another time.

In a properly maintained build, database modules should simply look like:

```kotlin
plugins {
	id("your.project.databaseModule")
}

dependencies {
	implementation(libs.something.specific)
}
```

Everything else should be abstracted away within the convention plugin. Build scripts should rarely be longer than a few dozen lines. Not only is that easier to maintain, it makes Gradle faster.

Still, sometimes, we need some kind of additional logic that is specific to a single project, and there's no worth going through convention plugins.
This article is to guide through these times. We will be looking at a few steps to improve build scripts.

## The basics

In this guide, we will concentrate on creating Gradle tasks. A task is a thing you can invoke on the Gradle command line. Each task typically contains one more action and some configuration.

A basic task can be created using:

```kotlin
val taskName by tasks.registering {
	// This is executed at configuration-time
	println("Configuring!")

	doLast {
		// This is executed at execution-time
		println("Executing!")
	}
}
```

If you run `./gradlew :taskName`, you will see:

```text
…
> Configuring project :example
Configuring!

> Task :taskName
Executing!
…
```

If you have the configuration cache enabled, and run the same `./gradlew :taskName` command again, you will see:

```text
…
Reusing configuration cache.

> Task :taskName
Executing!
…
```

The configuration block hasn't run again because Gradle remembers each property of the task that was set. In this example, the task didn't have any configuration, and Gradle knows it, so it knows there is no configuration that needs to execute.

Our goals are to make the build more idiomatic so it's easier for people who don't know it to find things they're looking for, and to make the build faster.

For the rest of this article, we'll be using an example of a project that includes a JS frontend handled via NPM. We want to teach Gradle how to build and test the project so it can be included in a JVM server's resources.

Currently, the build script looks like:

```kotlin
plugins {
	alias(libs.plugins.node)
	id("maven-publish")
}

node {
	version.set(libs.versions.node)
	npmVersion.set(libs.versions.npm)
	download.set(true)
}

val install by tasks.creating(NpmTask::class) {
	dependsOn("npmSetup")
	npmCommand.set(listOf("install", "--target-arch=x64"))
}

val uiBuild by tasks.creating(NpmTask::class) {
	dependsOn("install")
	npmCommand.set(listOf("run", "build"))
}

val app by tasks.creating(Zip::class) {
	dependsOn("uiBuild")
	from("dist")
}

tasks.getByName("build").configure {
	dependsOn("app")
}

publishing {
	publications {
		register("front", MavenPublication::class) {
			artifact(app) {
				artifactId = "ui"
			}
		}
	}
}
```

## Document what your task does

The first step towards improving this build script is to document what the tasks do. Once this is done, our tasks will appear in `./gradlew :help` and will be categorized in the IDE. To do so, simply add:

```kotlin
group = "front"
description = "The description of the task"
```

to all tasks. The group is the category of the task, visible in `:help` and IntelliJ's UI.

For example;

```kotlin
val install by tasks.creating(NpmTask::class) {
	group = "front"
	description = "Installs dependencies from NPM."

	dependsOn("npmSetup")
	npmCommand.set(listOf("install", "--target-arch=x64"))
}
```

Don't just repeat what's visible in the name of the task, though. Try to provide information useful to the next developers.

## Be lazy

The most effective way to be faster is to do less work. This is especially true of build tools.

Before Gradle can do anything, it must analyze the configuration of each project. The longer it spends on configuring the projects, the longer the build times, which slows down the feedback cycle. Sadly, very old versions of Gradle didn't offer any way to avoid useless configuration work. To maintain backwards compatibility, all the old APIs are still there, but they really shouldn't be used.

| Avoid!                      | Prefer!                       |
|-----------------------------|-------------------------------|
| `tasks.create` and variants | `tasks.register` and variants |
| `tasks.getByName`           | `tasks.named`                 |
| `tasks.withType`            | `tasks.configureEach`         |

You can learn more about these methods [here](https://docs.gradle.org/current/userguide/task_configuration_avoidance.html).

We thus replace
```kotlin
val app by tasks.creating(Zip::class) {
	dependsOn("uiBuild")
	from("dist")
}

tasks.getByName("build").configure {
	dependsOn("app")
}
```
by 
```kotlin
val app by tasks.registering(Zip::class) {
	dependsOn("uiBuild")
	from("dist")
}

tasks.named("build") {
	dependsOn("app")
}
```

The build task could be simplified further, because Gradle generates type-safe accessors for tasks from plugins in the `plugins {}` block:
```kotlin
tasks.build {
	dependsOn("app")
}
```
Similarly, we can type-safely use the `app` task we just declared in the dependency:
```kotlin
tasks.build {
	dependsOn(app)
}
```

We thus have:
```kotlin
plugins {
    alias(libs.plugins.node)
    id("maven-publish")
}

node {
    version.set(libs.versions.node)
    npmVersion.set(libs.versions.npm)
    download.set(true)
}

val install by tasks.registering(NpmTask::class) {
    dependsOn(tasks.npmSetup)
    npmCommand.set(listOf("install", "--target-arch=x64"))
}

val uiBuild by tasks.registering(NpmTask::class) {
    dependsOn(install)
    npmCommand.set(listOf("run", "build"))
}

val app by tasks.registering(Zip::class) {
    dependsOn(uiBuild)
    from("dist")
}

tasks.build {
	dependsOn(app)
}

publishing {
    publications {
        register("front", MavenPublication::class) {
            artifact(app) {
                artifactId = "ui"
            }
        }
    }
}
```

## Split tasks into independent units

The task that spends the more time now is `uiBuild`, which invokes `npm run build`. Let's see what this consists of, looking at the `package.json` file:

```json
{
	"name": "foo",
	"version": "0.1.0",
	"scripts": {
		"build": "npm run generate-api && ng lint && ng build --configuration=production"
	}
}
```

NPM is a dependency management system, not a task executor: it doesn't understand anymore about these commands than what's written here. However, Gradle is an entire build tool, and could execute these much faster if it knew about them.

For example, `ng lint` and `ng build --configuration=production` are two tasks that do not modify the source code (`ng lint` prints its output to the standard output and `ng build` generates a `dist` directory). These two tasks could run in parallel. However, both of these tasks must run after `generate-api`, as it's not possible to compile the codebase if the API stubs aren't available.

Instead of having a single task that does all three, let's create three different tasks and teach Gradle how to execute them.
```kotlin
val generateApi by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Generates the API stubs"

	npmCommand.set(listOf("run", "generate-api"))

	dependsOn(install)
}

val lint by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Verifies that the source code corresponds to our standards"
	
	npmCommand.set(listOf("run", "ng", "lint"))
	
	dependsOn(install, generateApi)
}

val dist by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Builds the final executable"
	
	npmCommand.set(listof("run", "ng", "build"))
	
	dependsOn(install, generateApi)
}

// We keep this task for backwards-compatibility: if we have an existing CI script, you won't have to modify it.
// You can also remove it.
val uiBuild by tasks.registering(NpmTask::class) {
	dependsOn(generateApi, lint, dist)
}
```

At the moment, Gradle cannot run `lint` and `dist` in parallel, that will come later in this article. However, it already knows that these two tasks are independent and can run in any order.

So far, the build hasn't fundamentally changed. Apart from the order which may be slightly different each time, the same steps happen every time. In fact, since we've introduced Gradle into the mix, the build is actually slower than just calling NPM directly. It's time to make Gradle work a little.

## Inputs and outputs

Gradle's goal is to create files from other files. Gradle knows it must do as little work as possible, so it uses a lot of information on the files to understand what needs to be done. For this to work, it must know which files are being touched.

To do so, we will use the `inputs` and `outputs` properties of tasks to declare which files are read and which files are written by each task.
```kotlin
val generateApi by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Generates the API stubs"

	npmCommand.set(listOf("run", "generate-api"))

	inputs.dir("api")
	inputs.file("generate-api.js")
	outputs.dir("src/api")
	dependsOn(install)
}

val lint by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Verifies that the source code corresponds to our standards"
	
	npmCommand.set(listOf("run", "ng", "lint"))
	
	inputs.dir("src")
	dependsOn(install, generateApi)
}

val dist by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Builds the final executable"
	
	npmCommand.set(listof("run", "ng", "build"))
	
	inputs.dir("src")
	outputs.dir("dist")
	dependsOn(install, generateApi)
}
```

Gradle now knows which tasks create which files. If we run the same command twice without changing any inputs, Gradle knows that the outputs are up to date, and does nothing. If you run `./gradlew :front:dist` twice, the second time should display:
```text
> Task :front:generateApi UP-TO-DATE
> Task :front:dist UP-TO-DATE
```

This means that Gradle has detected that the inputs for these tasks are exactly the same as the previous run, therefore nothing needs to be done. By default, Gradle bases itself on the modification dates of the files, but can go much further for some file types (for example, it can compare the binary API of JAR files).

!!! tip
    If Gradle doesn't mark the task as `UP-TO-DATE` when you run it a second time, run the same command with `--info`. This will print information about each task, what it did and why it did it. It will also print which inputs of a task changed.

At this point, you should be careful to specify _all_ inputs and outputs. If you forget an input, Gradle won't rerun when that input changes.
You can also declare inputs to regular variables, for example:
```kotlin hl_lines="8"
val dist by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Builds the final executable"

	npmCommand.set(listof("run", "ng", "build"))

	inputs.dir("src")
	inputs.property("nodeVersion", libs.versions.node)
	outputs.dir("dist")
	dependsOn(install, generateApi)
}
```

Sometimes, there may be more inputs that you initially think. For example, our `install` task becomes:
```kotlin
val install by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Installs NPM dependencies"

	npmCommand.set(listOf("install", "--target-arch=x64"))
	
	inputs.file("package.json")
	inputs.file("package-lock.json")
	outputs.dir("node_modules")
	dependsOn(tasks.npmSetup)
}
```
If we forget `package-lock.json`, we will not get the new dependencies installed when we change branches. Remember, with NPM, `package-lock.json` is the source of truth for dependencies.

Since Gradle knows the outputs, we can simplify our tasks further.
For example,
```kotlin
val app by tasks.registering(Zip::class) {
    dependsOn("uiBuild")
    from("dist")
}
```
can be replaced by
```kotlin
val app by tasks.registering(Zip::class) {
	from(uiBuild)
}
```
because Gradle already knows that `"dist"` is the output of `:uiBuild`. Gradle wires the dependency by itself.

## Continuous mode

Because Gradle now understands the file structure, we can now ask it to automatically rebuild modified files. Run any Gradle command with `--continuous` to get a continuously running build, which automatically rebuilds all files whenever anything changes.
```shell
./gradlew :front:dist --continuous
```

This is much more powerful than other 'watch' tasks in other tools. For example, if we change branches, Gradle will detect it, reload its configuration, download any new dependencies that may be missing, etc.

## Handling tasks with no outputs

You may have spotted an issue with our current setup: `:lint` is never considered `UP-TO-DATE`.

```kotlin
val lint by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Verifies that the source code corresponds to our standards"
	
	npmCommand.set(listOf("run", "ng", "lint"))
	
	inputs.dir("src")
	dependsOn(install, generateApi)
}
```

This is because it doesn't declare outputs. Because Gradle doesn't know what it builds, it prefers to stay safe and always rerun it. In effect, `:lint` doesn't build anything, it is just successful or failed. It may additionally print more information to the terminal.

The simplest way to fix this is to make the task create a marker file. This file will be empty and only present in the temporary build directory, but Gradle will be able to use it to track modification times. To do so, we use `doLast` to execute some code after the task has run all its actions, and declare that file as an output:
```kotlin
val lint by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Verifies that the source code corresponds to our standards"
	
	npmCommand.set(listOf("run", "ng", "lint"))
	
	val marker = project.layout.buildDirectory.file("lint-marker")
	
	inputs.dir("src")
	outputs.file(marker)
	dependsOn(install, generateApi)
	
	doLast {
		marker.get().writeText("")
	}
}
```

Now, we should see that `:lint` is marked `UP-TO-DATE` when it is run twice in a row.

Depending on your workflow, following the article up to here may already have yielded major performance improvements to your day-to-day life. On an enterprise project I worked on, this led to an 80% decrease in incremental build times because the UI was much more rarely modified than everything else, so rebuilding it all the time was purely wasted time.

Let's clean our work a little before continuing.

## Hooking into lifecycle tasks

To help users interact with a project they don't know about, Gradle provides a few lifecycle tasks. These tasks have no direct actions but they depend on tasks that do. Because their name is standard, users should be able to rely on their existence on any build they come across. Once we start creating our own tasks, we should follow these conventions to help other users that want to get to work on the codebase without having to understand the entire Gradle setup.

There are three such tasks:

`assemble`

:   The `assemble` task is meant to regroup all tasks related to building the "products" of this project. For example, it should compile the production binaries of an application, generate the final bundle of a documentation website, generate a Docker image for that site, etc.

`check`

:   The `check` task regroups all tasks that answer the question "Is my codebase correct?". This includes all possible verification actions: running automated tests, running a linter, verifying that there were no API breaking changes, etc.

:   This is the primary task a contributor should be using before sharing their code to someone else. If `./gradlew check` succeeds, users should be confident that their changes are correct. If you use Git hooks or other automation tools, this is the task you should run to verify your changes.

`build`

:   The `build` task is nothing but a shortcut to calling both `assemble` and `check`.

:   For this reason, build scripts shouldn't attach dependencies directly to the `build` task. Instead, build scripts should categorize their tasks between "it's building something" or "it's verifying something" and attaching them to the corresponding lifecycle task (respectively `assemble` and `check`).

:   Additionally, I've seen multiple times the usage of `./gradlew build -x check`. `-x` means "skip this task". Since `build` already means `assemble check`, this line is read as "Run the tasks `assemble` and `check`, but don't run the task `check`". Instead, just use `./gradlew assemble`.

!!! note
    The `:test` task isn't a standard lifecycle task. It is created by the Java plugin.

    By convention, it corresponds to running unit tests, which should be fast and deterministic. Other kinds of tests should have their own task (e.g. `integrationTest`) which is declared a dependency of `check`.

Let's apply these changes to our build script. We identify that:

- `:lint` is a verification task.
- `:dist` is a production task.

Therefore,
```kotlin
tasks.assemble {
	dependsOn(dist)
}

tasks.check {
	dependsOn(lint)
}
```

!!! info
    If Gradle complains that the tasks don't exist, add the built-in `base` plugin:
    ```kotlin
    plugins {
        base
    }
    ```
    In practice, it's likely that you're already using it through another plugin.

Now, users who don't know anything about the project can run `./gradlew check` and trust that their code has been verified.

There is another standard task that we don't support yet: `clean`.

## Cleaning up

Unlike `assemble`, `check` and `build`, which come from the `base` plugin, the `clean` task is built-in to Gradle itself. In fact, Gradle automatically generates cleanup tasks for any existing task. If you have a task called `foo` with some outputs declared, Gradle automatically declares the task `cleanFoo` which removes these outputs.

By default, calling `./gradlew clean` does not clean all tasks, however. I haven't seen an authoritative ruling on which tasks should be cleaned and which shouldn't. A common rule of thumb seems to be that outputs produced through network access should be kept, and outputs produced through local commands should be removed.

Following this rule, we will not clean the task `install` (which downloads files from NPM) but we will clean the task `dist` (which builds the final bundle).

To register a task to be cleaned, simply register it as a dependency of the `clean` task:
```kotlin
tasks.clean {
	dependsOn("cleanGenerateApi", "cleanDist")
}
```

The `lint` task's outputs are a marker file in the `build` directory, which is deleted by default by the `clean` task already.

Now that our tasks are up to the standard of modern Gradle builds, we can enable modern Gradle features for enhanced performance.

## Build caching

As mentioned in [Inputs and outputs](#inputs-and-outputs), Gradle is able to figure out which tasks need to rerun or not. It decides so by analyzing the file metadata and or contents, for example, the last modification date.

In everyday life, there are many cases where this isn't enough. For example, if we build the default branch, switch to another branch, do some changes, then come back to the default branch and rerun a build, we expect that this would be instant: nothing has changed since our last build in that branch. However, since we have run a build in another branch, our local `build` directory has been overwritten.

Gradle supports build caching: Gradle stores fingerprints of all supported tasks' executions in a global directory (under `~/.gradle/caches`). When running a supported task, if it isn't up to date, Gradle will hash the task inputs and look into the global directory. If it finds a matching entry, it uses it instead of running the task again.

Certain tasks don't make sense to cache. For example, tasks that zip some local files don't make sense to cache because caching them implies zipping them already, so the cost is the same. In general, you shouldn't try to make a task from a plugin cacheable: if it's not already, contact the plugin authors.

You can mark your own tasks as cacheable using `cacheIf`:
```kotlin hl_lines="10"
val dist by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Builds the final executable"

	npmCommand.set(listof("run", "ng", "build"))

	inputs.dir("src")
	inputs.property("nodeVersion", libs.versions.node)
	outputs.dir("dist")
	outputs.cacheIf { true }
	
	dependsOn(install, generateApi)
}
```

With this version, if we build, switch branches, build, come back, build again, we should see:
```text
> Task :front:dist FROM-CACHE
```

However, if we clone the same project in another directory and build, it will not reuse our previous build! This is because, by default, Gradle considers inputs to be absolute. Since the new repository has a different absolute path, Gradle thinks they are different inputs.

In this situation, two builds of the same code in two different directories will result in the same bundle. We can tell Gradle that declaring the path sensitivity:
```kotlin hl_lines="7"
val dist by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Builds the final executable"

	npmCommand.set(listof("run", "ng", "build"))

	inputs.dir("src").withPathSensitivity(RELATIVE)
	inputs.property("nodeVersion", libs.versions.node)
	outputs.dir("dist")
	outputs.cacheIf { true }
	
	dependsOn(install, generateApi)
}
```

With this change, Gradle can reuse builds across different clone paths. While users don't frequently have the same project cloned multiple times (if you do, you should use [`git worktree`](https://mskadu.medium.com/mastering-git-worktree-a-developers-guide-to-multiple-working-directories-c30f834f79a5)), you can see how this would be beneficial on your CI server of choice.

## Build caching (but more)

What if everyone on the team could share builds, and every build benefited everyone else? Gradle does support this through the Remote Build Cache.

In the simplest setup, the team hosts a Remote Build Cache server. To ensure it doesn't get polluted with broken builds, only CI pipelines running on the default branch are allowed to write to the cache. Everyone else can pull from the cache. On large projects, we rarely modify _everything_ in a single branch, so team members benefit from the remote cache for everything they haven't changed locally. Once a branch is merged, its contents are already in the cache before developers pull the branch.

Additionally, CI pipelines can reuse builds from each other, making small builds that change few files much faster. At OpenSavvy, we saw a ~40% decrease in CI pipeline length after enabling the Remote Build Cache.

The Remote Build Cache is nothing more than a simple HTTP file storage server. You can reimplement its protocol or host the official docker image. Gradle provides [Docker instructions](https://docs.gradle.com/develocity/build-cache-node/#docker) and [Kubernetes instructions](https://docs.gradle.com/develocity/build-cache-node/#kubernetes).

Configuring the build cache consists of adding a few lines in the `settings.gradle.kts` file. Since it doesn't change any configuration to tasks, it is out of scope of this article, but you can [learn more in the official documentation](https://docs.gradle.org/current/userguide/build_cache.html#sec:build_cache_configure_remote).

## Configuration caching

The next step in Gradle's evolution is to enable the Configuration Cache by default. The configuration phase consists of executing the different build scripts to understand what the user is trying to build.

With the configuration cache enabled, Gradle can entirely skip the configuration phase if you run the same command multiple times. To be the same command, the invocation needs to be identical: same arguments, same environment variables, etc.

However, the configuration cache has some [pre-requisites](https://docs.gradle.org/current/userguide/configuration_cache_requirements.html) in the way the tasks are declared. As a rule of thumb: keep it simple, and it will hopefully work. If it doesn't, the build will crash with a report that sometimes helps.

## The final version

While this is certainly more verbose than the initial version, I hope you will be able to understand each change and the benefits it brings.

```kotlin
plugins {
	alias(libs.plugins.node)
	id("maven-publish")
}

node {
	version.set(libs.versions.node)
	npmVersion.set(libs.versions.npm)
	download.set(true)
}

val install by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Installs dependencies from NPM"
	
	npmCommand.set(listOf("install", "--target-arch=x64"))

	inputs.file("package.json").withPathSensitivity(NAME_ONLY)
	inputs.file("package-lock.json").withPathSensitivity(NAME_ONLY)
	outputs.dir("node_modules")
	dependsOn(tasks.npmSetup)
}

val generateApi by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Generates the API stubs"

	npmCommand.set(listOf("run", "generate-api"))

	inputs.dir("api").withPathSensitivity(RELATIVE)
	inputs.file("generate-api.js").withPathSensitivity(NAME_ONLY)
	outputs.dir("src/api")
	outputs.cacheIf { true }
	dependsOn(install)
}

val lint by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Verifies that the source code corresponds to our standards"

	npmCommand.set(listOf("run", "ng", "lint"))

	val marker = project.layout.buildDirectory.file("lint-marker")

	inputs.dir("src").withPathSensitivity(RELATIVE)
	outputs.file(marker)
	outputs.cacheIf { true }
	dependsOn(install, generateApi)
	
	doLast {
		marker.get().writeText("")
	}
}

val dist by tasks.registering(NpmTask::class) {
	group = "front"
	description = "Builds the final executable"

	npmCommand.set(listof("run", "ng", "build"))

	inputs.dir("src").withPathSensitivity(RELATIVE)
	inputs.property("nodeVersion", libs.versions.node)
	outputs.dir("dist")
	outputs.cacheIf { true }
	dependsOn(install, generateApi)
}

// We keep this task for backwards-compatibility: if we have an existing CI script, you won't have to modify it.
// You can also remove it.
val uiBuild by tasks.registering(NpmTask::class) {
	dependsOn(generateApi, lint, dist)
}

val app by tasks.registering(Zip::class) {
	dependsOn(dist)
}

tasks.assemble {
	dependsOn(dist)
}

tasks.check {
	dependsOn(lint)
}

tasks.clean {
	dependsOn("cleanGenerateApi", "cleanDist")
}

publishing {
	publications {
		register("front", MavenPublication::class) {
			artifact(app) {
				artifactId = "ui"
			}
		}
	}
}
```

While having a single such `build.gradle.kts` is acceptable, projects shouldn't have multiple complex build scripts that are copies of each other. Once you start having build duplication, it is time to look into creating custom plugins—I promise, it's not as hard as it sounds! Though, that is a story for another time.
