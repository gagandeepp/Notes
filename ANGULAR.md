# Angular Framework Guide

## Overview

Angular is a web framework that empowers developers to build fast, reliable applications that users love.

---

## Components

Components are the main building blocks of Angular applications. Each component represents a part of a larger web page. Organizing an application into components helps provide structure to your project, clearly separating code into specific parts that are easy to maintain and grow over time.

### Basic Component

```typescript
// user-profile.ts
@Component({
  selector: 'user-profile',
  template: `
    <h1>User profile</h1>
    <p>This is the user profile page</p>
  `,
  styles: `
    h1 {
      font-size: 3em;
    }
  `,
})
export class UserProfile {
  /* Your component code goes here */
}
```

### Component with External Files

```typescript
// user-profile.ts
@Component({
  selector: 'user-profile',
  templateUrl: 'user-profile.html',
  styleUrl: 'user-profile.css',
})
export class UserProfile {
  // Component behavior is defined in here
}
```

### Importing and Using a Component

To import and use a component, you need to:

1. In your component's TypeScript file, add an import statement for the component you want to use.
2. In your `@Component` decorator, add an entry to the `imports` array for the component you want to use.
3. In your component's template, add an element that matches the selector of the component you want to use.

```typescript
// user-profile.ts
import { ProfilePhoto } from 'profile-photo.ts';

@Component({
  selector: 'user-profile',
  imports: [ProfilePhoto],
  template: `
    <h1>User profile</h1>
    <profile-photo />
    <p>This is the user profile page</p>
  `,
})
export class UserProfile {
  // Component behavior is defined in here
}
```

> **Note:** By default, Angular components are standalone, meaning you can directly add them to the `imports` array of other components. Components created with an earlier version of Angular may instead specify `standalone: false` in their `@Component` decorator. For these components, you import the `NgModule` in which the component is defined.
>
> **Important:** In Angular versions before 19.0.0, the `standalone` option defaults to `false`.

---

## Signals & Reactivity

In Angular, you use **signals** to create and manage state. A signal is a lightweight wrapper around a value.

### Creating and Using Signals

Use the `signal` function to create a signal for holding local state:

```typescript
import { signal } from '@angular/core';

// Create a signal with the `signal` function.
const firstName = signal('Morgan');

// Read a signal value by calling it — signals are functions.
console.log(firstName()); // Morgan

// Change the value of this signal by calling its `set` method with a new value.
firstName.set('Jaime');

// You can also use the `update` method to change the value
// based on the previous value.
firstName.update((name) => name.toUpperCase());
```

Angular tracks where signals are read and when they're updated. The framework uses this information to do additional work, such as updating the DOM with new state. This ability to respond to changing signal values over time is known as **reactivity**.

### Computed Signals

A `computed` is a signal that produces its value based on other signals.

```typescript
import { signal, computed } from '@angular/core';

const firstName = signal('Morgan');
const firstNameCapitalized = computed(() => firstName().toUpperCase());

console.log(firstNameCapitalized()); // MORGAN
```

A computed signal is **read-only**; it does not have a `set` or an `update` method. Instead, the value of the computed signal automatically changes when any of the signals it reads change:

```typescript
import { signal, computed } from '@angular/core';

const firstName = signal('Morgan');
const firstNameCapitalized = computed(() => firstName().toUpperCase());

console.log(firstNameCapitalized()); // MORGAN

firstName.set('Jaime');
console.log(firstNameCapitalized()); // JAIME
```

---

## Templates

Component templates aren't just static HTML — they can use data from your component class and set up handlers for user interaction.

### Interpolation

```typescript
@Component({
  selector: 'user-profile',
  template: `<h1>Profile for {{ userName() }}</h1>`,
})
export class UserProfile {
  userName = signal('pro_programmer_123');
}
```

### Property Binding

Angular supports binding dynamic values into DOM properties with square brackets:

```typescript
@Component({
  // Set the `disabled` property of the button based on the value of `isValidUserId`.
  template: `<button [disabled]="!isValidUserId()">Save changes</button>`,
})
export class UserProfile {
  isValidUserId = signal(false);
}
```

You can also bind to HTML attributes by prefixing the attribute name with `attr.`:

```html
<!-- Bind the `role` attribute on the `<ul>` element to value of `listRole`. -->
<ul [attr.role]="listRole()"></ul>
```

Angular automatically updates DOM properties and attributes when the bound value changes.

### Event Binding

Angular lets you add event listeners to an element in your template with parentheses:

```typescript
@Component({
  // Add a 'click' event handler that calls the `cancelSubscription` method.
  template: `<button (click)="cancelSubscription($event)">Cancel subscription</button>`,
})
export class UserProfile {
  cancelSubscription(event: Event) {
    /* Your event handling code goes here. */
  }
}
```

### Conditional Rendering

```html
<h1>User profile</h1>
@if (isAdmin()) {
  <h2>Admin settings</h2>
  <!-- ... -->
} @else {
  <h2>User settings</h2>
  <!-- ... -->
}
```

### List Rendering

Angular uses the `track` keyword to associate data with the DOM elements created by `@for`:

```html
<h1>User profile</h1>
<ul class="user-badge-list">
  @for (badge of badges(); track badge.id) {
    <li class="user-badge">{{ badge.name }}</li>
  }
</ul>
```

---

## Forms (Signals-based)

### Basic Form

```typescript
import { ChangeDetectionStrategy, Component, signal } from '@angular/core';
import { form, FormField } from '@angular/forms/signals';

interface LoginData {
  email: string;
  password: string;
}

@Component({
  selector: 'app-root',
  templateUrl: 'app.html',
  styleUrl: 'app.css',
  imports: [FormField],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class App {
  loginModel = signal<LoginData>({
    email: '',
    password: '',
  });

  loginForm = form(this.loginModel);
}
```

```html
<form>
  <label>
    Email:
    <input type="email" [formField]="loginForm.email" />
  </label>
  <label>
    Password:
    <input type="password" [formField]="loginForm.password" />
  </label>
  <p>Hello {{ loginForm.email().value() }}!</p>
  <p>Password length: {{ loginForm.password().value().length }}</p>
</form>
```

### Form Validation

```typescript
import { ChangeDetectionStrategy, Component, signal } from '@angular/core';
import { email, form, FormField, required } from '@angular/forms/signals';

interface LoginData {
  email: string;
  password: string;
}

@Component({
  selector: 'app-root',
  templateUrl: 'app.html',
  styleUrl: 'app.css',
  imports: [FormField],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class App {
  loginModel = signal<LoginData>({
    email: '',
    password: '',
  });

  loginForm = form(this.loginModel, (schemaPath) => {
    required(schemaPath.email, { message: 'Email is required' });
    email(schemaPath.email, { message: 'Enter a valid email address' });
    required(schemaPath.password, { message: 'Password is required' });
  });

  onSubmit(event: Event) {
    event.preventDefault();
    // Perform login logic here
    const credentials = this.loginModel();
    console.log('Logging in with:', credentials);
    // e.g., await this.authService.login(credentials);
  }
}
```

---

## Dependency Injection & Services

When you need to share logic between components, Angular leverages **dependency injection** that allows you to create a "service" which allows you to inject code into components while managing it from a single source of truth.

### Defining a Service

```typescript
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class Calculator {
  add(x: number, y: number) {
    return x + y;
  }
}
```

### Injecting a Service

```typescript
import { Component, inject } from '@angular/core';
import { Calculator } from './calculator';

@Component({
  selector: 'app-receipt',
  template: `<h1>The total is {{ totalCost }}</h1>`,
})
export class Receipt {
  private calculator = inject(Calculator);
  totalCost = this.calculator.add(50, 25);
}
```

---

## View Encapsulation

Every component has a view encapsulation setting that determines how the framework scopes a component's styles. There are four view encapsulation modes:

| Mode | Description |
|------|-------------|
| **Emulated** *(default)* | Styles only apply to elements defined in that component's template. Angular generates a unique HTML attribute for each component instance and inserts it into CSS selectors. |
| **ShadowDom** | Uses the web standard Shadow DOM API. Styles inside the shadow tree cannot affect elements outside of it. Also impacts event propagation, `<slot>` API, and DevTools element display. |
| **ExperimentalIsolatedShadowDom** | Same as ShadowDom, but strictly guarantees global styles cannot affect elements in the shadow tree, and vice versa. |
| **None** | Disables all style encapsulation. All styles behave as global styles. |

---

## Component Selectors

Angular matches selectors **statically at compile-time**. Changing the DOM at run-time does not affect the components rendered.

### Types of Selectors

| Type | Example |
|------|---------|
| Type selector | `<profile-photo />` |
| Attribute selector | `<input type="text" [required]="true"/>` |
| Class selector | `.menu-item` |

### `:not` Pseudo-class

Angular supports the `:not` pseudo-class to narrow which elements a selector matches:

```typescript
@Component({
  selector: '[dropzone]:not(textarea)',
  ...
})
export class DropZone { }
```

### Attribute Selector on Native Elements

Consider an attribute selector when creating a component on a standard native element:

```typescript
@Component({
  selector: 'button[yt-upload]',
  ...
})
export class YouTubeUploadButton { }
```

---

## Component Inputs

### Basic Input

```typescript
import { Component, input } from '@angular/core';

@Component({ /*...*/ })
export class CustomSlider {
  // Declare an input named 'value' with a default value of zero.
  value = input<number>(0);
}
```

```html
<custom-slider [value]="50" />
```

The `input` function returns an `InputSignal`. You can read the value by calling the signal:

```typescript
// Create a computed expression that reads the value input
label = computed(() => `The slider's value is ${this.value()}`);
```

> Signals created by the `input` function are **read-only**.

### Input with Transform

```typescript
@Component({
  selector: 'custom-slider',
  /*...*/
})
export class CustomSlider {
  label = input('', { transform: trimString });
}

function trimString(value: string | undefined): string {
  return value?.trim() ?? '';
}
```

### Built-in Transform Functions

Angular includes two built-in transform functions for the most common scenarios:

```typescript
import { Component, input, booleanAttribute, numberAttribute } from '@angular/core';

@Component({ /*...*/ })
export class CustomSlider {
  disabled = input(false, { transform: booleanAttribute });
  value = input(0, { transform: numberAttribute });
}
```

### Input Alias

You can specify the `alias` option to change the name of an input in templates:

```typescript
@Component({ /*...*/ })
export class CustomSlider {
  value = input(0, { alias: 'sliderValue' });
}
```

```html
<custom-slider [sliderValue]="50" />
```
