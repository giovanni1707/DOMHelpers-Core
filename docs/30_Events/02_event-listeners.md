[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Event Listeners

This page is a complete guide to attaching and managing event listeners in DOM Helpers. We'll start with the simplest possible case and build up to the full feature set — including options, collections, deduplication, and cleanup.

---

## Starting Simple — One Event on One Element

The simplest thing you can do is attach one listener to one element. The `addEventListener` key goes inside `.update()`, and the value is an array with the event name and the handler:

```js
// The element with id="btn" gets a click listener
Elements.btn.update({
  addEventListener: ['click', handleClick]
});
```

That's it. Under the hood this calls `element.addEventListener('click', handleClick)` — it's the same native API, just written through the update system so it can sit alongside other changes in one place.

---

## Multiple Events on One Element

In plain JavaScript, adding five events to one element means five separate calls:

```js
// Plain JS — five lines of near-identical setup
const card = document.getElementById('product-card');

card.addEventListener('mouseenter', () => card.classList.add('hovered'));
card.addEventListener('mouseleave', () => card.classList.remove('hovered'));
card.addEventListener('click',      () => card.classList.toggle('selected'));
card.addEventListener('focus',      () => card.setAttribute('aria-pressed', 'true'));
card.addEventListener('blur',       () => card.setAttribute('aria-pressed', 'false'));
```

DOM Helpers handles all five in one call using the object format — each key is an event name, each value is the handler:

```js
Elements.card.update({
  addEventListener: {
    // Hover effects
    mouseenter: () => Elements.card.update({ classList: { add: 'hovered' } }),
    mouseleave: () => Elements.card.update({ classList: { remove: 'hovered' } }),

    // Toggle selection on click
    click: () => Elements.card.update({ classList: { toggle: 'selected' } }),

    // Update ARIA state for keyboard users
    focus: (e) => e.target.update({ setAttribute: { 'aria-pressed': 'true' } }),
    blur:  (e) => e.target.update({ setAttribute: { 'aria-pressed': 'false' } })
  }
});
```

One call, five events. When you need to add more later, you just add another key to the same object — no new `addEventListener` calls needed.

---

## Listener Options

Sometimes you need more control over how a listener behaves — when it fires, how the browser handles it, or how long it lives. That's what options are for.

Add a third item to the array format to pass an options object:

```js
Elements.btn.update({
  addEventListener: ['click', handleClick, { once: true }]
});
```

Here are all the options you can use:

| Option | What It Does |
|---|---|
| `once: true` | The listener fires **once**, then automatically removes itself — you don't have to clean it up manually |
| `passive: true` | Tells the browser this listener will **never** call `preventDefault()`, so the browser can scroll and paint without waiting. Use this for scroll and touch events to avoid jank. |
| `capture: true` | Makes the listener fire during the **capture phase** — before the event reaches the target element. Explained in detail on the Propagation page. |
| `signal` | Pass an `AbortSignal` here and the listener will be **automatically removed** when you abort the controller. Great for cleaning up groups of listeners at once. |

### `once: true` — fire and forget

```js
// This handler runs one time when the modal's transition finishes, then disappears
Elements.modal.update({
  addEventListener: ['transitionend', onModalOpened, { once: true }]
});
```

Without `{ once: true }`, you'd have to remove the listener manually from inside the handler. This is much cleaner.

### `passive: true` — smooth scrolling

```js
// Telling the browser "we won't block scrolling here, go ahead and paint freely"
Elements.scrollContainer.update({
  addEventListener: ['scroll', updateScrollPosition, { passive: true }]
});
```

Modern browsers warn you in the console if you attach scroll or touch listeners without `passive: true`, because it can cause noticeable scroll lag. Adding it is a quick win.

### `capture: true` — intercept early

```js
// This listener fires before any handler on child elements
Elements.overlay.update({
  addEventListener: ['click', closeOverlay, { capture: true }]
});
```

This is covered fully in the [Propagation page](./03_event-propagation-bubbling-capturing). For now, just know the option exists.

### `signal` — grouped cleanup with `AbortController`

`AbortController` is a browser API that lets you cancel things — including event listeners. You create a controller, attach its `signal` to your listeners, and when you call `controller.abort()`, all of those listeners are removed at once:

```js
// Create a controller to manage the lifecycle of these listeners
const controller = new AbortController();

Elements.pauseBtn.update({
  addEventListener: ['click', pauseSession, { signal: controller.signal }]
});
Elements.stopBtn.update({
  addEventListener: ['click', stopSession, { signal: controller.signal }]
});

// Later — both listeners removed in one line
controller.abort();
```

This is especially useful when you have a group of listeners that all belong to the same "session" or UI state, and you want to tear them all down cleanly when that state ends.

---

## Removing Listeners

To remove a specific listener, use `removeEventListener` with the same event name and the same handler function:

```js
function handleClick(e) {
  console.log('clicked');
}

// Attach
Elements.btn.update({ addEventListener: ['click', handleClick] });

// Detach later
Elements.btn.update({ removeEventListener: ['click', handleClick] });
```

### The function reference rule

This is the most important thing to understand about removing listeners: **you must pass the exact same function reference** that you used to add it.

Every time you write `() => {}` or `function() {}` inline, JavaScript creates a brand new function object. Two arrow functions that look identical are not equal to each other:

```js
// ❌ This will NOT work
// A new arrow function is created each time — they are different objects
Elements.btn.update({
  addEventListener: ['click', () => console.log('clicked')]
});
Elements.btn.update({
  removeEventListener: ['click', () => console.log('clicked')] // Different function!
});

// ✅ This WILL work
// Both calls use the same variable, which holds the same function object
const logClick = () => console.log('clicked');

Elements.btn.update({ addEventListener:    ['click', logClick] });
Elements.btn.update({ removeEventListener: ['click', logClick] });
```

**Rule of thumb:** if you might need to remove a listener later, always save it in a named variable first.

### Removing a capture-phase listener

If you added a listener with `{ capture: true }`, you also need to include that option when removing it — otherwise the browser won't find the right listener:

```js
function handleCapture(e) { /* ... */ }

// Add with capture
Elements.overlay.update({
  addEventListener: ['click', handleCapture, { capture: true }]
});

// Remove — must include capture: true to match
Elements.overlay.update({
  removeEventListener: ['click', handleCapture, { capture: true }]
});
```

---

## Automatic Deduplication

Here's something DOM Helpers handles for you that plain JavaScript doesn't: **registering the same listener twice does nothing the second time**.

```js
function handleClick(e) {
  console.log('clicked');
}

// Called inside a loop 10 times — but only ONE listener is ever attached
for (let i = 0; i < 10; i++) {
  Elements.btn.update({ addEventListener: ['click', handleClick] });
}

// Click the button → 'clicked' appears once in the console
```

Compare that with plain JavaScript:

```js
// Plain JS — this attaches 10 separate listeners
const btn = document.getElementById('btn');
for (let i = 0; i < 10; i++) {
  btn.addEventListener('click', handleClick);
}
// Click the button → 'clicked' appears 10 times!
```

The library keeps an internal record of every listener it has registered. Before adding a new one, it checks: is this exact handler already attached to this exact event on this exact element? If yes, it skips it silently.

Why does this matter? Because in real apps, setup code runs more than once — inside reactive loops, component re-mounts, or initialization functions that get called multiple times. Deduplication means you never have to worry about accidentally stacking listeners.

---

## Attaching Events to a Group of Elements

One of the most common tasks in JavaScript is doing the same thing across multiple elements — like attaching a click handler to every button on the page. In plain JavaScript you reach for `forEach`:

```js
// Plain JS — loop per event type
const buttons = document.querySelectorAll('.btn');

buttons.forEach(btn => btn.addEventListener('click', handleClick));
buttons.forEach(btn => btn.addEventListener('focus', handleFocus));
buttons.forEach(btn => btn.addEventListener('blur',  handleBlur));
```

In DOM Helpers, `Collections` gives you a group of elements that already behaves like a single unit. Call `.update()` on the collection and it applies to every element inside it:

### By CSS class

```js
// Attach click and focus listeners to every element with class="btn"
Collections.ClassName.btn.update({
  addEventListener: {
    click: handleClick,
    focus: handleFocus,
    blur:  handleBlur
  }
});
```

### By tag name

```js
// Attach an input handler to every <input> element on the page
Collections.TagName.input.update({
  addEventListener: ['input', validateInput]
});
```

### By name attribute

```js
// Attach a change listener to every element with name="option"
Collections.Name.option.update({
  addEventListener: ['change', handleOptionChange]
});
```

### Using Selector

You can also query elements with a CSS selector and attach listeners to the result:

```js
// Find all required fields inside a specific form, then attach listeners to all of them
const fields = Selector.within(Elements.signupForm).queryAll('input[required]');

fields.update({
  addEventListener: {
    blur:  validateField,
    focus: clearFieldError
  }
});
```

The library iterates over every element in the collection internally — you just describe what you want, and it handles the loop. Deduplication applies per element, so calling this multiple times is safe.

---

## Using `e.target.update()` Inside Handlers

When you attach a listener to a collection, every element in that collection gets the same handler function. But when the handler fires, how do you know *which specific element* was interacted with?

That's what `e.target` is for. It's the specific element that triggered the event. And because DOM Helpers enhances it automatically, `.update()` is already available on it:

```js
Collections.ClassName.item.update({
  addEventListener: {
    click: (e) => {
      // e.target = the specific .item that was clicked, not all of them
      // .update() is already available — no querySelector needed
      e.target.update({
        classList: { toggle: 'active' },
        dataset:   { selected: 'true' }
      });
    }
  }
});
```

Each time a user clicks a different item, `e.target` points to that item. You never need to re-query the DOM.

---

## Practical Patterns

### Switch active tab

A common UI pattern: click a tab to make it active, deactivate all others.

```js
function activateTab(e) {
  // Step 1: deactivate every tab
  Collections.ClassName.tab.update({
    classList:    { remove: 'active' },
    setAttribute: { 'aria-selected': 'false' }
  });

  // Step 2: activate only the one that was clicked
  e.target.update({
    classList:    { add: 'active' },
    setAttribute: { 'aria-selected': 'true' }
  });
}

Collections.ClassName.tab.update({
  addEventListener: { click: activateTab }
});
```

### Run something once, then stop

```js
// This handler fires the first time the user clicks "I agree", then never again
Elements.agreeBtn.update({
  addEventListener: ['click', () => {
    Elements.welcomePanel.update({ style: { display: 'none' } });
    Elements.mainContent.update({ style: { display: 'block' } });
  }, { once: true }]
});
```

### Track scroll position without hurting performance

```js
Elements.page.update({
  addEventListener: ['scroll', () => {
    const scrolled = window.scrollY > 80;

    // Add or remove the 'scrolled' class based on position
    Elements.header.update({
      classList: { [scrolled ? 'add' : 'remove']: 'scrolled' }
    });
  }, { passive: true }]  // passive: true tells the browser we won't block scrolling
});
```

### Respond to keyboard shortcuts from anywhere on the page

```js
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape') {
    // Close the modal if it's open
    Elements.modal.update({
      classList:    { remove: 'open' },
      setAttribute: { 'aria-hidden': 'true' }
    });
  }
});
```

---

## Quick Reference

| Key | Value Shape | What Happens |
|---|---|---|
| `addEventListener` | `['click', fn]` | Attaches a single listener |
| `addEventListener` | `['click', fn, { once: true }]` | Attaches a listener with options |
| `addEventListener` | `{ click: fn, focus: fn }` | Attaches multiple listeners at once |
| `removeEventListener` | `['click', fn]` | Removes the listener |
| `removeEventListener` | `['click', fn, { capture: true }]` | Removes a capture-phase listener |

---

## Summary

Here's what to remember from this page:

- **Array format** (`['click', fn]`) — for one event, optionally with options as a third item
- **Object format** (`{ click: fn, focus: fn }`) — for multiple events on the same element, no loop needed
- **Options** — `once`, `passive`, `capture`, and `signal` give you control over listener behavior and lifecycle
- **To remove a listener**, use `removeEventListener` with the same function reference you used to add it
- **Deduplication is automatic** — calling `.update()` with the same listener twice does nothing
- **Collections** — `.update({ addEventListener: ... })` works on a group of elements just like it does on one
- **`e.target.update()`** — the clicked element is always enhanced inside the handler

---

Next: **[Event Propagation, Bubbling & Capturing](./03_event-propagation-bubbling-capturing)**
