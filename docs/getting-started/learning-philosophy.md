[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)
[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Why DOM Helpers?

## The honest answer

Plain JavaScript works. The browser gives you `document.getElementById`, `addEventListener`, `classList`, `style`, `localStorage` — stable, well-documented, universally supported. For a small project, that's all you need.

The friction starts when things grow.

Functions get longer. The same `document.getElementById` call appears five times for the same element. State lives in variables scattered across files. When a button should be disabled, you find every place that touches it and update them all manually. When a form submits, a single handler mixes DOM reads, validation logic, API calls, and error display.

This is not a JavaScript problem. It's a structure problem. DOM Helpers is the structure.

---

## What DOM Helpers is not

Before explaining what it is, it helps to be clear about what it isn't.

**It is not a framework.** It does not control how your application is structured. It does not have opinions about components, lifecycle hooks, or templates. It does not own your render cycle.

**It is not a compiler target.** There is no JSX, no special syntax, no build step required. You write JavaScript. The browser runs JavaScript.

**It is not a virtual DOM.** Updates go directly to the real DOM — immediately, predictably, and visibly in DevTools.

**It is not a replacement for the platform.** `querySelector`, `addEventListener`, `fetch`, `localStorage` — these still exist and still work. DOM Helpers adds structure around them, not a wall between you and them.

---

## What DOM Helpers is

DOM Helpers is a collection of ten modules that each solve one specific category of DOM work:

- **Finding elements** — once, with automatic caching
- **Updating elements** — with a single consistent syntax for any property
- **Working with groups** — helpers for collections, not just individual elements
- **Reactive state** — objects that automatically propagate changes
- **State-driven rendering** — map state values to DOM configurations declaratively
- **Browser storage** — persistence without serialization boilerplate
- **Form handling** — read, validate, and submit without manual wiring
- **Animations** — awaitable, composable, directly on elements
- **Async utilities** — debounce, throttle, fetch with timeout and retry
- **Client-side routing** — a full SPA router with guards and transitions

Each module has a clear, limited purpose. Each can be used independently. Together, they give you the structure that plain JavaScript leaves you to build yourself — every time, from scratch, differently.

---

## The learning philosophy

DOM Helpers was built with a specific belief: **you should understand what your code is doing.**

Most frontend frameworks solve the readability problem by raising the abstraction level until you no longer see the DOM at all. You write components, the framework decides when and how to render them. This works — but it comes with a cost. When something goes wrong, you debug the framework. When the framework changes, your knowledge partially expires.

DOM Helpers stays close to the platform. When you use `.update()`, you are setting properties on a real DOM element. When you use `Reactive.state()`, you are creating a proxy that tracks property access. When you use `Conditions.whenState()`, you are declaring what DOM configurations map to what values — and the library applies them by calling the same DOM APIs you already know.

Nothing is hidden. You can read the source. You can understand the mechanism. You can reason about what the browser is doing because you are still working with the browser.

This matters for learning. If you understand what DOM Helpers is doing under the hood, you understand the DOM. That knowledge transfers to any project, any library, any framework. The goal is not to make DOM work disappear — it's to make it manageable while keeping it visible.

---

## The comparison point

Not React. Not Vue. Not Svelte.

Those frameworks solve a different problem: large teams, complex UIs, component reuse across projects, design systems. They are excellent at what they do.

DOM Helpers is for when you do not need a framework. When you want to write JavaScript directly. When you want to add interactivity to an existing page without replacing it. When you want to understand how reactivity works before you trust a compiler to implement it for you. When you want to ship something without a build step.

The comparison point is **plain JavaScript** — what you'd write without any library. DOM Helpers makes that plain JavaScript cleaner, more consistent, and easier to scale.

---

## Why not just use jQuery?

jQuery solved the wrong problem. Its main contribution was cross-browser normalization — something the platform itself solved years ago. `$('.class')` is not meaningfully different from `document.querySelectorAll('.class')` today.

jQuery does not have a reactive state system. It does not have structured form handling. It does not have a routing solution. It does not help you manage what happens when data changes and the DOM needs to reflect that change.

DOM Helpers is not a selector convenience. It is a complete system for building interactive DOM-driven applications — in plain JavaScript, without a framework.

---

## Where to go from here

If this resonates, start here:

- **[What Is DOM Helpers Core](./01_what-is-dom-helpers-core.md)** — a detailed introduction to the library and its design
- **[Why DOM Helpers Exists](./02_why-dom-helpers-exists.md)** — eight concrete problems, shown side-by-side with plain JavaScript
- **[Architecture Overview](./03_architecture-overview.md)** — how the ten modules divide responsibility
- **[Installation](./installation.md)** — get it running in under two minutes
