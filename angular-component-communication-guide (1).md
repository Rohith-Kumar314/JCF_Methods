# Angular Component Communication: From Zero to Advanced

> **Goal:** After this guide you will never again wonder "is it `[]` or `()`?", and you will be able to choose the right communication technique for any app you build.
> **Version:** Written for **Angular 22.x** (22.0 shipped June 3, 2026; 22.1 on July 29, 2026). Needs **Node 22+** and **TypeScript 6**. Run `ng version` to confirm yours.
> **Code style:** Modern Angular: signal-based `input()`, `output()`, `model()`, the `@if` / `@for` template syntax, and the v22 defaults (components are standalone and **OnPush** by default; services use `@Service()`). The older decorator style (`@Input`, `@Output`, `@Injectable`) is shown side by side where you'll still meet it in real projects.

---

## Table of Contents

0. [What's New in Angular 22](#0-whats-new-in-angular-22)
1. [The One Mental Model That Ends the Confusion](#1-the-one-mental-model-that-ends-the-confusion)
2. [Template Syntax Basics](#2-template-syntax-basics-everything-that-looks-like-or-)
3. [Parent → Child with `input()`](#3-parent--child-input)
4. [Child → Parent with `output()`](#4-child--parent-output)
5. [Two-Way Binding `[( )]` with `model()`](#5-two-way-binding--with-model)
6. [Template Reference Variables and `viewChild`](#6-template-reference-variables-and-viewchild)
7. [Content Projection (`ng-content`)](#7-content-projection-ng-content)
8. [Services: Talking Across Any Components](#8-services-talking-across-any-components)
9. [RxJS Subjects and Signals Interop](#9-rxjs-subjects-and-signals-interop)
10. [Router-Based Communication](#10-router-based-communication)
11. [Advanced Patterns](#11-advanced-patterns)
12. [Decision Guide: Which Technique Do I Use?](#12-decision-guide-which-technique-do-i-use)
13. [Top 12 Mistakes (and Fixes)](#13-top-12-mistakes-and-fixes)
14. [Cheat Sheet](#14-cheat-sheet)
15. [Practice: Drills](#15-practice-drills)
16. [Practice: Build the "TaskFlow" App](#16-practice-build-the-taskflow-app)
17. [Answers to Drills and Reference Solution Pieces](#17-answers-to-drills-and-reference-solution-pieces)

---

## 0. What's New in Angular 22

You do not need any of this to understand `[]` and `()`: those rules never change. But v22 changed some **defaults and recommended APIs**, and this guide uses them.

| Area | Angular 22 | Where in this guide |
|---|---|---|
| Change detection | **`OnPush` is now the default.** The old default is renamed `Eager`. | 11.2 |
| Services | New **`@Service()`** decorator (same as `@Injectable({ providedIn: 'root' })`); the CLI generates it. | 8 |
| Lazy services | New **`injectAsync()`** loads a service only when needed. | 8.5 |
| Forms | **Signal Forms are stable** (`@angular/forms/signals`, `[formField]`). | 11.7 |
| Custom form controls | **`FormValueControl`** replaces the `ControlValueAccessor` ceremony. | 11.8 |
| Async data | **`resource()`, `rxResource()`, `httpResource()` are stable.** `debounced()` is new (experimental). | 9.5 |
| Router | `withComponentInputBinding()` takes options; `paramsInheritanceStrategy` now defaults to `'always'`. | 10 |
| Compiler | `strictTemplates` on by default; an input/model and an output with the same name is now an **error (NG1054)**. | 5 |
| HTTP | `HttpClient` uses **Fetch** by default. | n/a |
| Standalone | Components are standalone by default (since v19), so **no `standalone: true`**. | everywhere |

**File naming note:** recent CLI versions generate `task-item.ts` with a class called `TaskItem` (no `Component` suffix). This guide keeps the `Component` suffix in class names because it makes the examples easier to read. Both styles are fine; just be consistent.

**Upgrading an older project?** Run `ng update @angular/core @angular/cli`. Migrations add `changeDetection: Eager` to your existing components so nothing silently changes behaviour.

---

## 1. The One Mental Model That Ends the Confusion

Think of every component as a **box with a wall**.

```
            ┌───────────────── PARENT ─────────────────┐
            │                                          │
            │      data goes IN        events come OUT │
            │          │                     ▲         │
            │          ▼                     │         │
            │   ┌──────────── CHILD ───────────────┐   │
            │   │    [ square brackets ]  ( round  │   │
            │   │                         brackets)│   │
            │   └──────────────────────────────────┘   │
            └──────────────────────────────────────────┘
```

| Syntax | Direction | Say it out loud | Think of it as |
|---|---|---|---|
| `[prop]="expr"` | **Data goes INTO** the target | "set property *prop* to the value of *expr*" | **Square = box = put something in the box** |
| `(event)="handler()"` | **Events come OUT of** the target | "when *event* happens, run *handler*" | **Round = a speech bubble / "listen"** |
| `[(prop)]="x"` | **Both** | "keep *prop* and *x* in sync" | **Banana in a box** 🍌📦 (round inside square) |
| `{{ expr }}` | Data into **text** | "print this value" | Interpolation |
| `#name` | Nothing flows; it's a **label** | "give this element a nickname" | Template reference |

### The golden rules (memorize these four lines)

1. **`[ ]` → the right side is an *expression* evaluated by Angular, and the result is pushed into the target.**
2. **`( )` → the right side is a *statement* (a function call or assignment) that runs when the event fires.**
3. **The thing in the brackets is always the *child's / element's* name; the thing in the quotes is always the *parent's* (your own) stuff.**
4. **`[ ]` and `( )` work on BOTH native HTML elements AND your own components. The syntax is identical; only the target differs.**

```html
<!-- Target is a NATIVE element: <button> -->
<button [disabled]="isSaving" (click)="save()">Save</button>

<!-- Target is YOUR component: <app-user-card> -->
<app-user-card [user]="selectedUser" (deleted)="onDelete($event)" />
```

Read the second line as: *"Into app-user-card's `user` input, put my `selectedUser`. When app-user-card's `deleted` output fires, run my `onDelete` with the payload."*

### The "which side is whose?" trick

```html
<app-child  [childThing]="myThing"  (childEvent)="myHandler($event)" />
              ▲           ▲             ▲            ▲
        child's name   my name     child's name   my name
```

Left of `=` is **the child's API**. Right of `=` is **your own code**. If you remember only one thing, remember this.

---

## 2. Template Syntax Basics: Everything That Looks Like `[]` or `()`

Before components talk to each other, learn how a template talks to its **own** class. All examples use this component:

```ts
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-demo',
  template: `...see below...`,
})
export class DemoComponent {
  name = signal('Asha');
  isLoading = signal(false);
  imageUrl = 'assets/logo.png';
  count = signal(0);

  increment() { this.count.update(c => c + 1); }
}
```

### 2.1 Interpolation `{{ }}`: data → text

```html
<h1>Hello, {{ name() }}!</h1>        <!-- signals are READ by calling them: name() -->
<p>2 + 2 = {{ 2 + 2 }}</p>
```

> Signals are functions. In templates and in code, **read a signal with `()`**. This `()` is a *function call*, not event binding. Don't confuse them; event binding `()` always appears **around a name on the left of `=`**.

### 2.2 Property binding `[prop]`: data → DOM/component property

```html
<img [src]="imageUrl" />
<button [disabled]="isLoading()">Save</button>
<input [value]="name()" />
```

Why not just `src="{{ imageUrl }}"`? That works for strings, but `[ ]` is the universal way and keeps types (booleans, objects, arrays) intact. **`[disabled]="false"` really sets false. `disabled="false"` (plain HTML attribute) still disables the button!** This is a classic trap.

### 2.3 Attribute, class and style bindings

```html
<td [attr.colspan]="span">…</td>           <!-- attributes with no DOM property -->
<div [class.active]="isActive">…</div>      <!-- toggle ONE class -->
<div [class]="{ active: isActive, big: isBig }">…</div>
<p [style.color]="isError ? 'red' : 'green'">…</p>
<p [style.width.px]="width">…</p>           <!-- unit suffix -->
```

### 2.4 Event binding `(event)`: DOM/component → your code

```html
<button (click)="increment()">+1</button>
<input (input)="onType($event)" />
<input (keyup.enter)="submit()" />          <!-- key filtering built in -->
```

`$event` is the payload of the event:

- For **native DOM events**, `$event` is the DOM `Event` object.
- For **component outputs**, `$event` is whatever the child emitted.

```ts
onType(event: Event) {
  const value = (event.target as HTMLInputElement).value;
  this.name.set(value);
}
```

### 2.5 Two-way binding `[( )]`: banana in a box

With forms (`FormsModule` imported):

```html
<input [(ngModel)]="username" />
```

This is just **shorthand** for:

```html
<input [ngModel]="username" (ngModelChange)="username = $event" />
```

> **This is the single most important insight about `[( )]`:** it's *not* new magic. It is `[x]="v"` **plus** `(xChange)="v = $event"`. Angular looks for an input named `x` and an output named `xChange`. That naming convention is the entire trick. We'll build our own in section 5.

### 2.6 Template reference variable `#`

```html
<input #box (keyup.enter)="add(box.value)" />
<button (click)="add(box.value)">Add</button>
```

`#box` gives you the element itself, usable anywhere **in the same template**.

### 2.7 Quick self-test

What's wrong?

```html
<button disabled="isLoading()">Save</button>       <!-- (a) -->
<button (click)="increment">+1</button>            <!-- (b) -->
<img src="imageUrl" />                             <!-- (c) -->
```

<details><summary>Answers</summary>

- (a) Plain attribute: the string `"isLoading()"` is truthy, so the button is *always* disabled. Use `[disabled]="isLoading()"`.
- (b) Missing `()`: that references the method but never calls it. Use `(click)="increment()"`.
- (c) Looks for a file literally named `imageUrl`. Use `[src]="imageUrl"`.
</details>

---

## 3. Parent → Child: `input()`

**Use when:** the parent has data and wants to *give* it to a child (display only, or as configuration).

### 3.1 Minimal example (modern signal input)

**Child:**

```ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  template: `
    <div class="card">
      <h3>{{ name() }}</h3>
      <p>Age: {{ age() }}</p>
    </div>
  `,
})
export class UserCardComponent {
  name = input<string>('Unknown');   // optional, with default
  age  = input.required<number>();   // required: compile error if parent forgets it
}
```

**Parent:**

```ts
@Component({
  selector: 'app-root',
  imports: [UserCardComponent],
  template: `
    <!-- [ ] because we pass an EXPRESSION (a variable / number) -->
    <app-user-card [name]="userName" [age]="31" />

    <!-- no [ ] because it's a constant STRING -->
    <app-user-card name="Static Name" [age]="22" />
  `,
})
export class AppComponent {
  userName = 'Ravi';
}
```

### 3.2 `[age]="31"` vs `age="31"`: the classic trap

| You write | Child receives | Type |
|---|---|---|
| `age="31"` | `"31"` | **string** |
| `[age]="31"` | `31` | **number** |
| `[age]="'31'"` | `"31"` | string (you made a string expression) |
| `flag="false"` | `"false"` | string (truthy!) |
| `[flag]="false"` | `false` | boolean |

**Rule:** if it isn't a plain string, use `[ ]`.

### 3.3 Reading an input

Signal inputs are **read-only signals**; call them: `name()`. They work seamlessly with `computed()` and `effect()`:

```ts
import { computed, effect } from '@angular/core';

export class UserCardComponent {
  firstName = input.required<string>();
  lastName  = input.required<string>();

  fullName = computed(() => `${this.firstName()} ${this.lastName()}`);

  constructor() {
    effect(() => console.log('Name changed to', this.fullName()));
  }
}
```

> You **cannot** assign to an input inside the child (`this.name.set(...)` doesn't exist). Data flows *down only*. If the child needs to change it, it must **ask** the parent via an output (section 4) or use `model()` (section 5).

### 3.4 Alias, transform

```ts
import { booleanAttribute, numberAttribute } from '@angular/core';

export class ButtonComponent {
  // parent writes [label]="..." but class property is called text
  text     = input('', { alias: 'label' });

  // <app-button disabled /> now means true (HTML-style boolean attribute)
  disabled = input(false, { transform: booleanAttribute });

  // <app-box size="40" /> becomes the number 40
  size     = input(16, { transform: numberAttribute });
}
```

### 3.5 Older decorator style (you'll see it everywhere)

```ts
import { Input, OnChanges, SimpleChanges } from '@angular/core';

export class UserCardComponent implements OnChanges {
  @Input() name = 'Unknown';
  @Input({ required: true }) age!: number;

  ngOnChanges(changes: SimpleChanges) {
    if (changes['name']) {
      console.log('name changed:', changes['name'].previousValue, '→', changes['name'].currentValue);
    }
  }
}
```

Same parent template. The only differences are *how the child declares and reads it*.

### 3.6 Objects and arrays are passed **by reference**

```ts
// Child accidentally mutates the parent's object!
user = input.required<User>();
rename() { this.user().name = 'Hacked'; }   // ❌ mutates parent's data
```

Treat inputs as **immutable**. If the child wants to change something, it emits a *new* value or event and the parent updates (section 4).

---

## 4. Child → Parent: `output()`

**Use when:** something happened in the child (click, selection, submit) and the parent needs to know.

### 4.1 Minimal example

**Child:**

```ts
import { Component, input, output } from '@angular/core';

@Component({
  selector: 'app-todo-item',
  template: `
    <span [class.done]="todo().done">{{ todo().title }}</span>
    <button (click)="toggled.emit(todo().id)">Toggle</button>
    <button (click)="deleted.emit(todo().id)">🗑</button>
  `,
})
export class TodoItemComponent {
  todo    = input.required<Todo>();      // data IN
  toggled = output<number>();            // event OUT, payload: number
  deleted = output<number>();
}
```

**Parent:**

```ts
@Component({
  selector: 'app-todo-list',
  imports: [TodoItemComponent],
  template: `
    @for (t of todos(); track t.id) {
      <app-todo-item
        [todo]="t"
        (toggled)="onToggle($event)"
        (deleted)="onDelete($event)" />
    }
  `,
})
export class TodoListComponent {
  todos = signal<Todo[]>([
    { id: 1, title: 'Learn Angular', done: false },
    { id: 2, title: 'Build app', done: false },
  ]);

  onToggle(id: number) {
    this.todos.update(list =>
      list.map(t => t.id === id ? { ...t, done: !t.done } : t));
  }
  onDelete(id: number) {
    this.todos.update(list => list.filter(t => t.id !== id));
  }
}
```

Notice the full cycle, called **"data down, events up"**:

```
Parent  ──[todo]="t"────────▶  Child        (data DOWN)
Parent  ◀──(toggled)="..."───  Child        (event UP)
Parent updates its state → new data flows down again.
```

> **`$event` here is the payload passed to `.emit(...)`**, not a DOM event. `toggled.emit(5)` means `$event === 5`.

### 4.2 Payload design

- Emit **one thing**: a primitive, or an object for multiple values.

```ts
saved = output<{ id: number; title: string }>();
this.saved.emit({ id: 1, title: 'New' });
```

- For "just a signal, no data," use `output<void>()` and `.emit()`.

### 4.3 Naming convention

- Name outputs as **events in past tense or "what happened"**: `saved`, `deleted`, `selectionChange`. Not `onSave` (it hides the fact it's a notification, not a handler).
- **Don't** name an output like a native event (`click`, `change`): a native event of the same name from the host element will also fire your handler and cause confusion.

### 4.4 Older style

```ts
import { Output, EventEmitter } from '@angular/core';

export class TodoItemComponent {
  @Output() toggled = new EventEmitter<number>();
  // usage: this.toggled.emit(5);
}
```

Parent template is identical.

### 4.5 Output → parent method that is *itself* an input of another child

Siblings can communicate **via the parent**: Child A emits → parent stores → parent passes to Child B.

```html
<app-filter  (changed)="filter.set($event)" />
<app-results [filter]="filter()" />
```

`filter.set($event)` is a legitimate statement in `( )`. No handler method needed.

---

## 5. Two-Way Binding `[( )]` with `model()`

**Use when:** parent and child should always agree on a value (custom inputs, toggles, sliders, tabs, dialogs `open` state).

### 5.1 Revealing the magic

```html
<app-counter [(value)]="count" />
```

is exactly

```html
<app-counter [value]="count" (valueChange)="count = $event" />
```

So a component supports `[( )]` for `value` if it has an **input called `value`** and an **output called `valueChange`**.

### 5.2 `model()` creates both at once

```ts
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <button (click)="value.update(v => v - 1)">−</button>
    <span>{{ value() }}</span>
    <button (click)="value.update(v => v + 1)">+</button>
  `,
})
export class CounterComponent {
  value = model(0);        // creates input "value" AND output "valueChange"
}
```

Parent (works with a signal **or** a plain property):

```ts
@Component({
  imports: [CounterComponent],
  template: `
    <app-counter [(value)]="count" />
    <p>Parent sees: {{ count() }}</p>
    <button (click)="count.set(0)">Reset from parent</button>
  `,
})
export class AppComponent {
  count = signal(5);
}
```

Unlike `input()`, a `model()` is **writable inside the child** (`.set()`, `.update()`), and the change flows *up* automatically.

> **Angular 22 compiler rule (NG1054):** `value = model(0)` already creates the `valueChange` output. Do **not** also declare `valueChange = output<number>()`, or any second input/output that ends up with the same public name. The compiler now reports it as an error.

### 5.3 Pre-signal way (manual)

```ts
export class CounterComponent {
  @Input() value = 0;
  @Output() valueChange = new EventEmitter<number>();

  inc() { this.valueChange.emit(this.value + 1); }
}
```

Name matching (`value` + `valueChange`) is essential.

### 5.4 When NOT to use two-way binding

Two-way binding is convenient but can make data flow hard to trace in large apps. Prefer *one-way in + event out* when the parent needs to **validate, transform, or reject** the change. Use `model()` for simple "UI control" components.

---

## 6. Template Reference Variables and `viewChild`

**Use when:** the parent needs to *directly* call a method or read a property of a child (imperative control). Examples: `focus()`, `reset()`, `open()`, `scrollTo()`.

### 6.1 In the template: `#ref`

```ts
@Component({
  selector: 'app-timer',
  template: `<p>{{ seconds() }}s</p>`,
})
export class TimerComponent {
  seconds = signal(0);
  private id?: ReturnType<typeof setInterval>;
  start() { this.id = setInterval(() => this.seconds.update(s => s + 1), 1000); }
  stop()  { clearInterval(this.id); }
  reset() { this.stop(); this.seconds.set(0); }
}
```

```html
<app-timer #timer />
<button (click)="timer.start()">Start</button>
<button (click)="timer.stop()">Stop</button>
<button (click)="timer.reset()">Reset</button>
```

`#timer` on a component gives you the **component instance**. On a plain element it gives you the **DOM element**.

> **Limit:** `#timer` is only visible *inside this template*, not in the parent's TypeScript.

### 6.2 In TypeScript: `viewChild()`

```ts
import { viewChild, afterNextRender } from '@angular/core';

export class AppComponent {
  timer = viewChild.required(TimerComponent);      // component
  input = viewChild<ElementRef<HTMLInputElement>>('box');  // by #box name

  startAll() {
    this.timer().start();                     // a signal! call it
    this.input()?.nativeElement.focus();
  }
}
```

- `viewChild(...)` → the **first** match, or `undefined`.
- `viewChild.required(...)` → throws if absent.
- `viewChildren(...)` → all matches (a signal of array).
- Results are available **after the view renders**, not in the constructor. Use them in event handlers, `effect()`, or `afterNextRender()`.

Decorator style: `@ViewChild(TimerComponent) timer!: TimerComponent;`

### 6.3 Should you use it?

It's *imperative* and tightly couples parent and child. Prefer inputs/outputs first. Reach for `viewChild` for things that are naturally commands (focus, play, scroll, open modal).

---

## 7. Content Projection (`ng-content`)

**Use when:** you're building a **reusable wrapper** (card, modal, tabs, layout) and want the parent to supply *HTML* as the content, not just data.

### 7.1 Single slot

```ts
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <ng-content />
    </div>
  `,
})
export class CardComponent {}
```

```html
<app-card>
  <h3>Anything here</h3>
  <p>…gets rendered where ng-content is.</p>
</app-card>
```

### 7.2 Multiple slots with `select`

```ts
template: `
  <header><ng-content select="[card-title]" /></header>
  <main><ng-content /></main>                         <!-- default slot -->
  <footer><ng-content select="[card-actions]" /></footer>
`
```

```html
<app-card>
  <h3 card-title>Profile</h3>
  <p>Body goes to the default slot.</p>
  <button card-actions (click)="save()">Save</button>
</app-card>
```

**Important:** projected content is **created and owned by the parent**. So `(click)="save()"` refers to the *parent's* `save()`. The wrapper doesn't need to know anything about it. This is "inversion of control."

### 7.3 Passing a *template* with data: `ngTemplateOutlet`

When the child must render the parent's markup **per item**, passing its own data back:

**Child (a generic list):**

```ts
import { NgTemplateOutlet } from '@angular/common';
import { contentChild, TemplateRef } from '@angular/core';

@Component({
  selector: 'app-list',
  imports: [NgTemplateOutlet],
  template: `
    <ul>
      @for (item of items(); track $index) {
        <li>
          <ng-container
            [ngTemplateOutlet]="rowTpl() ?? defaultTpl"
            [ngTemplateOutletContext]="{ $implicit: item, index: $index }" />
        </li>
      }
    </ul>
    <ng-template #defaultTpl let-item>{{ item }}</ng-template>
  `,
})
export class ListComponent<T> {
  items  = input.required<T[]>();
  rowTpl = contentChild<TemplateRef<unknown>>('row');
}
```

**Parent:**

```html
<app-list [items]="users()">
  <ng-template #row let-user let-i="index">
    <strong>{{ i + 1 }}.</strong> {{ user.name }} ({{ user.email }})
  </ng-template>
</app-list>
```

Here the **child supplies data** (`let-user` = `$implicit`) and the **parent supplies layout**. This is how libraries like Angular Material build flexible tables.

### 7.4 `contentChild` vs `viewChild`

| | Looks at |
|---|---|
| `viewChild` | What's in **my own** template |
| `contentChild` | What was **projected into me** between my tags |

---

## 8. Services: Talking Across Any Components

**Use when:** components are **not** parent/child (siblings, cousins, distant relatives), or lots of components need the same state (cart, auth user, theme, notifications).

Passing data through 5 levels of inputs ("prop drilling") is painful. Instead, put the data in a **service** and let any component `inject()` it.

### 8.1 A signal-based store service

```ts
// cart.service.ts
import { Service, signal, computed } from '@angular/core';

export interface CartItem { id: number; name: string; price: number; qty: number; }

@Service()                                 // ONE shared instance for the whole app
export class CartService {
  private readonly _items = signal<CartItem[]>([]);

  // Expose read-only views so outsiders can't mutate directly
  readonly items = this._items.asReadonly();
  readonly count = computed(() => this._items().reduce((n, i) => n + i.qty, 0));
  readonly total = computed(() => this._items().reduce((s, i) => s + i.price * i.qty, 0));

  add(product: Omit<CartItem, 'qty'>) {
    this._items.update(list => {
      const found = list.find(i => i.id === product.id);
      return found
        ? list.map(i => i.id === product.id ? { ...i, qty: i.qty + 1 } : i)
        : [...list, { ...product, qty: 1 }];
    });
  }
  remove(id: number) { this._items.update(l => l.filter(i => i.id !== id)); }
  clear()            { this._items.set([]); }
}
```

> **`@Service()` is the Angular 22 way** to write what used to be `@Injectable({ providedIn: 'root' })`. Same behaviour, shorter, and `ng generate service` creates it by default. One rule: a `@Service()` class must get its own dependencies with `inject()` (not constructor parameters). If you open an older codebase you'll see `@Injectable({ providedIn: 'root' })` everywhere; it still works, and Angular 22.1 added a migration schematic to convert it.

### 8.2 Two unrelated components use it

```ts
// product-list.component.ts: WRITES to the service
@Component({
  selector: 'app-product-list',
  template: `
    @for (p of products; track p.id) {
      <div>{{ p.name }} {{ p.price }}
        <button (click)="cart.add(p)">Add</button>
      </div>
    }
  `,
})
export class ProductListComponent {
  cart = inject(CartService);
  products = [{ id: 1, name: 'Pen', price: 10 }, { id: 2, name: 'Book', price: 250 }];
}

// header-cart-badge.component.ts: READS from the service
@Component({
  selector: 'app-cart-badge',
  template: `🛒 {{ cart.count() }} items | ₹{{ cart.total() }}`,
})
export class CartBadgeComponent {
  cart = inject(CartService);
}
```

No inputs, no outputs, no parent. Both components share the *same* service instance, so the badge updates instantly.

### 8.3 Service scope: *one instance* vs *one per component*

| How provided | Instances |
|---|---|
| `@Service()` (or `@Injectable({ providedIn: 'root' })`) | One for the whole app (singleton) |
| `@Service({ autoProvided: false })` + `providers: [MyService]` in a **component** | A **new instance per component** (shared with its children only) |
| `providers` in a **route** config | One per route subtree |

```ts
@Service({ autoProvided: false })   // NOT global: you must provide it yourself
export class WizardState { /* signals… */ }

@Component({
  selector: 'app-wizard',
  providers: [WizardState],         // fresh state each time <app-wizard> is used
  ...
})
```

(You can also use `@Injectable()` with no options for this case, e.g. `ng generate service wizard-state --injectable`.)

Use component-level providers when each *instance* of a feature should have its own state (a wizard, a dialog, a row editor) while its internal children still share it.

### 8.4 Rules of thumb for services

- Keep fields `private`/`readonly` and expose **methods** to change state. This is your "events up" channel.
- Expose `asReadonly()` signals. Consumers can *read* but not *write*.
- Services should not know about components or the DOM.

### 8.5 Lazy-loading a heavy service with `injectAsync()` (new in v22)

For a service you only need after a user action (PDF export, charting, rich-text tools), don't pay for it at startup:

```ts
export class AdminComponent {
  private reports = injectAsync(() => import('./report.service').then(m => m.ReportService));

  async exportPdf() {
    const svc = await this.reports();   // downloaded + created on first use
    await svc.exportPdf();
  }
}
```

The service must be auto-provided (`@Service()` or `providedIn: 'root'`), and `injectAsync` must be called in an injection context (a field initializer is perfect). For everyday state-sharing services, plain `inject()` is still what you want.

---

## 9. RxJS Subjects and Signals Interop

Signals are great for **state** ("what is the current value?"). RxJS is great for **streams of events over time** (debounce, HTTP, websockets, cancellation).

### 9.1 Event bus with `Subject` (use sparingly)

```ts
@Service()
export class NotificationService {
  private readonly _messages$ = new Subject<string>();
  readonly messages$ = this._messages$.asObservable();

  notify(msg: string) { this._messages$.next(msg); }
}
```

```ts
// any component
this.notifications.messages$
  .pipe(takeUntilDestroyed(this.destroyRef))
  .subscribe(msg => this.toast(msg));
```

### 9.2 `Subject` vs `BehaviorSubject` vs `ReplaySubject`

| Type | New subscribers get… | Use for |
|---|---|---|
| `Subject` | Only future events | Fire-and-forget events (toast, "saved!") |
| `BehaviorSubject(initial)` | The **latest value** immediately | Current state (selected user, theme) |
| `ReplaySubject(n)` | The last *n* values | History/log |

### 9.3 Template consumption

```html
@if (user$ | async; as user) { {{ user.name }} }
```

or convert to a signal:

```ts
import { toSignal, toObservable } from '@angular/core/rxjs-interop';

user  = toSignal(this.userService.user$, { initialValue: null });
query = signal('');
results = toSignal(
  toObservable(this.query).pipe(
    debounceTime(300),
    switchMap(q => this.api.search(q))
  ),
  { initialValue: [] }
);
```

### 9.4 Preventing memory leaks

Subscriptions to long-lived streams (services) must be cleaned up:

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
// in constructor / injection context:
stream$.pipe(takeUntilDestroyed()).subscribe(...);
```

`async` pipe and `toSignal` clean up automatically.

### 9.5 Async data with resources (stable in v22)

For "fetch something whenever these signals change", Angular 22 recommends **resources** instead of hand-wiring `toObservable` + `switchMap` + `toSignal`:

```ts
import { httpResource } from '@angular/common/http';

export class UserDetailComponent {
  id = input.required<string>();

  // re-fetches automatically whenever id() changes
  user = httpResource<User>(() => `/api/users/${this.id()}`);
}
```

```html
@if (user.isLoading()) { <p>Loading…</p> }
@else if (user.error()) { <p>Failed. <button (click)="user.reload()">Retry</button></p> }
@else if (user.hasValue()) { <h2>{{ user.value().name }}</h2> }
```

Related tools: `resource()` (any Promise), `rxResource()` (any Observable), and the **experimental** `debounced(() => this.query(), 300)` which gives you a *resource* that settles only after the source signal has stopped changing for 300 ms. Feed `debounced(...).value()` into an `httpResource` for a clean search-as-you-type box.

---

## 10. Router-Based Communication

URLs are also a communication channel: shareable, bookmarkable, and survive refresh.

### 10.1 Route params → component **inputs** (the clean modern way)

```ts
// app.config.ts
provideRouter(routes, withComponentInputBinding())

// routes
{ path: 'users/:id', component: UserDetailComponent }
```

```ts
export class UserDetailComponent {
  id = input.required<string>();      // from :id
  tab = input<string>();              // from ?tab=profile  (query param)
  user = httpResource<User>(() => `/api/users/${this.id()}`);   // see 9.5
}
```

`withComponentInputBinding()` takes options in v22: `queryParams` (default `true`; set `false` to stop binding query params to inputs) and `unmatchedInputBehavior` (`'alwaysUndefined'` default, or `'undefinedIfStale'`).

> **Heads-up:** in v22 `paramsInheritanceStrategy` defaults to `'always'`, so child routes inherit parent route params. If an old app relied on the previous behaviour, set it to `'emptyOnly'` in `provideRouter(routes, withRouterConfig({ paramsInheritanceStrategy: 'emptyOnly' }))`.

Navigate:

```html
<a [routerLink]="['/users', u.id]" [queryParams]="{ tab: 'profile' }">Open</a>
```

### 10.2 Route `data` and `resolve`

```ts
{ path: 'admin', component: AdminComponent, data: { title: 'Admin' } }
```

Also bound to inputs when `withComponentInputBinding()` is on.

### 10.3 Pass state during navigation

```ts
this.router.navigate(['/checkout'], { state: { coupon: 'SAVE10' } });
// in target: history.state.coupon   (lost on hard refresh: use only for convenience data)
```

---

## 11. Advanced Patterns

### 11.1 Smart (container) vs Dumb (presentational) components

| Smart / container | Dumb / presentational |
|---|---|
| Injects services, fetches data | Only `input()`s and `output()`s |
| Owns state | Owns nothing (maybe tiny UI state) |
| Few of them (pages) | Many of them (cards, rows, buttons) |
| Hard to reuse | Easy to reuse and test |

```
UserPageComponent (smart)  ── inject(UserService)
 ├─ UserFilterComponent (dumb)   [filter] (filterChange)
 ├─ UserListComponent   (dumb)   [users]  (selected)
 │    └─ UserRowComponent (dumb) [user]   (delete)
 └─ UserDetailComponent (dumb)   [user]   (save)
```

This one habit prevents ~80% of messy Angular code.

### 11.2 `OnPush` is the default in v22 (so immutability matters everywhere)

Before v22 you had to write `changeDetection: ChangeDetectionStrategy.OnPush` yourself. **Now every component is `OnPush` unless you say otherwise.** The old behaviour is called `Eager`:

```ts
@Component({
  changeDetection: ChangeDetectionStrategy.Eager,   // opt OUT (rarely needed; migrations add this to old components)
  ...
})
```

An `OnPush` component re-renders only when: an **input's reference changes**, an event fires inside it, a **signal it reads** changes, or `async` emits. Hence:

```ts
// ❌ same array reference: the OnPush child will not update
this.todos().push(newTodo);

// ✅ new reference
this.todos.update(l => [...l, newTodo]);
```

Signals + immutable updates + `OnPush` = fast and predictable. Because this is now the default for **all** your components, the "never mutate inputs or signal values" rule from sections 3.6 and 11.2 is not optional style advice anymore: mutations are what make the screen appear "stuck".

### 11.3 `@for` and `track`

```html
@for (t of todos(); track t.id) {
  <app-todo-item [todo]="t" (deleted)="remove($event)" />
}
```

`track t.id` lets Angular reuse DOM and component instances instead of rebuilding the list.

### 11.4 Host bindings and listeners: the component binding **itself**

`[ ]` and `( )` also apply to the component's own host element via the `host` metadata:

```ts
@Component({
  selector: 'app-tile',
  host: {
    '[class.selected]': 'selected()',
    '[attr.aria-pressed]': 'selected()',
    '(click)': 'toggle()',
    '(keydown.space)': 'toggle()',
    'role': 'button',
    'tabindex': '0',
  },
  template: `<ng-content />`,
})
export class TileComponent {
  selected = model(false);
  toggle() { this.selected.update(s => !s); }
}
```

Same rule: `[ ]` pushes values **into** the host element; `( )` **listens** to the host.

### 11.5 Parent injected by the child (compound components)

When a set of components is designed to work together (tabs, accordion, stepper), a child can inject its parent:

```ts
@Component({ selector: 'app-tabs', template: `...`, providers: [] })
export class TabsComponent {
  tabs = signal<TabComponent[]>([]);
  active = signal(0);
  register(t: TabComponent) { this.tabs.update(l => [...l, t]); }
}

@Component({ selector: 'app-tab', template: `@if (isActive()) { <ng-content /> }` })
export class TabComponent {
  private parent = inject(TabsComponent);
  title = input.required<string>();
  private index = this.parent.tabs().length;
  isActive = computed(() => this.parent.active() === this.index);
  constructor() { this.parent.register(this); }
}
```

Use for tightly-coupled families only. A tiny, hidden contract between components.

### 11.6 `InjectionToken` for configuration or swapping behaviour

```ts
export const API_URL = new InjectionToken<string>('API_URL');

// app.config.ts
providers: [{ provide: API_URL, useValue: 'https://api.example.com' }]

// anywhere
private baseUrl = inject(API_URL);
```

### 11.7 Signal Forms in 60 seconds (stable in v22)

Signal Forms put a **plain signal** at the centre; the form is a reactive view over it. Everything lives in `@angular/forms/signals`.

```ts
import { Component, signal } from '@angular/core';
import { form, FormField, required } from '@angular/forms/signals';

@Component({
  selector: 'app-task-editor',
  imports: [FormField],
  template: `
    <input [formField]="taskForm.title" />
    @if (taskForm.title().touched() && taskForm.title().invalid()) {
      <small>Title is required</small>
    }
    <button [disabled]="taskForm().invalid()" (click)="save()">Save</button>
  `,
})
export class TaskEditorComponent {
  model = signal({ title: '', priority: 3, done: false });     // the source of truth

  taskForm = form(this.model, path => {
    required(path.title);                                        // validation rules
  });

  save() { console.log(this.model()); }                          // always the latest values
}
```

How this fits the `[]` / `()` mental model: `[formField]` is just another **input-style binding**. The thing on the left (`formField`) belongs to the directive; the thing on the right (`taskForm.title`) is *your* field. No `ngModelChange` or `valueChange` needed: values flow both ways automatically.

Gotchas:
- `[formField]="taskForm.title"` (the field path), **not** `taskForm().title`. `taskForm()` gives you the *state* of the whole form; `taskForm.title()` gives you the *state* of one field (`.value()`, `.touched()`, `.invalid()`, `.errors()`).
- Update the model with `.set()` / `.update()`. Never replace it with a new `signal(...)`, or the form keeps pointing at the old one.
- Older tutorials (from the experimental days, Angular 21 and earlier) use `[field]`; in v22 the directive is `[formField]`.
- Legacy Reactive Forms and template-driven `ngModel` still work. New code should prefer Signal Forms; migrate old forms gradually.

### 11.8 Custom form controls with `FormValueControl`

Before v22 (and in lots of existing code) a custom control meant implementing `ControlValueAccessor`: `writeValue`, `registerOnChange`, `registerOnTouched`, `setDisabledState`, plus an `NG_VALUE_ACCESSOR` provider. With Signal Forms, the contract is basically **"expose a `value` model."**

```ts
import { Component, input, model } from '@angular/core';
import { FormValueControl } from '@angular/forms/signals';

@Component({
  selector: 'app-rating',
  template: `
    @for (s of [1, 2, 3, 4, 5]; track s) {
      <button type="button" [disabled]="disabled()" (click)="value.set(s)">
        {{ s <= value() ? '★' : '☆' }}
      </button>
    }
  `,
})
export class RatingComponent implements FormValueControl<number> {
  value = model(0);                 // REQUIRED: the one thing the interface needs
  disabled = input(false);          // optional: Angular binds the field's disabled state to it
}
```

```html
<app-rating [formField]="taskForm.priority" />
```

You opt in to more by declaring more optional signals (`disabled`, `readonly`, `errors`, `touched`…). Declare only what your control needs. Note: in v22 the `touched` state is an **input** plus a `touch()` **output** (the old writable `touched` model was removed during stabilization). Use a checkbox-style `FormCheckboxControl` (with a `checked` model) for boolean controls.

**Works with older forms too:** per the Angular docs a `FormValueControl` can also be used inside Reactive and template-driven forms, unchanged:

```html
<app-rating formControlName="priority" />
<app-rating [(ngModel)]="myRating" />
```

#### Legacy: `ControlValueAccessor`

You'll still meet this in existing projects and third-party widgets, and v22 keeps it fully working (even inside signal forms). The same rating control the old way:

```ts
import { Component, forwardRef, signal } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-rating-legacy',
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => RatingLegacyComponent),
    multi: true,
  }],
  template: `
    @for (s of [1,2,3,4,5]; track s) {
      <button type="button" [disabled]="disabled()" (click)="select(s)">
        {{ s <= value() ? '★' : '☆' }}
      </button>
    }
  `,
})
export class RatingLegacyComponent implements ControlValueAccessor {
  value = signal(0);
  disabled = signal(false);
  private onChange: (v: number) => void = () => {};
  private onTouched: () => void = () => {};

  select(v: number) {
    if (this.disabled()) return;
    this.value.set(v);
    this.onChange(v);       // tell the form: value changed
    this.onTouched();
  }
  writeValue(v: number | null) { this.value.set(v ?? 0); }       // form → component
  registerOnChange(fn: (v: number) => void) { this.onChange = fn; }
  registerOnTouched(fn: () => void) { this.onTouched = fn; }
  setDisabledState(d: boolean) { this.disabled.set(d); }
}
```

Compare the sizes: that's why `FormValueControl` is the new default.

### 11.9 Larger state: when a service isn't enough

| App size | Tool |
|---|---|
| Small/medium | Signal-based services (section 8) |
| Medium/large, many features | NgRx **SignalStore** (`@ngrx/signals`) |
| Large, event-sourcing, devtools, strict patterns | NgRx Store + Effects |

Start with services. Move up only when you feel real pain.

### 11.10 Choosing between `effect()` and `computed()`

- **Derived data → `computed()`** (pure, no side effects).
- **Side effect (log, localStorage, call an API) → `effect()`**.
- Don't use `effect()` to copy one signal into another. That's what `computed()` or `linkedSignal()` are for.

---

## 12. Decision Guide: Which Technique Do I Use?

```
Is the other component my direct CHILD (it's in my template)?
├─ Yes
│   ├─ I want to give it data ............................ input()   [prop]="x"
│   ├─ It needs to notify me ............................. output()  (event)="fn($event)"
│   ├─ Both need to stay in sync on one value ............ model()   [(prop)]="x"
│   ├─ I want to give it HTML/templates .................. <ng-content> / ng-template
│   └─ I need to trigger its method (focus, open, reset).. #ref or viewChild()
│
├─ It's my PARENT (compound component) ................... inject(ParentComponent)
│
└─ No (siblings / far apart / unrelated)
    ├─ Shared STATE (cart, user, theme) ................... Service with signals
    ├─ One-off EVENTS (toast, "saved") .................... Service with Subject
    ├─ Data identified by URL (id, filters) ............... Router params / query params
    ├─ Whole feature with complex state ................... NgRx SignalStore
    └─ Siblings, only 2 of them, same parent .............. Via parent: A (event)→parent→[input] B
```

---

## 13. Top 12 Mistakes (and Fixes)

| # | Mistake | Fix |
|---|---|---|
| 1 | `<app-x value="42">` expecting a number | `[value]="42"` |
| 2 | `<button (click)="save">` (no parentheses) | `(click)="save()"` |
| 3 | `[(ngModel)]` fails: *"Can't bind to ngModel"* | Import `FormsModule` in the component's `imports` |
| 4 | `[(value)]` fails on custom component | Needs input `value` **and** output `valueChange` (use `model()`) |
| 5 | Forgetting `()` when reading a signal: `{{ name }}` prints a function | `{{ name() }}` |
| 6 | Mutating an input object/array inside the child | Emit an event; let the parent create a new object |
| 7 | `.push()` on an array signal and nothing updates (worse in v22: every component is `OnPush`) | `list.update(l => [...l, item])` |
| 8 | Using `viewChild` in the constructor → `undefined` | Use after render: handlers, `effect`, `afterNextRender` |
| 9 | Output named like a native event (`click`) | Rename (`itemClicked`) |
| 10 | Subscribing in a component without unsubscribing | `async` pipe, `toSignal`, or `takeUntilDestroyed()` |
| 11 | Using a service for everything, even simple parent→child | Prefer input/output for direct children |
| 12 | Two-way binding everywhere → can't trace changes | One-way `input` + `output` for anything with logic |

---

## 14. Cheat Sheet

```html
{{ value() }}                           <!-- text interpolation -->
[prop]="expr"                           <!-- data INTO a property / child input -->
[attr.name]="expr"                      <!-- attribute -->
[class.name]="bool"   [style.color]="c" <!-- class / style -->
(event)="doIt($event)"                  <!-- listen: DOM event OR child output -->
[(prop)]="x"                            <!-- two-way = [prop] + (propChange) -->
#ref                                    <!-- nickname for element/component -->
@if (cond) { … } @else { … }            <!-- control flow -->
@for (x of xs; track x.id) { … }
<ng-content select="[slot]" />          <!-- content projection -->
```

```ts
// CHILD API
name   = input<string>('');             // optional input
id     = input.required<number>();      // required input
saved  = output<Item>();                // output   → this.saved.emit(item)
value  = model(0);                      // two-way  → input + output "valueChange"

// QUERIES
child  = viewChild.required(Child);     // from my own template
kids   = viewChildren(Child);
slot   = contentChild(Foo);             // from projected content

// SHARING
@Service() class MyService {}           // v22: root-provided service (uses inject())
svc    = inject(MyService);             // DI
lazy   = injectAsync(() => import('./heavy.service'));   // v22: load on demand
state  = signal(0);  d = computed(() => state() * 2);
effect(() => console.log(state()));

// RXJS BRIDGE / ASYNC
toSignal(obs$)   toObservable(sig)
user = httpResource<User>(() => `/api/users/${id()}`);   // stable in v22
```

| I want… | Write |
|---|---|
| Pass a variable to child | `[x]="myVar"` |
| Pass constant text | `x="hello"` |
| Pass number / boolean / object | `[x]="42"`  `[x]="true"`  `[x]="obj"` |
| React to a child event | `(evt)="fn($event)"` |
| Sync a value with child | `[(x)]="myVar"` |
| Disable a button conditionally | `[disabled]="cond"` |
| Run on button click | `(click)="fn()"` |
| Toggle a CSS class | `[class.on]="cond"` |
| Share state across the app | `@Service()` class + signals |

---

## 15. Practice: Drills

### Drill A: `[ ]`, `( )`, `[( )]`, or nothing?

Fill in the correct syntax. (Answers in section 17.)

```html
1.  <app-badge  ___  label="New"  />                          <!-- constant string -->
2.  <app-badge  ___  count="unread"  />                       <!-- unread is a variable -->
3.  <button  ___  = "delete()" >Delete</button>               <!-- run on click -->
4.  <app-search  ___  = "onSearch($event)" />                 <!-- child output "search" -->
5.  <app-slider  ___  = "volume" />                           <!-- child has model "value" -->
6.  <input  ___  = "isLocked" />                              <!-- disable conditionally -->
7.  <div  ___  = "isActive" >                                 <!-- toggle class "active" -->
8.  <app-chart  ___  = "[1,2,3]" />                           <!-- array input "data" -->
9.  <input  ___  = "email" />                                 <!-- ngModel two-way -->
10. <img  ___  = "user.avatar" />                             <!-- set src dynamically -->
```

### Drill B: Spot the bug

```html
(a) <app-user [user]="selectedUser" (select)="current = $event" />
    <!-- Child's class: user = input.required<User>(); selected = output<User>(); -->

(b) <app-toggle [(on)]="isOn" />
    <!-- Child's class: on = input(false); toggled = output<boolean>(); -->

(c) <app-list [items]="'todos'" />
    <!-- Child: items = input.required<Todo[]>(); parent has: todos: Todo[] -->

(d) <app-counter [value]="count" (valueChange)="count++" />
    <!-- Child: value = model(0). What's the logic problem? (count is a plain number) -->
```

### Drill C: Which tool?

Pick one: *input / output / model / viewChild / ng-content / service / router*

1. A "Delete" button inside a row must tell the list to remove it.
2. A navbar shows the logged-in user's name; the profile page can change it.
3. A reusable modal where the caller provides the body and footer buttons.
4. A page that shows product details for `/products/42`.
5. A parent wants to focus an input inside a child `<app-search-box>` on a keyboard shortcut.
6. A star-rating widget whose value the parent can set **and** the widget can change.
7. Shows the number of items in the cart on every page.
8. A table column where the parent decides how each cell looks.

---

## 16. Practice: Build the "TaskFlow" App

A small task manager that makes you use **every** technique. Build it level by level. Each level has **requirements**, **acceptance checks** and **hints**. Don't peek at the solution pieces in section 17 until you've tried.

### Setup

```bash
node -v                              # must be 22 or newer
npm i -g @angular/cli                # v22
ng new taskflow --style=css --ssr=false
cd taskflow
ng serve
```

> The CLI generates `task-item.ts` / class `TaskItem` (no suffix) and `@Service()` services. The reference solutions below use `...Component` names for readability. Name yours either way.

Create components with `ng g c <name>`. Suggested structure:

```
src/app/
  models/task.ts
  services/task.service.ts           (Level 4)
  components/
    task-item/       (dumb)          Level 1
    task-list/       (dumb)          Level 1
    task-form/       (dumb)          Level 2
    rating/          (custom control)Level 5
    card/            (projection)    Level 3
    stats-bar/                       Level 4
  pages/
    board-page/      (smart)
    task-detail-page/                Level 5
```

```ts
// models/task.ts
export interface Task {
  id: number;
  title: string;
  done: boolean;
  priority: number;     // 1 to 5
}
```

---

### Level 1: Data down, events up (`input`, `output`)

**Build:** `TaskItemComponent` and `TaskListComponent` (both dumb), and `BoardPageComponent` (smart, holds `tasks = signal<Task[]>([...])`).

**Requirements**
1. `TaskItemComponent` has `task = input.required<Task>()`, and outputs `toggled` (emits task id) and `deleted` (emits task id).
2. A done task shows with line-through using `[class.done]`.
3. `TaskListComponent` takes `tasks = input.required<Task[]>()`, renders items with `@for` + `track`, and **re-emits** the item events upward (`toggled`, `deleted`).
4. `BoardPageComponent` handles both events and updates the signal **immutably**.

**Acceptance:** Clicking toggle crosses the title out; clicking delete removes the row. Neither Item nor List ever modifies the tasks array.

**Hints:** The event travels two hops: `Item →(toggled)→ List →(toggled)→ Board`. In List's template: `(toggled)="toggled.emit($event)"`.

---

### Level 2: Forms, two-way binding, template refs

**Build:** `TaskFormComponent`.

**Requirements**
1. Has a text input for the title using a **template reference variable** (`#titleBox`) *or* `[(ngModel)]`.
2. A priority `<select>` bound with `[(ngModel)]` or `[value]` + `(change)`.
3. Output `added = output<{ title: string; priority: Task['priority'] }>()`.
4. The Add button is `[disabled]` when the title is empty/whitespace.
5. Pressing Enter (`(keyup.enter)`) also adds. After adding, clear the input.
6. Board page handles `(added)` and generates an `id`.

**Bonus 1:** Rebuild this form with Signal Forms (`form()`, `[formField]`, `required()`), as in section 11.7.

**Bonus 2:** Create `app-priority-filter` with `selected = model<number | null>(null)` and use it as `<app-priority-filter [(selected)]="priorityFilter" />` in the board; show a filtered list via `computed()`.

**Acceptance:** No empty tasks possible. Filter works without the filter component knowing anything about the list.

---

### Level 3: Content projection

**Build:** `CardComponent` with three slots: a title (`select="[card-title]"`), a default body slot, and an actions footer (`select="[card-actions]"`).

**Requirements**
1. Wrap the task form inside a card ("Add Task"), the task list inside a card ("Your Tasks") with a footer button "Clear completed" whose click handler lives in the **board page**.
2. Add an input `collapsed = model(false)` to the card with a header button that toggles it using `[(collapsed)]`. Use `@if (!collapsed())` around the body slot. (Observe: the parent can also collapse it programmatically.)

**Acceptance:** `CardComponent` contains no knowledge of tasks.

**Stretch:** Make `TaskListComponent` accept an optional `<ng-template #row let-task>` via `contentChild` + `ngTemplateOutlet` so the caller can customize how each row looks.

---

### Level 4: Services and shared state

**Build:** `TaskService` (decorated with `@Service()`) that owns the tasks.

**Requirements**
1. Move the `tasks` signal from the board page to the service. Expose `tasks` (readonly), plus methods `add`, `toggle`, `remove`, `clearCompleted`.
2. Add computed signals in the service: `total`, `doneCount`, `remainingCount`.
3. Create `StatsBarComponent` placed in `app.component.html` (the app header, **outside** the board page). It displays "3 / 7 done" by injecting the service. It has **no inputs**.
4. Persist to `localStorage` using an `effect()` inside the service constructor; load initial value from it.
5. Now the board page becomes thin: just inject the service and wire it to the dumb components.

**Acceptance:** Adding a task in the board instantly updates the stats bar even though they share no parent/child relationship.

**Stretch:** Add a `NotificationService` with a `Subject<string>` and a `ToastComponent` in the app shell that displays "Task added" for 2 seconds. Hint: `takeUntilDestroyed`.

---

### Level 5: Router + custom control + imperative API

**Requirements**
1. Routes: `/` → Board, `/tasks/:id` → `TaskDetailPage`, `**` → redirect to `/`.
2. Enable `withComponentInputBinding()`. In `TaskDetailPage` declare `id = input.required<string>()` and compute the task with `computed()` from the service (convert to number).
3. In each task row, an `<a [routerLink]="['/tasks', task().id]">` opens details.
4. Detail page contains a **Signal Form** (`form()` + `[formField]`): title (required), priority via your **custom `RatingComponent` implementing `FormValueControl<number>`**, and a done checkbox. Save calls a service `update()` and navigates back. *(Stretch: also make a second `RatingLegacyComponent` with `ControlValueAccessor` and use it in a classic reactive form, to feel the difference.)*
5. Support `?edit=true` query param bound to `edit = input(false, { transform: booleanAttribute })` which toggles read-only mode.
6. In the Board page, add a keyboard shortcut (`@HostListener('window:keydown./')` or `host: {'(window:keydown./)': ...}`) that focuses the title input of `TaskFormComponent` by calling its exposed `focus()` method via `viewChild.required(TaskFormComponent)`.

**Acceptance:** Refreshing `/tasks/3?edit=true` shows the edit form for task 3. The rating control works with `[formField]`, and clearing the title makes Save impossible.

---

### Level 6: Polish and architecture

1. In Angular 22 every component is already `OnPush`. Hunt for bugs anyway: search your code for `.push(`, `.splice(`, and assignments like `this.task().title = ...`. Fix any mutation with an immutable update. (Bonus: set one component to `ChangeDetectionStrategy.Eager`, mutate something on purpose, and observe the difference.)
2. Add tests for `TaskItemComponent`: set input via `fixture.componentRef.setInput('task', ...)`, click the button, assert the output emitted.
3. Draw (on paper) the data-flow diagram of your app. Label every arrow as *input*, *output*, *model*, *service*, or *router*.
4. Write a one-sentence justification for each place you used a service instead of input/output.

**Self-grading checklist**

- [ ] I never typed `[` or `(` without knowing whether the name on the left belongs to the child/element.
- [ ] No child mutates its inputs.
- [ ] Every component is either clearly *smart* or clearly *dumb*.
- [ ] No manual `subscribe()` without cleanup (or I used `httpResource` / `toSignal`).
- [ ] I never mutate arrays/objects held in signals or received as inputs.
- [ ] I can explain `[(x)]` as `[x]` + `(xChange)` from memory.

---

## 17. Answers to Drills and Reference Solution Pieces

### Drill A answers

| # | Answer | Why |
|---|---|---|
| 1 | *(nothing)* `label="New"` | Constant string needs no brackets |
| 2 | `[count]="unread"` | Pass a variable |
| 3 | `(click)="delete()"` | Run on event |
| 4 | `(search)="onSearch($event)"` | Child output |
| 5 | `[(value)]="volume"` | Child `model` named `value` |
| 6 | `[disabled]="isLocked"` | Boolean property |
| 7 | `[class.active]="isActive"` | Class toggle |
| 8 | `[data]="[1,2,3]"` | Array literal is an expression |
| 9 | `[(ngModel)]="email"` | Two-way form binding |
| 10 | `[src]="user.avatar"` | Dynamic property |

### Drill B answers

- **(a)** The child's output is called `selected`, not `select`. Should be `(selected)="current = $event"`.
- **(b)** `[(on)]` needs an input `on` **and** an output named exactly `onChange`. The child's output is called `toggled`, so the binding fails. Fix: in the child use `on = model(false)` (creates both), or rename the output to `onChange`.
- **(c)** `[items]="'todos'"` passes the **string** `"todos"`. Should be `[items]="todos"`.
- **(d)** It compiles, but `count++` ignores the value the child sent (`$event`), so if the child emits `10` you still only add 1. Use `(valueChange)="count = $event"`, or simply `[(value)]="count"`.

### Drill C answers

1. **output**: the row emits `deleted`; the list/page removes it.
2. **service**: shared state across unrelated components.
3. **ng-content** (with `select` slots).
4. **router** (`withComponentInputBinding` → `id = input()`).
5. **viewChild** (imperative `focus()`).
6. **model** (two-way) or `ControlValueAccessor` if used in forms.
7. **service** (signal `count` computed).
8. **ng-content / ngTemplateOutlet** with a `contentChild` template.

---

### Reference solution pieces for TaskFlow

#### Level 1: `task-item.component.ts`

```ts
import { Component, input, output } from '@angular/core';
import { Task } from '../../models/task';

@Component({
  selector: 'app-task-item',
  template: `
    <li>
      <input type="checkbox" [checked]="task().done" (change)="toggled.emit(task().id)" />
      <span [class.done]="task().done">{{ task().title }} (P{{ task().priority }})</span>
      <button (click)="deleted.emit(task().id)">✕</button>
    </li>
  `,
  styles: `.done { text-decoration: line-through; opacity: .6; }`,
})
export class TaskItemComponent {
  task = input.required<Task>();
  toggled = output<number>();
  deleted = output<number>();
}
```

#### Level 1: `task-list.component.ts`

```ts
@Component({
  selector: 'app-task-list',
  imports: [TaskItemComponent],
  template: `
    <ul>
      @for (t of tasks(); track t.id) {
        <app-task-item [task]="t"
                       (toggled)="toggled.emit($event)"
                       (deleted)="deleted.emit($event)" />
      } @empty {
        <p>No tasks yet 🎉</p>
      }
    </ul>
  `,
})
export class TaskListComponent {
  tasks = input.required<Task[]>();
  toggled = output<number>();
  deleted = output<number>();
}
```

#### Level 2: `task-form.component.ts`

```ts
@Component({
  selector: 'app-task-form',
  imports: [FormsModule],
  template: `
    <input #titleBox
           placeholder="New task…"
           [(ngModel)]="title"
           (keyup.enter)="submit()" />
    <select [(ngModel)]="priority">
      @for (p of [1,2,3,4,5]; track p) { <option [ngValue]="p">P{{ p }}</option> }
    </select>
    <button [disabled]="!title().trim()" (click)="submit()">Add</button>
  `,
})
export class TaskFormComponent {
  added = output<{ title: string; priority: Task['priority'] }>();
  title = signal('');                // [(ngModel)] works with a writable signal
  priority = signal<Task['priority']>(3);
  box = viewChild.required<ElementRef<HTMLInputElement>>('titleBox');

  submit() {
    const t = this.title().trim();
    if (!t) return;
    this.added.emit({ title: t, priority: this.priority() });
    this.title.set('');
    this.focus();
  }
  focus() { this.box().nativeElement.focus(); }   // used in Level 5
}
```

> If your Angular version complains about `[(ngModel)]` with a signal, use a plain property (`title = ''`) or `[ngModel]="title()" (ngModelChange)="title.set($event)"`, which is exactly the desugared form from section 2.5.

#### Level 3: `card.component.ts`

```ts
@Component({
  selector: 'app-card',
  template: `
    <section class="card">
      <header>
        <ng-content select="[card-title]" />
        <button (click)="collapsed.update(c => !c)">{{ collapsed() ? '▸' : '▾' }}</button>
      </header>
      @if (!collapsed()) {
        <div class="body"><ng-content /></div>
        <footer><ng-content select="[card-actions]" /></footer>
      }
    </section>
  `,
})
export class CardComponent {
  collapsed = model(false);
}
```

Usage:

```html
<app-card>
  <h2 card-title>Your Tasks</h2>
  <app-task-list [tasks]="tasks()" (toggled)="svc.toggle($event)" (deleted)="svc.remove($event)" />
  <button card-actions (click)="svc.clearCompleted()">Clear completed</button>
</app-card>
```

#### Level 4: `task.service.ts`

```ts
@Service()
export class TaskService {
  private readonly _tasks = signal<Task[]>(this.load());
  readonly tasks = this._tasks.asReadonly();

  readonly total = computed(() => this._tasks().length);
  readonly doneCount = computed(() => this._tasks().filter(t => t.done).length);
  readonly remainingCount = computed(() => this.total() - this.doneCount());

  constructor() {
    effect(() => {
      try { localStorage.setItem('tasks', JSON.stringify(this._tasks())); } catch {}
    });
  }

  add(title: string, priority: Task['priority']) {
    const id = Date.now();
    this._tasks.update(l => [...l, { id, title, priority, done: false }]);
  }
  toggle(id: number) {
    this._tasks.update(l => l.map(t => t.id === id ? { ...t, done: !t.done } : t));
  }
  remove(id: number) { this._tasks.update(l => l.filter(t => t.id !== id)); }
  clearCompleted()   { this._tasks.update(l => l.filter(t => !t.done)); }
  update(id: number, patch: Partial<Task>) {
    this._tasks.update(l => l.map(t => t.id === id ? { ...t, ...patch } : t));
  }
  getById(id: number) { return this._tasks().find(t => t.id === id); }

  private load(): Task[] {
    try { return JSON.parse(localStorage.getItem('tasks') ?? '[]'); } catch { return []; }
  }
}
```

`stats-bar.component.ts`:

```ts
@Component({
  selector: 'app-stats-bar',
  template: `{{ svc.doneCount() }} / {{ svc.total() }} done ({{ svc.remainingCount() }} left)`,
})
export class StatsBarComponent { svc = inject(TaskService); }
```

#### Level 5: routing and detail page

(Imports you'll need: `form, FormField, required, FormValueControl` from `'@angular/forms/signals'`; `booleanAttribute, computed, effect, inject, input, model, signal` from `'@angular/core'`.)

```ts
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [provideRouter(routes, withComponentInputBinding())],
};

// app.routes.ts
export const routes: Routes = [
  { path: '', component: BoardPageComponent },
  { path: 'tasks/:id', component: TaskDetailPageComponent },
  { path: '**', redirectTo: '' },
];
```

```ts
@Component({
  selector: 'app-task-detail-page',
  imports: [FormField, RatingComponent, RouterLink],
  template: `
    @if (task(); as t) {
      @if (edit()) {
        <input [formField]="taskForm.title" />
        <app-rating [formField]="taskForm.priority" />
        <label><input type="checkbox" [formField]="taskForm.done" /> Done</label>
        <button [disabled]="taskForm().invalid()" (click)="save()">Save</button>
      } @else {
        <h2>{{ t.title }} (P{{ t.priority }})</h2>
        <a [routerLink]="[]" [queryParams]="{ edit: true }">Edit</a>
      }
    } @else { <p>Not found</p> }
  `,
})
export class TaskDetailPageComponent {
  private svc = inject(TaskService);
  private router = inject(Router);

  id   = input.required<string>();                              // from :id
  edit = input(false, { transform: booleanAttribute });         // from ?edit=true

  task = computed(() => this.svc.getById(Number(this.id())));

  model = signal({ title: '', priority: 3, done: false });
  taskForm = form(this.model, path => {
    required(path.title);
  });

  constructor() {
    effect(() => {
      const t = this.task();
      if (t) this.model.set({ title: t.title, priority: t.priority, done: t.done });
    });
  }

  save() {
    if (this.taskForm().invalid()) return;
    this.svc.update(Number(this.id()), this.model());
    this.router.navigate(['/']);
  }
}
```

The rating control for the page:

```ts
@Component({
  selector: 'app-rating',
  template: `
    @for (s of [1, 2, 3, 4, 5]; track s) {
      <button type="button" [disabled]="disabled()" (click)="value.set(s)">
        {{ s <= value() ? '★' : '☆' }}
      </button>
    }
  `,
})
export class RatingComponent implements FormValueControl<number> {
  value = model(1);
  disabled = input(false);
}
```

Board page keyboard shortcut:

```ts
@Component({
  ...
  host: { '(window:keydown./)': 'onSlash($event)' },
})
export class BoardPageComponent {
  svc = inject(TaskService);
  form = viewChild.required(TaskFormComponent);

  onSlash(e: Event) {
    e.preventDefault();
    this.form().focus();
  }
}
```

#### Level 6: unit test example

```ts
it('emits toggled with the task id', () => {
  const fixture = TestBed.createComponent(TaskItemComponent);
  fixture.componentRef.setInput('task', { id: 7, title: 'x', done: false, priority: 2 });
  fixture.detectChanges();

  let received: number | undefined;
  fixture.componentInstance.toggled.subscribe(v => (received = v));

  fixture.nativeElement.querySelector('input[type=checkbox]').click();
  expect(received).toBe(7);
});
```

---

## Final Words: The Three-Sentence Summary

1. **`[ ]` puts data into something; `( )` listens for something coming out of it; `[( )]` does both and is just `[x]` + `(xChange)`.**
2. **Between parent and child: `input` down, `output` up, `model` for two-way, `ng-content` for markup, `viewChild` for commands.**
3. **Between everything else: `@Service()` classes with signals for state, Subjects for events, resources for fetched data, the router for URL-driven data.**

When you're stuck, ask: *"Who owns this data, and who is allowed to change it?"* The owner holds the state; everyone else receives it through an input/service and requests changes through an output/method. Do that consistently and your Angular apps will stay simple as they grow.

Happy building! 🚀
