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

[//]: # (*ngIf)

[//]: # (*ngFor)
[//]: # (formatting pipes (| translate, etc))
[//]: # (new syntax for loops and conditions?)
[//]: # (Building DSLs on top of existing components)

### Inputs and outputs

### Content projection

## State management

The main challenge of writing UIs is the mutation of state: if the state was static, we wouldn't need a client-side framework at all and could just return a baked page from the backend directly. Thus, a client-side framework must be able to manage mutation of state and re-rendering following that change, quickly, with few resources.

[//]: # (Service)

[//]: # (Component class vs ViewModel)

[//]: # (Subject vs Flow; pipe vs function)

[//]: # (Change detection cycle)

[//]: # (Angular has absolutely everything ready-to-go)
