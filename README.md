# DOMHelpers Core

A modular JavaScript library for building reactive, declarative DOM-driven applications — without a framework, without a build step, without a virtual DOM.

---

## JavaScript is more powerful than you think

JavaScript is often treated as the "basic" option — something you graduate away from once you pick a framework. DOM Helpers is built on the opposite belief: **JavaScript is one of the most powerful and flexible languages available**, and writing it directly — without abstractions in the way — gives you full visibility into what your application is doing.

DOM Helpers embraces this. It lets you write declarative, reactive, modern code while staying in plain JavaScript. Everything you learn using the library deepens your understanding of the language. Everything you know about JavaScript applies directly to the library. There is no gap between the two.

---

## The problem with plain DOM code

Plain JavaScript works. The browser gives you everything you need. For small projects, it's completely sufficient.

The friction starts as things grow — repeated DOM queries for the same element, updates scattered across functions, mixed responsibilities in handlers, no shared pattern for applying multiple changes at once. The code works, but it becomes increasingly difficult to read, maintain, and reason about.

Most solutions to this problem raise the abstraction level until the DOM disappears. You work with components, templates, and compilers instead of elements, properties, and events. That's powerful — but it comes at a cost: a new mental model, a build pipeline, and a layer between your code and the browser.

DOM Helpers takes a different approach.

---

## The bridge between imperative and declarative

DOM Helpers sits at the intersection of two styles that are often treated as opposites.

**Imperative JavaScript** — direct, explicit, procedural — is what most developers start with. You tell the browser exactly what to do, step by step.

**Declarative code** — describing *what* you want rather than *how* to get it — is what makes applications readable and maintainable at scale.

DOM Helpers lets you write declaratively while the browser executes imperatively underneath. The transition is seamless because you never leave JavaScript. The same knowledge that lets you write `element.style.display = 'none'` directly also lets you write:

```js
Elements.update({
  loginBtn:  { style: { display: 'none' } },
  dashboard: { style: { display: 'block' } },
  userName:  { textContent: user.name }
});
```

One call. Multiple elements. A clear declaration of intent. No framework. No compilation. Just JavaScript.

---

## Reactive without leaving the language

State management is where most plain JavaScript projects eventually reach for a framework. DOM Helpers brings reactivity directly to JavaScript — no special syntax, no compiler, no virtual DOM.

```js
const state = Reactive.state({ count: 0, user: null, loading: false });

// This runs immediately, and again whenever count changes
Reactive.effect(() => {
  Elements.counter.update({ textContent: state.count });
});

// Update state — the DOM follows automatically
state.count++;
```

This is not magic. It is a JavaScript `Proxy` tracking property access and scheduling updates. You can understand exactly what is happening. And understanding it means you understand a core JavaScript concept — not just a framework feature.

---

## State-driven DOM rendering — no virtual DOM required

DOM Helpers replaces the virtual DOM with something simpler: a direct mapping from state values to DOM configurations.

```js
Conditions.whenState(() => formStatus.value, {
  idle: {
    '#submitBtn': { disabled: false, textContent: 'Submit' },
    '#spinner':   { style: { display: 'none' } }
  },
  loading: {
    '#submitBtn': { disabled: true, textContent: 'Submitting...' },
    '#spinner':   { style: { display: 'block' } }
  },
  error: {
    '#submitBtn': { disabled: false, textContent: 'Try Again' },
    '#errorMsg':  { style: { display: 'block' } }
  }
});

// Change state — the DOM reconfigures itself
formStatus.value = 'loading';
```

Declare what each state looks like. Change the value. DOM Helpers handles the rest — by calling the same DOM APIs you already know.

---

## Full SPA applications — no special syntax

The SPA Router module enables complete single-page application routing without any compile step or template language.

```js
Router
  .define([
    { path: '/',        view: '#home-template',    title: 'Home' },
    { path: '/profile', view: '#profile-template', title: 'Profile' },
    { path: '*',        view: '#404-template',     title: 'Not Found' }
  ])
  .mount('#app')
  .setTransition('fade')
  .start({ mode: 'history' });

// Auth guard in one line
Router.requireAuth(() => !!StorageUtils.load('token'), '/login');
```

History management, view transitions, navigation guards, route params — all built in. All plain JavaScript.

---

## One consistent pattern for everything

In plain JavaScript, every type of DOM change has a different syntax. Text content, styles, attributes, classes, and events each follow a different pattern. DOM Helpers unifies them.

```js
// Plain JavaScript — four different patterns
element.textContent = 'Hello';
element.classList.add('active');
element.setAttribute('aria-label', 'Close');
element.addEventListener('click', handler);

// DOM Helpers — one pattern for everything
element.update({
  textContent:      'Hello',
  classList:        { add: 'active' },
  setAttribute:     { 'aria-label': 'Close' },
  addEventListener: { click: handler }
});
```

This consistency is not just ergonomic. It makes code readable at a glance — you see what is changing, on what element, in a single structured declaration.

---

## What you learn, you keep

This is the core of the philosophy.

When you learn how `.update()` works, you understand DOM properties. When you learn how `Reactive.state()` works, you understand JavaScript Proxies. When you learn how `Conditions.whenState()` works, you understand state machines applied to the DOM.

None of this knowledge expires. It is not tied to a compiler version, a framework release cycle, or a syntax that only works inside a specific toolchain. It is JavaScript — and it will transfer to every project you work on, with or without DOM Helpers.

---

## Modules

| Module | What it does |
|---|---|
| **Core** | Element access, collections, and selector queries — all with `.update()` |
| **Enhancers** | Bulk updaters, Id/ClassName/TagName shortcuts, global query |
| **Reactive** | State, effects, computed values, watch, forms, collections, storage |
| **Conditions** | State-to-DOM mappings — declarative conditional rendering |
| **Storage Utils** | localStorage / sessionStorage with namespaces and cross-tab sync |
| **Native Enhance** | Patches native DOM methods so existing code gains `.update()` automatically |
| **SPA Router** | Full client-side routing with transitions, guards, and params |
| **DOM Form** | Form values, validation, and submission without boilerplate |
| **Animation** | Awaitable fade, slide, and transform animations, chainable |
| **Async Helpers** | Debounce, throttle, enhanced fetch, parallel and race utilities |

---

## Links

- [GitHub Repository](https://github.com/giovanni1707/DOMHelpers-Core)
- [DOMHelpers Reactive](https://github.com/DOMHelpers-Js/reactive/)
