---
date:
  created: 2025-02-10
slug: angular-compose
tags:
  - JS/TS
  - Kotlin
---

# Deep dive into Angular and Compose differences

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

##### Angular

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

##### Compose

With Compose, a component is just a Kotlin function annotated with `@Composable`:

```kotlin
@Composable
fun Counter(
	value: Int,
	onChange: (Int) -> Int,
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

Having a templating language is great for looking at the HTML file and visually understanding the structure of the component–but only if the template isn't cluttered. Although the TypeScript file exists to convert domain data into viewable data, such code is particularly inconvenient to write for pure formatting issues. For example, if you had a user profile and wanted to display the last name in Title Case, or if you wanted to replace enum elements by their text representation, or if you wanted to print a decimal number with a certain precision, or if you wanted to internationalize some text… Creating a new view object with mappers just for these simple transformations would be quite a pain.

Since these kinds of transformations are very common, Angular provides its own concept of functions: pipes. Pipes are TypeScript classes that implement a pure operation and are called directly within the template using a shorthand syntax:
```html
<h5>{{ userName | titleCase }}</h5>
```

This pipe could be implemented as follows:
```typescript
@Pipe({
	name: 'titleCase',
})
export class TitleCasePipe implements PipeTransform {
	transform(value: string): string {
		// https://stackoverflow.com/a/196991
		return value.replace(
			/\w\S*/g, 
			text => text.charAt(0).toUpperCase() + text.substring(1).toLowerCase()
		)
	}
}
```
This syntax isn't particularly complex to understand and I do think it is a great addition to Angular. However, it is one more concept with one more syntax that developers have to learn. While the overall idea is easy to memorize, the details are less so (which annotation should it use? What are its parameters? Which interface should it implement?).

In contrast, Compose again relies on Kotlin features. Since the UI is written using method calls, data transformations can simply be done using other functions.
```kotlin
H5 {
	Text(userName.toTitleCase())
}
```
```kotlin
fun String.toTitleCase() =
	split(" ").joinToString(" ") { word -> 
		word.replaceFirstChar { it.titlecase() } 
	}
```

#### Building complex components

Having access to all HTML elements as well as conditionals and loops is a great start, but these are relatively low-levels so building complex components may be verbose. Angular thus introduces a higher abstraction: directives.

Directives are TypeScript classes that are applied to a view and modify it in some way. [Attribute directives](https://angular.dev/guide/directives/attribute-directives) access the underlying DOM element and [structural directives](https://angular.dev/guide/directives/structural-directives) can decide the arity of a component (display it once, display it multiple times, or don't display it at all).

For example, we may want to create a table:
```html
<table>
	<tr>
		<th>First name</th>
		<th>Last name</th>
	</tr>
	@for (user of users; track user.id) {
		<tr>
			<td>{{ user.firstName }}</td>
			<td>{{ user.lastName }}</td>
		</tr>
	}
</table>
```
So far, we have only used a basic loop. But what if we wanted to allow users to sort the table by each column? We could add buttons to each header, implement our own sorting in the TypeScript file, etc, but that would be a lot of code that would clutter the template, and also would need to be duplicated each time we use a table, if we wanted consistency.

Instead, attribute directives allow us to invoke reusable features. Using the library [Angular Material](https://material.angular.io/), we can support sorted headers by editing our code to:
```html hl_lines="3 5 6"
<!-- Inspired by https://material.angular.io/components/sort/examples -->
<!-- Licensed under the MIT license -->
<table matSort (matSortChange)="sortData($event)">
	<tr>
		<th mat-sort-header="firstName">First name</th>
		<th mat-sort-header="lastName">Last name</th>
	</tr>
	@for (user of sortedUsers; track user.id) {
		<tr>
			<td>{{ user.firstName }}</td>
			<td>{{ user.lastName }}</td>
		</tr>
	}
</table>
```
and adding to the component:
```typescript
@Component({ /* … */ })
export class MyComponent {
	// …
	
	public sortData(sort: Sort): void {
		const data = this.users.slice();
		if (!sort.active || sort.direction === '') {
			this.sortedUsers = data;
			return;
		}
		
		this.sortedUsers = data.sort((a, b) => {
			const isAsc = sort.direction === 'asc';
			switch (sort.active) {
				case 'firstName':
					return compare(a.firstName, b.firstName, isAsc);
				case 'lastName':
					return compare(a.lastName, b.lastName, isAsc);
				default:
					return 0;
			}
		});
	}
}
```

As we can see, the template part is not particularly more complex than the initial version. The component code is significantly more complex, with the `sortData` method that is entirely boilerplate, but there exist other libraries with require much less code. However, the intrinsic problems remain: since we are "patching" existing HTML code, we must mimic its structure. Since headers are described far from cell contents, we must use some kind of magic value to identify which header affects which value.

Compose doesn't have a direct equivalent of directives. In fact, directives are a solution for a specific case in a larger problem space: creating higher abstractions to add features to existing components. Angular has a second solution for another specific case of that problem space: content projection. Compose expands content projection so it can fix both problems in a single feature, which we will discuss later in this article, in the [Content projection](#content-projection) section.

A fictitious way to implement sorted table headers in Compose (though I may release this as a library someday) may look like this:
```kotlin
SortedTable(users, trackBy = { it.id }) {
	column("First name", sortBy = { it.firstName }) {
		Text(it.firstName)
	}
	column("Last name", sortBy = { it.lastName }) {
		Text(it.lastName)
	}
}
```

For brevity's sake, I won't expand on structural directives. As we have already seen in the [Loops](#loops) section, we can use Kotlin to create custom loops, and in the same way we can create an equivalent to any other kind of structural directive.

### Inputs and outputs

Both Angular and Compose break the system into components which can be reused in multiple places. Components share data between each other through a paradigm called __unilateral data flow__.

Unilateral data flow dictates that:

- A parent component provides data to a child component,
- A child component provides events to a parent component.

That is, if some data is used in two components, it should be __hoisted__ to their common parent in the view. The data flow rule ensures separation of concerns between components, and makes leaf components easier to test, as they contain only the state they are directly responsible for.

##### Angular

To implement these concepts, Angular allows annotating the state of a component with the `@Input` and `@Output` annotations. An `@Input` is a value passed from a parent component, whereas `@Output` annotates an `EventEmitter`, a special object that wires events through components.

```html title="subscribe.component.html"
<button (click)="onClick()">
	Subscribe to {{ author }}'s posts
</button>
```

```css title="subscribe.component.css"
button {
	margin: 2em;
	background: transparent;
	border: darkred 1px solid;
}
```

```ts title="subscribe.component.ts" hl_lines="8 9"
@Component({
	selector: 'subscribe',
	templateUrl: './subscribe.component.html',
	styleUrls: ['./subscribe.component.css']
})
export class SubscribeComponent {

	@Input({ required: true }) author!: string;
	@Ouput() click = new EventEmitter<void>();
	
	public onClick(): void {
		this.click.emit();
	}
}
```

As we can see, even though Angular is written in TypeScript, and TypeScript has a concept of mandatory and optional fields, we must use a presence assertion (`author!`) and specify that the field is required in Angular annotations. I have hope that this situation will be improved upon in future Angular versions, as creating yet another way to represent the absence of a value in the JS ecosystem is superfluous.

Other than that, declaring an `@Input` in Angular is close to optimal syntax-wise: Angular views inputs as part of the component state (thus a class variable) with the special behavior that a parent can set them (thus marked by an annotation).

Outputs are slightly less optimal as they must be declared of the type `EventEmitter<T>`, instantiated with a constructor, and also annotated with `@Output`. Anywhere within the component, we can then trigger the event by calling its `.emit()` method, which the parent component will receive if they bound the output.

##### Compose

Compose takes a step back and asks: what are really inputs and outputs? 

- Inputs are simple data values passed from the parent component to the child component, easy enough. 
- Outputs are actions that are triggered in a specific situation. In programming, we usually represent actions by functions, so let's do the same here.

Since components are functions, they can naturally accept inputs as regular parameters. Outputs are represented by parameters of a lambda type, that the parent can thus provide and the child can call. With this simple rule, we have eliminated the need for yet another concept.

```kotlin hl_lines="3 4"
@Composable
fun SubscribeButton(
	author: String,
	onClick: () -> Unit,
) {
	Button({
		onClick { onClick() }
		
		style {
			margin(2.em)
			backgroundColor(Color.Transparent)
			border(Color.DarkRed, 1.px, BorderStyle.Solid)
		}
	}) {
		Text("Subscribe to $author's posts")
	}
}
```

Since outputs are just functions, we already know how to declare them and how to call them, there is no need for a special type.

An added benefit of using regular function types is that we can control their signature. Angular's `EventEmitter<T>` is equivalent to Compose's `(T) -> Unit`. Using Compose, we can create outputs with multiple parameters or with a return type (for example to confirm whether the action was successful) which is much more complex to do using Angular.

### Content projection

Content projection refers to the ability of a parent component to inject arbitrary UI into a child component. A typical use-case is layout components like `Column` that place multiple sections on the UI without rendering UI themselves.
Content projection is a powerful pattern allowing immense reuse of components.

In the Compose world, content projection is also referred to as "slotting": we imagine a component which contains "slots" that a parent component may fill in.

Content projection is Angular's second way of building more complex components from existing ones, the first being [structural directives](#building-complex-components) which we have mentioned already. Compose doesn't have structural directives and bases all abstractions on content projection; Compose users thus use content projection frequently, whereas it is a rarer feature in Angular applications.

##### Angular

In Angular, content projection is represented by the special `<ng-content>` tag in a child component:

```html title="child.component.html"
<article>
	<h1>{{title}}</h1>
	<div class="body">
		<ng-content></ng-content>
	</div>
	<div class="footer">
		<ng-content select="article-footer"></ng-content>
	</div>
</article>
```

```css title="child.component.css"
.footer {
	font-size: small;
}
```

```ts title="child.component.ts"
@Component({
	selector: 'child',
	templateUrl: './child.component.html',
})
export class ChildComponent {
	
	@Input() readonly title: string;
	
}
```

When a child contains the `<ng-content>` special tag, the parent may put code when invoking the child, which will be placed instead of the child's `<ng-content>` tag.

```html title="parent.component.html"
<child title="I know everything">
	<p>Hello world,</p>
	<p>Lorem ipsum dolor sit amet</p>
	<article-footer>Author: John Smith</article-footer>
</child>
```

```ts title="parent.component.ts"
@Component({
	selector: 'parent',
	templateUrl: './parent.component.html',
})
export class ParentComponent {
}
```

In this example, the `<child>` call contains two `<p>` and one `<article-footer>`. Since `<article-footer>` is mentioned in the child's second `<ng-content>`, it replaces that one, and both `<p>` replace the other one. The end result in the DOM is something like:
```html hl_lines="4 5 8"
<article>
	<h1>I know everything</h1>
	<div class="body">
		<p>Hello world,</p>
		<p>Lorem ipsum dolor sit amet</p>
	</div>
	<div class="footer">
		<article-footer>Author: John Smith</article-footer>
	</div>
</article>
```

Content projection is very powerful, but unfortunately clashes with a few other features of Angular, which makes it far less useful than it would be otherwise be. For example, a component using content projection cannot be used within a template-driven form, as the link between the parent component and inputs projected within `<ng-content>` is severed, so any such inputs are not considered part of the form.

As often with Angular, the syntax is a bit verbose and confusing for newcomers: it's the only part of the parent-child relationship that is specified within the template file and not within TypeScript, for example. Still, I personally think content projection is a great tool, and would benefit from being more transparent to other Angular features so they would work better together.

##### Compose

Unsurprisingly, Compose again takes a step back a rethinks what content projection is and how it should be implemented. After all, our goal is simply to inject UI code within another component. When using Compose, "UI code" is just a method annotated with `#!kotlin @Composable`, and our components are functions, so we can create higher-order components in the same way we create higher-order functions:

```kotlin title="Child.kt"
@Composable
fun Child(
	title: String,
	body: @Composable () -> Unit,
	footer: @Composable () -> Unit,
) = Article {
	H1 { Text(title) }
		
	Div({
		classes("body")
	}) {
		body()
	}
		
	Div({
		classes("footer")
		style {
			fontSize(FontSize.Small)
		}
	}) {
		footer()
	}
}
```

```kotlin title="Parent.kt"
@Composable
fun Parent() {
	Child(
		title = "I know everything",
		body = {
			P { Text("Hello world,") }
			P { Text("Lorem ipsum dolor sit amet") }
		},
		footer = {
			Text("Author: John Smith")
		},
	)
}
```

You may think this syntax is very similar to how Compose declares events, and you'd be right, but the presence of the `#!kotlin @Composable` annotation makes them unambiguous.

Since this relies on Kotlin language features, it composes much better with other concepts which also do so. In particular, inline functions are useful to declare layouts than are used transparently within different contexts, like forms. Additionally, Kotlin can mark parameters as mandatory or optional (content projection is always optional in Angular), or even accept varargs—though I haven't seen a use-case for vararg content projection yet.

##### Examples

Coupled with Kotlin's DSL capabilities, content projection allows extremely concise declarations that are hard—if not impossible—to replicate in Angular.

This first one is a DSL for declaring an infinitely-scrolling feed that loads and unloads elements based on the viewport:
```kotlin
LazyColumn {
	item {
		Text("This is always the first element of the list")
	}
	
	items(200, key = { it }) {
		Text("Element $it")
	}
	
	items({ loadPageFromServer(it) }) {
		Text("Loaded the element $it")
	}
}
```

!!! tip "Angular equivalent?"
    If I wanted to create an Angular API similar to this, I would probably use structural directives and not content projection. I don't believe Angular provides many tools to conditionally project children, or to project them multiple times, so content projection probably wouldn't work well here.

The second one is a concise DSL for declaring tables:
```kotlin
Table(yourData, key = { it.id }) {
	Column("Date") {
		sortedBy { it.timestamp }
		render {
			Text(it.timestamp.toString())
		}
	}
	
	Column("Event") {
		render {
			Text(it.description)
		}
	}
	
	Column("Author") {
		sortedBy { it.authorName }
		render {
			Text(it.authorName)
		}
	}
}
```

## State management

The main challenge of writing UI is the mutation of state. Immutable data is much easier to work with in almost all regards, but if the state was immutable, we wouldn't need a client-side framework and could just return a baked page from the backend directly. Thus, a client-side framework must be able to manage mutation of state and re-rendering following that change, quickly, with few resources.

### Change detection

The first difficulty with dealing with change is knowing when it happened. Programming languages don't usually provide first-party ways to be notified when values change, or when they do it is intended for debuggers where performance is less a concern. Somehow, the framework must know when to re-render parts of the UI. Rendering as little as possible of the UI is crucial for performance, especially on low-end devices. Minimizing rendering is also important for accessibility.

More specifically, we are not interested in _all_ state changes of the application. We are only interested in changes that:

- apply to state displayed in the UI, or
- by changing value, impacts other state which itself validates one of these two points.

Over the years, many strategies for change detection have existed in many frameworks. Today, Angular has two main change detection detectors, the second being closer to Compose's method.

##### Angular: The bruteforce way

Traditionally, Angular state is stored as class fields in components. The obvious benefit of this approach is that everyone knows how to use class fields, and thus everyone knows how to manage Angular state. The downside is JavaScript doesn't provide a way to know when an arbitrary expression has changed. [`Proxy`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) exists but isn't used to my knowledge, probably because it is impractical in user-defined fields.

Angular adopts a bruteforce approach to change detection, powered by [Zone.js](https://www.npmjs.com/package/zone.js). Angular subscribes to a large amount of browser events: button clicks, inputs, timeouts, fetch requests finishing, etc, and uses heuristics to guess which component they could emanate from. Then, Angular recomptes all expressions bound in templates, sees if they changed, and if so starts re-rendering.

While this approach is conceptually simple for the developer, it is quite complex internally and also quite expensive to run. Angular frequently over-renders or over-computes. It is therefore crucial to make expressions bound in the view as simple as possible, ideally just reading variables.

Lastly, the most fine-grained change detection done through this method is based at the level of the component. Large components which rarely change except in very subtle ways must be recomputed entirely. For these reasons, the Angular team has been working on other change detection algorithms.

##### Angular: The async pipe

The `async` pipe allows binding an observable value into the view, in a way Angular can understand. An observable value is a special type that allows readers to be notified when the value changes. We will discuss these later in this article.

```html title="counter.component.html"
<div>{{ count$ | async }} elements</div>
```

```typescript title="counter.component.ts"

@Component({
	selector: 'counter',
	templateUrl: './counter.component.html',
})
export class CounterComponent implements OnInit {

	public count$!: Observable<number>;

	constructor(
		private counterService: CounterService,
	) {
	}
	
	ngOnInit(): void {
		this.count$ = this.counterService.countAsync();
	}
}
```

Since Angular subscribes to the `count$` observable itself, it knows when it changes. It can then run change detection and re-rending specifically for that value. Additionally, Angular can automatically unsubscribe from the observable when the component is destroyed.

The direct conversion of this component in bruteforce style would look something like:

```html title="counter.component.html"
<div>{{ count ?? '' }} elements</div>
```

```typescript title="counter.component.ts"

@Component({
	selector: 'counter',
	templateUrl: './counter.component.html',
})
export class CounterComponent implements OnInit, OnDestroy {

	public count: number = undefined;
	private subscription: Subscription | undefined = undefined;

	constructor(
		private counterService: CounterService,
	) {
	}
	
	ngOnInit(): void {
		this.subscription = this.counterService.countAsync()
			.subscribe(count => this.count = count);
	}
	
	ngOnDestroy(): void {
		this.subscription?.unsubscribe();
	}
}
```

This variant is already quite a bit longer, and although it seems to behave identically, it is actually quite a bit more expensive at runtime. Because Angular cannot know exactly when and which state changes, it has to guess, which may result in performance loss as Angular is forced to do useless work.

##### Angular: Signals

Recently, Angular has introduced the concept of signals. Signals will be described later in this article. For now, we can describe them as value wrappers: we can replace the wrapped value, which will notify Angular.

Components must opt in with [`signals: true`](https://github.com/angular/angular/discussions/49682), which entirely disables the bruteforce change detection algorithm for that component. Once this is done, component state must be represented using signals instead of class fields: changing a class field will not trigger a re-rendering anymore.

A downside of signals is that they cannot represent internally-mutable data. A signal may itself mutate, but any nested objects may not, as their mutations won't be tracked. Developers must thus work with deeply-immutable data structures of which only the roots are stored in signals.

Even though this is a big departure from previous Angular mechanisms, it is internally much simpler and efficient, because Angular can understand the application in much greater detail, making it possible to optimize the framework much further. Enforcing deeply-immutable data also removes a large category of hard-to-track bugs in developer code.

Note that you may use signals in Angular components without `signals: true`, for example to profit from the greatly simplified lifecycle management, but you will access any of the performance benefits then.

##### Compose

On the web, it is usually frowned upon to re-render every frame. Instead, developers should rely on CSS animations or the canvas to render per-frame content, as the DOM is simply not made for these situations. UI frameworks therefore only need to be faster than the DOM, so some loss of performance is acceptable.

Compose was initially created by the Android team as a complete replacement for the Android rendering pipeline. Compose isn't meant to be a framework declaring the structure to another renderer, it is meant to be the renderer itself. In the Android world, animations are made with regular Compose components that re-render every frame. Nowadays, this means some components may need to render as many as 120 times per second.

The bruteforce approach followed by default by Angular is therefore not an option. Instead, Compose follows what is essentially the same approach as Angular's `signals: true` components, where values are wrapped in a special mutator type that the framework understands deeply.

While this approach is more taxing on developers, as they need to learn how to manage state, it is so much more performant that it is the dominant approach followed by other frameworks, such as React.

<br>

Now that we understand the overall approach to change detection, we can discuss the ways both frameworks handle state management. In my eyes, there are two main reasons why state changes: when a local event happens (the user interacts with the app) and when a remote event happens (we fetch some data, another user edits data). The former is immediate (we can always access the value at the current time), and the latter is delayed.

Angular and Compose don't necessarily differentiate these two cases exactly in this way, but they both feature idiomatic ways to represent both of these situations.

### Immediate state

Immediate state is represented by framework-defined wrappers types that contain user-defined immutable data. Angular calls these wrapper types "signals" whereas Compose calls them "state".

#### Imperative state management

##### Wrapping and updating a value

Since these types wrap an existing value, frameworks must provide an easy way to read and write them:

```ts title="Angular"
const count = signal(0);

console.log('Reading the value', count());

count.set(5)
```

```kotlin title="Compose"
var count by mutableStateOf(0)

println("Reading the value $count")

count = 5
```

Both frameworks profit from their host language's type inference, so the wrapped type doesn't need to be explicitly written.
Compose's state declaration functions are more verbose (which we will see is a trend with Compose state management), but use-site uses plain old Kotlin syntax for variable management, thanks to Kotlin's `by` keyword which transparently delegates the getter and setter—whereas, Angular developers must remember to use `()` to read the value, and call `set()` when writing it. This is currently causing a minor pushback in the Angular world, as developers have been taught for years to never call functions within a template, but signal accessors are specifically designed to be used as such.

In both frameworks, wrapped types can be used within components, but also anywhere else. This is a major distinction compared to React, where state can only be managed within UI code, which creates the need for large meta frameworks for handling global state, like Redux.

##### State lifetime

Conceptually, there are two kinds of state lifetimes possible in a component: either state exists for the entire existence of that component, or state exists only during a single rendering pass. There, Angular and Compose have a different default. 

Since Angular components are class-based, it is intuitive that class fields would have the lifetime of the class, thus signals declared in a class exist for the lifetime of the component described by that class, and signals declared within functions are local to these functions. Angular doesn't have a single function which contains an entire rendering pass, instead splitting it into `ngOnChanges`, `ngAfterContentChecked` and many others.

Since Compose components are function-based, and a function call corresponds to a rendering pass, local variables intuitively only exist for a single rendering pass. To declare that a variable is kept between rendering passes, we use the `remember {}` helper. We can remember any kind of value, not just state. Because state is much more useful when it is stored for the entire lifetime of the component, it is very frequent to see state being remembered:
```kotlin
@Composable
fun Foo() {
	var count by remember { mutableStateOf(0) }
    
	// …
}
```
We can think of `remember {}` as a way to bind a value to the component lifetime. It thus can only be used within components.

One common pattern this allows us to do is to declare a class that contains internal state that can apparently mutate, and manage it as a single value. I personally find this pattern particularly useful when creating forms:
```kotlin
// Domain representation, immutable
data class User(
	val name: String,
	val age: Int,
)

// View representation, appears to be mutable
class MutableUser(user: User) {
	var name by mutableStateOf(user.name)
	var age by mutableStateOf(user.age)
    
	fun toDomain() = User(name, age) 
}

// UI code
@Composable
fun UserForm(user: User, save: (User) -> Unit) {
	val view = remember { MutableUser(user) }
    
	TextField("Name", view.name, onInput = { user.name = it })
	NumberField("Age", view.age, onInput = { user.age = it })
    
	SubmitButton(onClick = { save(view.toDomain()) })
}
```
Here, we declare the state in a class, an instance of which is remembered. The same pattern could be implemented in Angular, but rarely is. I suspect the added syntax when dealing with signals may be the cause of the impopularity of this pattern. It's interesting to note that such a grouping cannot be implemented with React.

##### More complex change detection

Our goal is to change state as little as possible, as each state change may cause a new rendering pass. However, state often changes in ways which are not relevant to the user. For example, if we query a backend service for a list of entities, and we decide to refresh the page after an operation (just in case data has changed), and the exact same data is returned by the server, we will actually get a new list instance which contains brand-new objects, which just happen to be exactly the same as the ones we already have. In this situation, we don't want to render the screen again, since user-relevant data has not changed.

JavaScript has many ways to represent equality of objects. By default, Angular uses [`Object.is()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is), which considers two arrays to always be different, no matter their content. The algorithm used can be overridden when declaring a signal:
```typescript
const entityList = signal([] as Entity[], { equal: customEqualityFunction })
```

Similarly, Compose allows overriding the comparison strategy:
```kotlin
val entityList by mutableStateOf(emptyList<Entity>(), customEqualPolicy())
```
In practice, this is rarely used, because the default behavior of Compose is to use Kotlin's `==` operator, which is already defined as structural equality for `data class` and collections, so we get the behavior we expect when querying remote data. Since each type can customize this behavior by overriding `equals`, the default behavior is often exactly what we want for all types.

#### Declarative state management

So far, we have managed state imperatively, by calling `.set()` or equivalent ourselves. However, the real strengths of wrapped state is the possibility to declare the relationship between multiple states and have the framework deal with updating them.

##### Creating dependent values

To start with, we can declare a state that is computed by combining the value of multiple other states. When any of these states change, the final state is updated. Just like with imperative states, if the result is identical to a previous value, nothing happens.

```typescript title="Angular"
const value = signal(0);
const isEven = computed(() => { value() % 2 === 0 });
```

```kotlin title="Compose"
var value by mutableStateOf(0)
val isEven by derivedStateOf { value % 2 === 0 }
```

Both frameworks automatically keep track of which states are read, and automatically subscribe and unsubscribe to them.

The main benefit of this approach is that it "buffers" changes. For example, if the view is very different for even and odd numbers, and the number changes in non-incremental ways (such that it often remains even or odd), reading the value directly within the view would trigger a rendering each time the value changes. However, reading the `isEven` computed state will ensure the view only attempts to render if the evenness actually changed.

The main downside is that this management of dependencies can be a bit more expensive, so it is usually discouraged if the resulting state changes with the same frequency as the initial states. Another downside is that it stores an intermediary value, which is fine most of the time, but may cause memory contention if too many are used or when they store large objects.

For values which change very frequently, Compose has another trick up its sleeve: since each component is a function that corresponds to a single rendering pass, we can declare local variables that read states:
```kotlin
@Composable
fun Foo() {
	var userName by remember { mutableStateOf("") }
	val email = "${userName}@company.com"
    
	// …
}
```
Assuming the `userName` is the primary source of recomposition in this component, storing another state for the `email` is pointless, as it will change each time anyway. The equivalent Angular pattern is creating a TypeScript getter which reads signals, and calling it in the view.

##### Creating dependent effects

In programming, we often say there are two different root concepts: data/variables and effects/functions. Just like we can declare some data that depends on states, we can declare effects that should be run each time some state changes.

Note however that this is considered fairly low-level and shouldn't be used frequently within end-user code. Valid use-cases include notifying a non-framework-aware entity that state has changed, registering and unregistering event listeners, logging data, etc. Such uses should be encapsulated within utility functions and isolated from business code.

In this example, we will ensure some state is synced with local storage, so it isn't lost even if the user closes the page: 
```typescript title="Angular"
const value = signal(window.localStorage.getItem("value") ?? 0);
effect(() => { window.localStorage.setItem("value", value()) });
```

```kotlin title="Compose"
@Composable
fun ValueEditor() {
	var value by remember { mutableStateOf(window.localStorage["value"] ?: 0) }
	
	SideEffect(value) {
		window.localStorage["value"] = value
	}
}
```

Unlike previous state management solutions, which were very similar between Angular and Compose, this one is quite different. First, Compose requires that effects be tied to a specific component, whereas Angular allows them to be used anywhere. Second, Compose's effects do not automatically keep track of dependencies, so we have to specify which values we depend on when declaring the effect.

Lastly, Compose has different kinds of effects. Both TypeScript and Kotlin make a distinction between regular functions and asynchronous/concurrent functions (TypeScript: `await`, Kotlin: `suspend`) which cannot be called within regular functions. Angular's `effect` cannot be used with an asynchronous operation, which is the exact equivalent to Compose's `SideEffect`. In both cases, if the operation is too slow, rendering will slow down. However, Compose also has `LaunchedEffect` and `DisposableEffect`.

`LaunchedEffect` is written similarly, but its action is marked `suspend`. The action is started in exactly the same situations as a regular `SideEffect`, but rendering continues without waiting for the action to finish. The action can thus run for multiple rendering cycles. However, if the dependencies change, the currently-running action is cancelled and a new one is started. If the component is removed from the page, the action is cancelled as well. Using this, we can create extremely simple debouncing:
```kotlin
@Composable
fun Foo() {
	var value by remember { mutableStateOf(window.localStorage["value"] ?: 0) }

	LaunchedEffect(value) {
		delay(300) //(1)!
		window.localStorage["value"] = value
	}
}
```

1. Delay writing to `LocalStorage` until a few milliseconds later. If the user is typing, this ensures we don't create lag by storing each intermediary values pointlessly.

Lastly, `DisposableEffect` is identical to `LaunchedEffect` but allows declaring a clean-up operation to do when the effect is restarted or when the component is removed.
```kotlin
@Composable
fun Foo() {
	DisposableEffect(Unit) {
		println("The component is being added to the view")
        
		onDispose {
			println("The component is being removed from the view")
		}
	}
}
```
Angular has a similar feature, but it cannot use asynchronous code:
```typescript
effect((onCleanup) => {
	console.log('The component is being added to the view');
	
	onCleanup(() => {
		console.log('The component is being removed from the view');
	});
});
```

##### More complex data structures

Angular's recent signal implementation has the advantage of being very easy to use and learn, because its API surface is very small: `signal()`, `computed()` and `effect()` replace almost the entirety of the previous system, which was based on dozens of lifecycle hooks with complex ordering. 

So far, while Compose provides more options, they can be emulated on top of Angular's primitives relatively easily. For example, `LaunchedEffect` can be implemented using a regular Angular effect by starting a `Promise` in the body and using an `AbortController` in the cleanup function—which is more or less how `LaunchedEffect` is implemented in Compose anyway. This is limited by JavaScript not having structured concurrency, but Angular's state system is still quite streamlined and terse. I find this quite welcome, as Angular didn't previously strike me as a concise technology code-wise, as we have seen in the UI declaration sections.

There is still a feature I miss from Compose state management, however. Angular signals always wrap an entire value, and cannot understand changes to only a part of the wrapped value. For example, if we want to add an element to a wrapped list, Angular forces us to overwrite the entire list to update the signal:
```typescript title="Angular"
const users = signal([new User("foo"), new User("bar")]);

users.update(previous => [...previous, new User("baz")]);
```
Compose instead has dedicated state types for collections:
```kotlin
val users = mutableStateListOf(User("foo"), User("bar"))

users.add(User("baz"))
```
Here, Compose gives us a `MutableList<User>` that we can use exactly as any other Kotlin list, but that notifies the framework when any mutation is executed. Compose provides the same feature for maps, and we can create our custom types with interior mutability detection, though this is considered an advanced topic.

### Delayed state

Delayed state cannot be accessed whenever we want. Instead of operating on the state itself, we describe the computations we want to perform, and the runtime will perform them later once the state is known.

TypeScript and Kotlin both have first-party support for asynchronous operations, through `async` in TS and `suspend` in Kotlin. These are different in a number of ways that won't be elaborated upon in this article, because `async` is rarely used in Angular applications. Instead, Angular developers work with RxJS.

[RxJS](https://rxjs.dev/) is an asynchronous stream library, specialized in representing events through time and operations on them, for example merging two streams, filtering events, etc.
```typescript
http.get<Updates[]>('/api/updates')
	.pipe(
		concatAll(),
		filter(update => update.id % 2 === 0),
		map(update => update.data),
	)
	.subscribe(updateData => {
		console.log('Received event', updateData);
	});
```
RxJS was the only state management solution in Angular until the recent introduction of Signals. This lead RxJS to be used in many more situations than it was designed for, giving it a reputation of being clunky. RxJS also isn't particularly friendly to beginners, but so is the entire concept of stream-based design. The primary API element of RxJS is the `Observable`, which is used to describe complex operations on data we do not necessarily have access to yet.

Kotlin represents delayed state through the first-party [KotlinX.Coroutines library](https://kotlinlang.org/docs/coroutines-overview.html), especially the `Flow` interface, which is very similar to RxJS' `Observable`.
```kotlin
client.get<List<Updates>>("/api/updates")
	.body()
	.asFlow()
	.filter { it.id % 2 == 0 }
	.map { it.data }
	.collect {
		println("Received event $it")
	}
```

Writing a full comparison between RxJS and KotlinX.Coroutines is out of scope for this article—they are overall quite similar anyway. I would say the two main differences are the way errors are handled, and structured concurrency.

Using RxJS, errors flow through the pipe the same way data does. Most operators can accept a second optional lambda to declare how the operator behaves in the presence of errors. If no such lambda is declared, errors are swallowed silently on subscription. Coroutines are designed such that exceptions behave transparently, as if they were regular functions. An exception will thus bubble up and be thrown during subscription, forcing the calling code to handle it.

The flagship feature of Coroutines is structured concurrency: a coroutine is executed in the context of a `Job`, which describes its lifecycle. At any point, a user can `cancel` the `Job`, killing the coroutine and all its children. In UI interfaces, cancelling unneeded work is very important to avoid bugs when users change pages or perform actions while other actions are ongoing. RxJS also supports cascading cancellation, but that is specifically handled by the caller instead of being wired in when the operation is declared. While the difference seems minor, in practice, it is extremely common to find examples of using RxJS where no cancellation is done at all, including at conferences. This is much rare in the Compose world because cancelling requests when the context change is a built-in feature and extremely important to multiple markets, including Android.

Lastly, Coroutines are based on the concept of a `suspend` function as a first-party citizen. Flows of a single element are widely discouraged, which greatly simplifies pipelines as merging two flows is rarely needed. RxJS doesn't have such a concept, forcing developers to understand complex flow merging strategies more intimately.

## Honorable mention

Angular and Compose are full frameworks, and I could have compared them through many other lenses. Before closing this article, here are a few of them:

- Angular is based on TypeScript, and Compose is based on Kotlin. I have already written an article [comparing the two languages](../2024/ts-kjs-types.md).
- Angular has an opinionated way of representing business logic in `Service` classes. Compose is a UI framework and doesn't concern itself with such aspects.
- Angular has an opinionated way of handling dependency injection. Compose doesn't, leaving the user free to use whichever technique they want.
- Angular splits components into a template and a TypeScript class. Compose only really requires the composable function, which takes the role of the template. However, for components which have complex logic (data loading, validation…) it is recommended to store data in an accompanying class, for example through [Compose ViewModel](https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-viewmodel.html).
- Build tooling. Angular, especially through standalone components and built-in navigation, is particularly efficient at breaking a complex app into smaller bundles, which massively decrease initial load times (though they also increase navigation times). At the time of writing, Compose doesn't have such a feature, though a few precursor pieces have been built.

## The ecosystem

Throughout this article, I have mostly been showcasing situations in which Compose is better—at least in my opinion—than Angular. After all, Compose is more recent, and a much more streamlined library with a clear goal, instead of being an all-encompassing framework.

Comparing Compose and Angular in such a way may be a bit unfair, however. Angular is somewhat analogous to the Java of the web ecosystem. Angular was there before, it will be there after. Angular is often late to new features, but it gets there, unlike many frameworks who effectively die after a few years.

In a way, being late is Angular's model. By showing up after its competitors, Angular can implement only the good ideas, and slowly get better over time. This mechanism is very similar to modern Java, which always looks at Kotlin, Scala and other modern languages for inspiration, focusing on incremental improvements in the meantime.

An enormous benefit of this strategy is the _ecosystem_. The amount of libraries available for Angular is absolutely massive, and anything you could ever want to do probably already has a binding. For long-running complex projects where speed of development doesn't matter as much as having everything available immediately, Angular is probably still a good choice, and will likely remain one in the foreseeable future.

In the end, the most important difference is whether you like working with a massive sprawling framework that has an opinionated way of doing _everything_ built-in, or if you prefer to take the best parts of multiple technologies and compose them yourself together.

This is the one aspect where I have less trust in Compose. Compose's complex history, being developed in part by the Android team, which heads a massively complex ecosystem with extremely hard barriers to entry, and JetBrains, which often mimics Android solutions because their main use-case is porting Android applications to other platforms rather than building new multiplatform experiences. I'm slightly worried Compose will grow more and more complex until it is nothing more than a port of Android, instead of being the promised platform-independent way of describing UI applications.

As always with these kinds of articles, I'm very curious what the future holds, and I hope you found these comparison points useful.
