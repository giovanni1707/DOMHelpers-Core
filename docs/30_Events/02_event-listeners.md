[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Event Listeners

This page covers everything about attaching and managing event listeners through DOM Helpers — single events, multiple events, listener options, deduplication, removal, and applying listeners across collections.

---

## The Plain JavaScript Problem

Before seeing the library's approach, it helps to see what you are replacing.

### Attaching multiple events on a single element

```js
// Plain JS — one call per event, every time
const card = document.getElementById('product-card');

card.addEventListener('mouseenter', () => card.classList.add('hovered'));
card.addEventListener('mouseleave', () => card.classList.remove('hovered'));
card.addEventListener('click',      () => card.classList.toggle('selected'));
card.addEventListener('focus',      () => card.setAttribute('aria-pressed', 'true'));
card.addEventListener('blur',       () => card.setAttribute('aria-pressed', 'false'));
```

Five events, five nearly identical lines. Scaling to ten events means ten lines, all carrying the same repetition: `element.addEventListener(...)`.

### Applying listeners to a set of elements

```js
// Plain JS — forEach per event type
const buttons = document.querySelectorAll('.btn');

buttons.forEach(btn => btn.addEventListener('click',  handleClick));
buttons.forEach(btn => btn.addEventListener('focus',  handleFocus));
buttons.forEach(btn => btn.addEventListener('blur',   handleBlur));
```

Three loops for three event types. The more events you need, the noisier this gets.

---

## Attaching Listeners with `.update()`

### Single event — array format

```js
Elements.btn.update({
  addEventListener: ['click', handleClick]
});
```

This directly calls `element.addEventListener('click', handleClick)` under the hood. It is the simplest form and works exactly like the native API.

### Single event with options

Pass options as the third element of the array:

```js
Elements.btn.update({
  addEventListener: ['click', handleClick, { once: true }]
});
```

All native `addEventListener` options are supported:

| Option | Type | Effect |
|---|---|---|
| `once` | boolean | Listener fires once, then auto-removes itself |
| `passive` | boolean | Handler will not call `preventDefault()` — browser optimizes scroll/touch |
| `capture` | boolean | Listener fires during the capture phase (before bubbling) |
| `signal` | AbortSignal | Listener auto-removes when the signal is aborted |

```js
// Once — fires one time, then gone
Elements.modal.update({
  addEventListener: ['transitionend', onModalOpened, { once: true }]
});

// Passive — critical for scroll performance
Elements.scrollContainer.update({
  addEventListener: ['scroll', onScroll, { passive: true }]
});

// Capture — fires before any child handlers
Elements.overlay.update({
  addEventListener: ['click', closeOverlay, { capture: true }]
});

// AbortSignal — controlled lifecycle
const controller = new AbortController();
Elements.btn.update({
  addEventListener: ['click', handleClick, { signal: controller.signal }]
});
// Remove all signal-linked listeners at once:
controller.abort();
```

---

### Multiple events — object format

When you need more than one event type on the same element, use an object:

```js
Elements.card.update({
  addEventListener: {
    mouseenter: () => Elements.card.update({ classList: { add: 'hovered' } }),
    mouseleave: () => Elements.card.update({ classList: { remove: 'hovered' } }),
    click:      () => Elements.card.update({ classList: { toggle: 'selected' } }),
    focus:      (e) => e.target.update({ setAttribute: { 'aria-pressed': 'true' } }),
    blur:       (e) => e.target.update({ setAttribute: { 'aria-pressed': 'false' } })
  }
});
```

One `.update()` call, zero repetition. Every key is an event type, every value is a handler.

The object format does not currently support per-event options (like `once` per individual key). When you need options, use the array format for that specific event alongside a separate `.update()`, or register those events individually.

---

## Removing Listeners

Use `removeEventListener` with the same `[eventType, handler]` array format:

```js
function handleClick(e) {
  console.log('clicked');
}

// Attach
Elements.btn.update({ addEventListener: ['click', handleClick] });

// Detach
Elements.btn.update({ removeEventListener: ['click', handleClick] });
```

### Why the reference must match

JavaScript's `removeEventListener` requires the exact same function reference that was used to add the listener. Arrow functions defined inline create a new reference each time they are evaluated:

```js
// ❌ Cannot remove — new reference every call
Elements.btn.update({
  addEventListener: ['click', () => console.log('clicked')]
});
Elements.btn.update({
  removeEventListener: ['click', () => console.log('clicked')]  // Different reference
});

// ✅ Can remove — same reference stored in variable
const logClick = () => console.log('clicked');
Elements.btn.update({ addEventListener: ['click', logClick] });
Elements.btn.update({ removeEventListener: ['click', logClick] });
```

### Removing with `capture` option

If a listener was added with `{ capture: true }`, it must also be removed with `{ capture: true }`:

```js
function handleCapture(e) { /* ... */ }

Elements.overlay.update({
  addEventListener: ['click', handleCapture, { capture: true }]
});

Elements.overlay.update({
  removeEventListener: ['click', handleCapture, { capture: true }]
});
```

---

## Deduplication — Never Register Twice

Every listener registered through `.update()` is tracked by the library. Calling `.update()` again with the same event type and the same handler reference will not register a second listener — it is silently skipped.

```js
function handleClick(e) { /* ... */ }

// Called 10 times in a loop — only one listener ever attached
for (let i = 0; i < 10; i++) {
  Elements.btn.update({ addEventListener: ['click', handleClick] });
}

// Clicking the button fires handleClick exactly once
```

This is unlike the native `addEventListener`, which happily stacks duplicate listeners:

```js
// Plain JS — attaches 10 separate listeners
const btn = document.getElementById('btn');
for (let i = 0; i < 10; i++) {
  btn.addEventListener('click', handleClick);
}
// Each click fires handleClick 10 times
```

Deduplication means you can safely call `.update()` inside reactive loops, re-renders, or initialization functions without worrying about listener accumulation.

---

## Events on Collections

Every collection in DOM Helpers supports `.update()`, which means you can attach events to a set of elements in a single call — no loop required.

### By class name

```js
// Attach a click listener to every .btn element at once
Collections.ClassName.btn.update({
  addEventListener: {
    click: handleClick,
    focus: handleFocus
  }
});
```

### By tag name

```js
// Attach input handler to every <input> on the page
Collections.TagName.input.update({
  addEventListener: ['input', validateInput]
});
```

### By name attribute

```js
// Attach change listener to every element with name="option"
Collections.Name.option.update({
  addEventListener: ['change', handleOptionChange]
});
```

### Via Selector

```js
// Attach listeners to all required fields inside a specific form
const fields = Selector.within(Elements.signupForm).queryAll('input[required]');
fields.update({
  addEventListener: {
    blur:   validateField,
    focus:  clearFieldError
  }
});
```

### The internal loop

When you call `.update({ addEventListener: ... })` on a collection, the library iterates every element and calls `addEventListener` on each one — but deduplication still applies per element. If you call the same update again, no element gets a second listener.

---

## Events and `e.target.update()`

Every handler function receives a standard DOM `Event` object. The library enhances `e.target` automatically, making `.update()` available on the element that received the event — without any additional lookup:

```js
Collections.ClassName.item.update({
  addEventListener: {
    click: (e) => {
      // e.target = the specific .item element that was clicked
      // .update() is already available — no getElementById, no querySelector
      e.target.update({
        classList: { toggle: 'active' },
        dataset:   { selected: 'true' }
      });
    }
  }
});
```

This is especially powerful when the handler is shared across many elements: each invocation receives the specific element that triggered the event, and you modify it without any DOM query.

---

## Practical Patterns

### Toggle active state on a group of tabs

```js
function activateTab(e) {
  // Remove active from all tabs
  Collections.ClassName.tab.update({
    classList: { remove: 'active' },
    setAttribute: { 'aria-selected': 'false' }
  });

  // Activate only the clicked one
  e.target.update({
    classList: { add: 'active' },
    setAttribute: { 'aria-selected': 'true' }
  });
}

Collections.ClassName.tab.update({
  addEventListener: { click: activateTab }
});
```

### One-time setup confirmation

```js
Elements.agreeBtn.update({
  addEventListener: ['click', () => {
    Elements.setupPanel.update({ style: { display: 'none' } });
    Elements.mainContent.update({ style: { display: 'block' } });
  }, { once: true }]
});
```

### Scroll tracking without jank

```js
Elements.page.update({
  addEventListener: ['scroll', (e) => {
    const y = window.scrollY;
    Elements.header.update({
      classList: { toggle: 'scrolled' },
      dataset:   { scrollY: String(Math.round(y)) }
    });
  }, { passive: true }]
});
```

### Keyboard shortcut on the document

```js
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape') {
    Elements.modal.update({
      classList:   { remove: 'open' },
      setAttribute: { 'aria-hidden': 'true' }
    });
  }
  if (e.key === 'Enter' && e.target.matches('.confirm-btn')) {
    submitForm();
  }
});
```

---

## API Quick Reference

| Key | Value | Result |
|---|---|---|
| `addEventListener` | `['click', fn]` | Attach single listener, no options |
| `addEventListener` | `['click', fn, { once: true }]` | Attach with options |
| `addEventListener` | `{ click: fn, focus: fn }` | Attach multiple listeners at once |
| `removeEventListener` | `['click', fn]` | Remove previously attached listener |
| `removeEventListener` | `['click', fn, { capture: true }]` | Remove capture-phase listener |

---

## Summary

- Use the **array format** (`['event', handler]`) for single events or when you need options
- Use the **object format** (`{ event: handler }`) for multiple events — cleaner and zero repetition
- **Store named references** for any handler you may need to remove later
- **Deduplication is automatic** — the same handler on the same event type will not register twice
- **Collections work identically** — `.update({ addEventListener: ... })` applies to every element in the set
- **`e.target.update()`** lets you modify the event's target element without a DOM query

---

Next: **[Event Propagation, Bubbling & Capturing](./03_event-propagation-bubbling-capturing)**
