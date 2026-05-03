[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Events in DOM Helpers

Events are the heartbeat of any interactive web page. Every click, every keypress, every form submission — all of it flows through the browser's event system. DOM Helpers gives you a cleaner, more expressive way to work with that system without sacrificing any of its power.

---

## What This Section Covers

This guide is a dedicated deep dive into how DOM Helpers handles events across all three entry points — `Elements`, `Collections`, and `Selector`. It covers:

| Topic | What You'll Learn |
|---|---|
| **Event Listeners** | How to attach, remove, and deduplicate listeners through `.update()` |
| **Multiple Events** | Object format for wiring up many event types in one declaration |
| **Event Propagation** | How bubbling and capturing flow through the DOM tree |
| **Controlling Propagation** | `stopPropagation`, `stopImmediatePropagation`, and `preventDefault` |
| **Delegation** | Listening on a parent and filtering by target |
| **`e.target.update()`** | The enhanced event object that lets you modify clicked elements inline |
| **Once and Passive** | Listener options for auto-cleanup and scroll performance |
| **Real-World Patterns** | Form validation, interactive cards, keyboard handling, and more |

---

## The Core Idea

In plain JavaScript, every event binding is a separate imperative call. Three events on one element means three lines of near-identical boilerplate:

```js
const btn = document.getElementById('btn');
btn.addEventListener('click',      handleClick);
btn.addEventListener('mouseenter', handleHover);
btn.addEventListener('keydown',    handleKey);
```

In DOM Helpers, all three live in a single `.update()` declaration:

```js
Elements.btn.update({
  addEventListener: {
    click:      handleClick,
    mouseenter: handleHover,
    keydown:    handleKey
  }
});
```

The same object-driven pattern applies everywhere: single elements via `Elements`, matched sets via `Collections`, and CSS-selector results via `Selector`. You write the same shape of code regardless of how many elements you are working with.

---

## Three Input Formats for `addEventListener`

The `addEventListener` key inside `.update()` accepts three formats — you choose based on how many events you need and whether you need options:

### Format 1 — Single event, no options

Pass an array of `[eventType, handler]`:

```js
Elements.btn.update({
  addEventListener: ['click', handleClick]
});
```

### Format 2 — Single event with options

Pass an array of `[eventType, handler, options]`:

```js
Elements.btn.update({
  addEventListener: ['click', handleClick, { once: true, passive: false }]
});
```

### Format 3 — Multiple events at once

Pass an object where each key is an event type and each value is a handler:

```js
Elements.card.update({
  addEventListener: {
    click:      handleClick,
    mouseenter: handleMouseEnter,
    mouseleave: handleMouseLeave,
    focus:      handleFocus,
    blur:       handleBlur
  }
});
```

| Format | Syntax | Best For |
|---|---|---|
| Array (no options) | `['click', fn]` | Simple, one-off bindings |
| Array (with options) | `['click', fn, { once: true }]` | Listener options needed |
| Object | `{ click: fn, mouseenter: fn }` | Multiple events on one element |

---

## `removeEventListener`

Remove a previously registered listener by passing the same format used to add it:

```js
function handleClick(e) { /* ... */ }

// Add
Elements.btn.update({ addEventListener: ['click', handleClick] });

// Remove later
Elements.btn.update({ removeEventListener: ['click', handleClick] });
```

The same function reference must be used — anonymous functions cannot be removed because each arrow function expression creates a new reference.

```js
// ❌ This cannot be cleaned up — no reference to remove
Elements.btn.update({
  addEventListener: ['click', () => doSomething()]
});

// ✅ Named reference — removable
function handleClick() { doSomething(); }

Elements.btn.update({ addEventListener: ['click', handleClick] });
// ... later ...
Elements.btn.update({ removeEventListener: ['click', handleClick] });
```

---

## Duplicate Prevention — Built In

A subtle but important guarantee: registering the same handler function for the same event type more than once will **not** attach it twice. The library tracks every registered listener internally and skips re-registration if a match is found.

```js
function handleClick(e) { /* ... */ }

// Called 5 times — only one click listener ever attached
for (let i = 0; i < 5; i++) {
  Elements.btn.update({ addEventListener: ['click', handleClick] });
}
```

In plain JavaScript this would attach five listeners, causing the handler to fire five times on each click. DOM Helpers silently deduplicates, so your setup code can run repeatedly without worry.

---

## `e.target.update()` — The Enhanced Event Object

Every handler registered through `.update()` receives a standard `Event` object — with one enhancement: `e.target` automatically has `.update()` available on it. You can modify the element that received the event directly, without querying the DOM again.

```js
Elements.btn.update({
  addEventListener: {
    click: (e) => {
      // No getElementById, no querySelector — the target is already enhanced
      e.target.update({
        textContent: 'Clicked!',
        disabled:    true,
        classList:   { add: 'done' }
      });
    }
  }
});
```

This is particularly useful inside delegated listeners where the target element is not known ahead of time:

```js
Elements.list.update({
  addEventListener: {
    click: (e) => {
      // e.target is whichever list item was clicked
      e.target.update({
        classList: { toggle: 'selected' },
        dataset:   { selected: 'true' }
      });
    }
  }
});
```

---

## What's Next

The following pages walk through each dimension of event handling in detail:

- **[Event Listeners](./02_event-listeners)** — full coverage of attaching, removing, deduplication, options (`once`, `passive`, `capture`), and events on collections
- **[Event Propagation, Bubbling & Capturing](./03_event-propagation-bubbling-capturing)** — how events travel through the DOM, how to control that flow, and event delegation patterns
- **[Advanced Patterns & Best Practices](./04_advanced-patterns-and-best-practices)** — real-world compositions, keyboard handling, accessible interactions, and cleanup strategies
