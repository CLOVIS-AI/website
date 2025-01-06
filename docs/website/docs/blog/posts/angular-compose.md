---
date:
  created: 2024-12-30
slug: angular-compose
tags:
  - JS/TS
  - Kotlin
---

# Comparing Angular and Compose

Angular and Compose are both declarative UI frameworks and can both target the web ecosystem. Still, their approach is very different. Let's take a look at the similarities and the differences.

<!-- more -->

## Google × Google: a bit of history

### Angular

In 2010, a team at Google created [AngularJS](https://angularjs.org/), the first true client-side web framework. At the time, the king of the web was [jQuery](https://jquery.com/), which focused on simplifying DOM manipulation. In contrast, AngularJS brought an opinionated way to build _components_: units of UI that are _composed_ together to build an entire page, instead of writing each page completely on its own.

AngularJS is strict: each component is composed of a JavaScript module which defines behavior, two-way data-binding to an HTML template defining the view, and a CSS file describing styling information for that specific component.

In 2016, AngularJS was rebranded to [Angular](https://angular.dev/) as it was entirely rewritten to focuses on TypeScript, expanding on its features. Today, Angular continues growing and changing as other frameworks are born and die. Rich of a massive ecosystem, Angular is a true _framework_: everything is available out of the box, including dependency injection, modularization, internationalization, forms management and more.

### Compose

In contrast, Compose follows an entirely different approach. Compose is modular: it is a small library for state management, Compose is a compiler plugin to represent self-updating data, Compose is a UI framework. In fact, I don't know a single page I could link to that could appropriately describe what Compose is today (though I certainly can link to [other people complaining about that fact](https://jakewharton.com/a-jetpack-compose-by-any-other-name/)).

[Jetpack Compose](https://developer.android.com/compose) (not _Compose_) was first announced by Google in 2019, with the first release in 2021. Jetpack Compose is a state management library accompanied by a compiler plugin and a UI framework that entirely rewrites Android's UI stack into a new standard. Jetpack Compose is the child of the Android's team decade of legacy, and from React's component and state model. But the team knew that the state management library could live without Android's UI stack, and built it from the start so it could be used alone, potentially with another UI stack, similarly to React × React DOM × React Native.

[Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/) was later created by JetBrains by porting Android's UI toolkit to other platforms, including the desktop and web targets. Since Compose Multiplatform is a port of Android's UI toolkit, all apps created it with it look and feel like Android apps, which isn't always the goal on such different form factors. On the web, this is exacerbated by the usage of the canvas which feels even stranger for web developers—no DevTools, no right click, no text selection, no element inspector.

JetBrains also maintains (but seldom advertises) [Compose HTML](https://github.com/JetBrains/compose-multiplatform?tab=readme-ov-file#compose-html), similar in spirit to React DOM. Compose HTML is much closer to what a web developer expects of the web, from the availability of all DOM elements, the possibility to interoperate with JS/TS libraries, and the full power of the DevTools. If you want to dive deeper into the differences between Compose Multiplatform and Compose HTML, I recommend David Herman's [excellent article](https://bitspittle.dev/blog/2024/c4w).

### Drawing comparisons

I am primarily a web developer and rarely work on mobile or desktop applications. As Angular is a web-only framework, comparing it to Jetpack Compose or Compose for Desktop makes little sense, I will mostly compare Angular to Compose HTML. However, most of my points are about general code structure and concepts, which carry over to other Compose flavors—yes, even [Mosaic](https://github.com/JakeWharton/mosaic?tab=readme-ov-file#mosaic).

## UI Components

The core of a UI framework is the _component_: instead of treating each page as a single entity, we write each component in isolation and compose them together to build more complex interfaces.

### Strict × lenient

Angular has a very strict definition of a component:

- A component's visual appearance is defined by a template, written in an HTML file.
- A component's behavior is defined by a TypeScript class, and has a complex lifecycle.
- Optionally, a CSS file can augment the HTML's design.
- Optionally, UI tests can be specified directly in a specification file.

Instead, Compose is less opinionated on the ways code can be structured:

- A component's visual appearance is defined by a Kotlin function annotated by `@Composable`. There is no template language.
- A component's behavior is _preferably_ defined in an accompanying class, but this is purely convention and in no way a feature.
- Everything else can be done as you see fit.

It is certainly true that Angular's approach brings structure and consistency, but it also brings a massive amount of boilerplate. To demonstrate this, let's create a simple counter component, with the following requirements:

- The component stores a single integer, with a starting value decided by the calling component.
- The component displays the value to the user, with a "+" and "-" buttons allowing to update it, which notifies the parent component.

<div style="margin: auto; width: min-content">
<div style="display: flex; gap: 1em">
<button style="padding: 1em" onclick="c=document.getElementById('counterValue'); c.textContent = c.textContent - 1">-</button>
<p id="counterValue">0</p>
<button style="padding: 1em" onclick="c=document.getElementById('counterValue'); c.textContent = c.textContent - -1">+</button>
</div>
</div>

(go ahead, click on the buttons!)

#### Angular

First, we create the HTML template, in which we can use data- and event-binding:

```html

<div class="counter">
	<button (click)="dec()"> <!--(1)!-->
		-
	</button>

	<span>{{ value }}</span> <!--(2)!-->

	<button (click)="inc()">
		+
	</button>
</div>
```

1. Registering an event.
2. One-way data binding. Angular also possesses two-way data binding, but it is not shown in this example.

Second, we create the CSS file to hold the style:

```css
.counter {
	display: flex;
	gap: 1em;
}
```

Third, we create the component proper:

```typescript
@Component({
	selector: 'counter',
	templateUrl: './counter.component.html',
	styleUrls: ['./counter.component.css']
})
export class CounterComponent {

	@Input({required: true}) public value!: number;
	@Output() public changeEvent = new EventEmitter<number>();

	public inc(): void {
		this.changeEvent.emit(this.value + 1);
	}

	public dec(): void {
		this.changeEvent.emit(this.value - 1);
	}
}
```

Finally, we register the component in a module.

#### Compose

With Compose, a component is just a Kotlin function annotated with `@Composable`:

```kotlin
@Composable
fun Counter(
	val value: Int,
	val onChange: () -> Int,
) = Row {

	Button(onClick = { onChange(value - 1) }) {
		Text("-")
	}

	Text("$value")

	Button(onClick = { onChange(value + 1) }) {
		Text("+")
	}
}
```

That's it. These two components are equivalent.

We will dive further into each difference shown here. The main point I want to make for now is the difference in mental model and the difference in quantity of code. Since Angular components are more boilerplate to create and maintain, we tend to see users create very large components, often exceeding hundreds of lines of template code, whereas this is rare in the Compose world.

Smaller components are often easier to maintain than large components, and we observe each day that the slightest inconvenience towards creating a component often discourages developers from doing so. For example, when rendering a list of elements, Compose developers will create a component to display a single element, and another component for rendering the list, whereas Angular developers will often create a single component that does both.

Another constraint Angular places on components is that they cannot be overloaded: if two components are almost identical but have a slight difference (e.g. working with true objects vs working with IDs of objects), Angular will require two different components with different names (or a mega-component which somehow does both), whereas Compose will allow two components that differ by their signature.

This section was optimistic in Angular's favor. In the real world, such a counter element would probably be used in a form of some kind. I'll leave as an exercise to the reader to figure out what would need to be added for that component to be form-aware (or you can read [this tutorial](https://blog.angular-university.io/angular-custom-form-controls/)).

### Templates × code

In this section, we'll compare how UI elements are declared. Angular has its own templating language, while Compose simply uses Kotlin functions.

#### Representing UI in code

In the old days of imperative frameworks, UI was declared by creating elements and assigning them to each other. While this was conceptually simple, it was not possible to look at some code and immediately know what the UI would look like when running.
```javascript
function onButtonClick() {
	const div = document.createElement('div');
	div.appendChild(document.createTextNode("You clicked once!"));
	const button2 = document.createElement('button');
	button2.innerText = 'Click';
	div.appendChild(button2);
	view.removeChild(button);
	view.appendChild(div);
}

button.addEventListener('click', onButtonClick);
```
Often, these technologies were accompanied by XML or HTML files describing the initial state of the application (jQuery, JavaFX, .NET…); but it wouldn't be possible to know at a glance the different states an application could reach.

Templating and data-binding allowed developers to more easily understand the possible cases by applying logic directly within components. The UI code for the initial state and other states could finally be written in the same syntax.
```html
<button *ngIf="!clicked; else #clicked" (click)="clicked = true">
	Click here!
</button>

<div #clicked>
	You clicked once!
	<button>Click</button>
</div>
```
```javascript
@Component({
	selector: 'component',
	templateUrl: './component.component.html',
	styleUrls: ['./component.component.css']
})
export class Component {

	public clicked = false;
}
```

However, the downside was that all developers now needed to master _two_ languages: a programming language to build the logic, and a templating language which had to have its own conditionals, loops, event management, and possibly even functions. Since templating languages are meant to look as much as possible like the actual markup, nice syntax is often simply not available.

As framework authors improve their templating language, there are only two possible paths:

- Either the authors are very strict to keep logic out of templates (like Mustache), in which case more boilerplate is needed to work with them,
- Or authors attempt to make templates as easy to use as possible (like Angular), making them look more and more like a bastardized version of their accompanying programming language.

Instead of bringing yet another language along, Compose completely does away with templating. Kotlin is built with the aim of providing Domain Specific Language (DSL) capabilities: we can create what looks like a custom declarative language within Kotlin. Compared to a real new language, authors have less control over the syntax. However, the upside is massive: all tooling automatically supports all DSLs (debuggers, profilers, syntax highlighting, refactorings…), and we can use _any_ Kotlin feature or library within any DSL, making them way more powerful than virtually any other DSL.

By taking advantage of this feature, Compose becomes much more lightweight (as there is no need for a specific parser, for IDE plugins, or tooling any general) while still being easier to learn (because everything comes from Kotlin itself, there is no other language). We only need to look at the reference to know which components exist, we already know the syntax.

Using Compose, components take the form of functions. They are actually quite different from regular functions, but we use the same syntax for familiarity and convenience. To differentiate them, components are annotated with `@Composable`.
```kotlin
@Composable
fun Component() {
	var clicked by remember { mutableStateOf(false) }
	
	if (clicked) {
		Button(onClick = { clicked = true }) {
			Text("Click here!")
		}
	} else {
		Div {
			Text("You clicked once!")
			Button(onClick = { }) {
				Text("Click")
			}
		}
	}
}
```

In the rest of this section, we will dive deeper into the various consequences of this decision.

#### Flow control

Since we represent all states within our UI code (whether templating or Kotlin), we need a way to control which states are or aren't active. The most basic ways to do so are conditionals and loops.

##### If

Angular has `*ngIf`:
```html
<div *ngIf="someBoolean">
</div>
```
which can also represent else conditions:
```html
<div *ngIf="someBoolean; else #otherComponent">
</div>

<div #otherComponent>
</div>
```
`*ngIf` doesn't support `else if`.

Recently, Angular added the alternative (and more performant) `@if`:
```html
@if (someBoolean) {
	<div>
	</div>
}
@else if (otherBoolean) {
	<div>
	</div>
}
@else {
	<div>
	</div>
}
```

This syntax is already much more familiar to JavaScript developers, but it isn't exactly identical. For example, the brackets are mandatory, and only a subset of JavaScript is allowed within the parentheses.

Compose just uses Kotlin's `if`;
```kotlin
if (someBoolean) {
	Div {}
} else if (someOtherBoolean) {
	Div {}
} else {
	Div {}
}
```

Here, there are no limitations. All features of Kotlin are usable.

##### Switch

Angular has `*ngSwitch` to make multiple equality comparisons at once:
```html
<div [ngSwitch]="expression">
	<div *ngSwitchCase="value1">A</div>
	<div *ngSwitchCase="value2">B</div>
	<div *ngSwitchCase="value3">C</div>
	<div *ngSwitchDefault>D</div>
</div>
```

Angular also has a newer improved syntax `@switch`:
```html
@switch (expression) {
	@case ("value1") {
		<div>A</div>
	}
	@case ("value2") {
		<div>B</div>
	}
	@case ("value3") {
		<div>C</div>
	}
	@default {
		<div>D</div>
	}
}
```
Although this newer syntax is more performant, it isn't particularly similar to JavaScript's `switch`, so they still need to learn it. And since switches are not as commonly used as the other flow control options (I certainly don't use them more than once a week in Angular), remembering the syntax at all can be tough.

Again, Compose uses Kotlin's `when`:
```kotlin
when (expression) {
	"value1" -> Div { Text("A") }
	"value2" -> Div { Text("B") }
	"value3" -> Div { Text("C") }
	else -> Div { Text("D") }
}
```
Not only are there no further limitations, but `when` is much more versatile, so we use it more often:
```kotlin
when (value) {
	is Loading -> ProgressIndicator(value.progress)
	is Failure -> FailureMessage(value.failure)
	is Success if (value.isNewBestScore) -> Text("New best score!")
	is Success -> Text("Current score: ${value.score}")
}
```

##### Loops

Another fondamental flow control keyword is the `for` loop, which appears almost everywhere a component is repeated multiple times.

Angular supports this using `*ngFor`:
```html
<div *ngFor="let item of list">
	{{ item.score }}
</div>
```

Additional options can be used to access more information about the loop. For example, to get the current index:
```html
<div *ngFor="let item of list; index as i">
	{{ i }}. {{ item.title }}
</div>
```

These examples are not idiomatic, however. When such lists change, all elements that are different from the previous run must be re-rendered. If some elements swapped places but are otherwise identical, they must be rendered just like if they were completely new elements, and all their state will be lost. Instead, we should communicate to Angular how we want to track items, such that Angular can simply reorder the items without re-rendering them:
```html
<div *ngFor="let item of list; trackBy=trackingFunction">
	{{ item.title }}
</div>
```

Note that the function cannot be written inline, even if it is a simple field access, as is most often the case. Instead, it must be written in the accompanying TypeScript file. 

Or, using the newer alternative syntax:
```html
@for (item of list; track item.id) {
	{{ item.title }}
}
```
Once again, the alternative is more similar to JavaScript but still different enough that it has to be learned separately.

Compose uses Kotlin's `for` syntax:
```kotlin
for (item in list) key(item.id) {
	Text(item.title)
}
```
For the same reasons as Angular, Compose needs a way to track reorders, which is done through the `key` function. It isn't directly part of the `for` construct (since that comes from Kotlin itself) and is slightly easier to forget, but it has the benefit of not being coupled to `for`, so it can be used in other loops as well.

Indeed, Kotlin has many kinds of loops. As is standard in C descendants, in addition to `for`, there is `while` and `do…while`. However, since Kotlin is capable of DSLs, we can create additional loops that aren't part of the language, but behave similarly. The best example is probably `repeat` from the standard library:
```kotlin
repeat(10) {
	key(it) {
		Text("Element #$it")
	}
}
```

Flow control is a great example of the difference in approach between Angular and Compose: since Compose doesn't have a templating syntax, it supports out-of-the-box much more variety than Angular can, including the ability for users to easily create custom flow control options, without having to teach users another way to do things. Users can simply write code with the tools they already know, and it works.

#### Formatting and conversions

[//]: # (formatting pipes (| translate, etc))

#### Building complex components

Even if templating languages become near programming languages, they are still technically different languages and need their own library ecosystem or higher-level tooling

[//]: # (Directive)
[//]: # (Building DSLs on top of existing components)

### Inputs and outputs

### Content projection

## Modeling

[//]: # (Service)
[//]: # (Component class vs ViewModel)

## State management

The main challenge of writing UIs is the mutation of state: if the state was static, we wouldn't need a client-side framework at all and could just return a baked page from the backend directly. Thus, a client-side framework must be able to manage mutation of state and re-rendering following that change, quickly, with few resources.

[//]: # (Mono vs suspend)
[//]: # (Subject vs Flow; pipe vs function)
[//]: # (Signal vs State)

[//]: # (Change detection cycle)

[//]: # (Angular has absolutely everything ready-to-go / the ecosystem)
