[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Events in DOM Helpers

Every interactive thing on a web page runs on events. A button click, a form submission, a key being pressed, a mouse hovering over a card — all of it flows through the browser's event system.

DOM Helpers doesn't replace that system. It makes it easier to work with — less repetition, cleaner code, and a few genuinely useful extras built in.

---

## Why This Section Exists

If you've written plain JavaScript before, you already know `addEventListener`. You've probably also felt the friction of calling it separately for every event on every element. DOM Helpers gives you a better way — and this section explains exactly how it works, from the basics all the way to real patterns you'll reach for in production.

Here's what's covered across these four pages:

| Page | What You'll Learn |
|---|---|
| **Introduction** *(this page)* | The core idea, the three input formats, and how `e.target.update()` works |
| **Event Listeners** | Attaching, removing, deduplication, options, and events on collections |
| **Propagation, Bubbling & Capturing** | How events travel through the DOM and how to control that journey |
| **Advanced Patterns & Best Practices** | Real-world compositions: forms, keyboards, custom events, cleanup |

---

## The Core Idea

In plain JavaScript, attaching multiple events to one element means one call per event — every time:

```js
// Plain JS — three separate calls for one element
const btn = document.getElementById('btn');

btn.addEventListener('click',      handleClick);
btn.addEventListener('mouseenter', handleHover);
btn.addEventListener('keydown',    handleKey);
```

There's nothing wrong with this. But it gets repetitive fast. If you have five events, you write five lines. If you have ten elements, you write fifty.

With DOM Helpers, all three events live in one place:

```js
// DOM Helpers — one call, all three events
Elements.btn.update({
  addEventListener: {
    click:      handleClick,
    mouseenter: handleHover,
    keydown:    handleKey
  }
});
```

Each key in the object is an event name. Each value is the handler function. One declaration, no repetition.

This same pattern works whether you're targeting a single element (`Elements`), a group of elements (`Collections`), or elements found with a CSS selector (`Selector`). The API is consistent across all three.

---

## Three Ways to Attach a Listener

The `addEventListener` key accepts three different formats. Which one you use depends on how many events you need and whether you need options like `once` or `passive`.

### Format 1 — One event, no options

Pass an array with the event name and the handler function:

```js
Elements.btn.update({
  addEventListener: ['click', handleClick]
});
```

Simple and direct. This is exactly what `element.addEventListener('click', handleClick)` does — just written through the update system.

### Format 2 — One event with options

Add a third item to the array — an options object:

```js
Elements.btn.update({
  addEventListener: ['click', handleClick, { once: true }]
});
// This listener will fire once, then automatically remove itself
```

All native `addEventListener` options are supported: `once`, `passive`, `capture`, and `signal`. You'll see each of these explained in the Event Listeners page.

### Format 3 — Multiple events at once

Pass an object where each key is an event name:

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

Five events, one call. This is the format you'll use most often when building interactive components.

**Quick comparison:**

| Format | When to Use |
|---|---|
| `['click', fn]` | A single event, no options needed |
| `['click', fn, { once: true }]` | A single event where you need options |
| `{ click: fn, focus: fn }` | Two or more events on the same element |

---

## Removing a Listener

To remove a listener you added with `addEventListener`, use `removeEventListener` with the same event name and handler:

```js
function handleClick(e) {
  console.log('clicked!');
}

// Add the listener
Elements.btn.update({ addEventListener: ['click', handleClick] });

// Remove it later
Elements.btn.update({ removeEventListener: ['click', handleClick] });
```

**One important rule:** you must pass the *same function reference* to remove it. If you use an arrow function written inline, you can't remove it — because every time you write `() => {}`, JavaScript creates a brand new function, and the two are never considered equal.

```js
// ❌ This won't work — two different arrow functions
Elements.btn.update({
  addEventListener: ['click', () => doSomething()]
});
Elements.btn.update({
  removeEventListener: ['click', () => doSomething()] // Not the same function!
});

// ✅ This works — same variable, same reference
const handleClick = () => doSomething();

Elements.btn.update({ addEventListener:    ['click', handleClick] });
Elements.btn.update({ removeEventListener: ['click', handleClick] });
```

If you know you'll need to remove a listener, save it in a variable.

---

## Duplicate Prevention

Here's something DOM Helpers handles automatically that plain JavaScript does not: if you attach the same handler to the same event more than once, only one listener is ever registered.

```js
function handleClick(e) {
  console.log('clicked');
}

// Called 10 times — but only ONE click listener is ever attached
for (let i = 0; i < 10; i++) {
  Elements.btn.update({ addEventListener: ['click', handleClick] });
}
// Clicking the button logs 'clicked' exactly once
```

In plain JavaScript, that loop would register ten separate listeners. Each click would fire the handler ten times.

DOM Helpers tracks every listener it registers and skips re-registration if the same function is already attached. This means you can safely call `.update()` inside a loop, a re-render, or any initialization code that might run more than once — without worrying about stacking duplicate listeners.

---

## `e.target.update()` — Modify the Clicked Element Directly

Every event handler you register through `.update()` receives the normal browser `Event` object — but with one helpful extra: `e.target` already has `.update()` available on it.

This means you can modify the element the user interacted with, right inside the handler, without doing a separate DOM query to find it:

```js
Elements.btn.update({
  addEventListener: {
    click: (e) => {
      // e.target is the button that was clicked
      // .update() is already there — no getElementById needed
      e.target.update({
        textContent: 'Clicked!',
        disabled:    true,
        classList:   { add: 'done' }
      });
    }
  }
});
```

This is especially handy when the same handler is attached to many elements, because `e.target` always refers to the *specific element* that was interacted with:

```js
// One listener handles clicks on every .item element in the list
Collections.ClassName.item.update({
  addEventListener: {
    click: (e) => {
      // e.target is whichever item was clicked — not all of them
      e.target.update({
        classList: { toggle: 'selected' }
      });
    }
  }
});
```

You get the right element, already enhanced, with no extra work.

---

## What's Next

- **[Event Listeners](./02_event-listeners)** — a full walkthrough of all the ways to attach and remove listeners, including options, collections, and the internal deduplication model
- **[Propagation, Bubbling & Capturing](./03_event-propagation-bubbling-capturing)** — how events move through the DOM tree, how to intercept them, and how to use that behavior to your advantage
- **[Advanced Patterns & Best Practices](./04_advanced-patterns-and-best-practices)** — real patterns you'll use in production: accessible components, keyboard handling, form validation, custom events, and listener cleanup
