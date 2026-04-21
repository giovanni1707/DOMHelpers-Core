# High-Value Features — DOM Helpers JS

This document demonstrates the most impactful real-world advantages of working with **Elements**, **Collections**, **Selector**, and **createElement** — compared to equivalent plain JavaScript. Every example is based on what the library actually does under the hood.

---

## 1. Multiple Events in a Single Declaration

### The plain JavaScript problem

Attaching multiple events to a single element in plain JavaScript means repeating `addEventListener` for every event type. The more events you need, the more lines of near-identical setup code you write.

```js
// Plain JS — 5 separate calls to wire up one card element
const card = document.getElementById('card');

card.addEventListener('mouseenter', () => card.classList.add('hovered'));
card.addEventListener('mouseleave', () => card.classList.remove('hovered'));
card.addEventListener('click',      () => card.classList.toggle('selected'));
card.addEventListener('focus',      () => card.setAttribute('aria-pressed', 'true'));
card.addEventListener('blur',       () => card.setAttribute('aria-pressed', 'false'));
```

### With the library — object format

Pass an object to `addEventListener` inside `.update()`. Every key is an event type, every value is the handler. One declaration, zero repetition:

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

**What makes this possible:** The library's `addEventListener` key accepts three formats:

| Format | Syntax | Use case |
|---|---|---|
| Single event (array) | `['click', handler]` | One event, legacy style |
| Single event with options | `['click', handler, { once: true }]` | One event + options |
| Multiple events (object) | `{ click: fn, mouseenter: fn }` | Multiple events, one declaration |

### Duplicate prevention

Every handler registered through `.update()` is tracked by the library. Calling `.update()` again with the same handler function will **not** register it twice — the library checks its internal tracking map first and skips registration if the handler is already attached.

```js
function handleClick(e) { /* ... */ }

// Called 10 times — only one click listener registered
for (let i = 0; i < 10; i++) {
  Elements.btn.update({ addEventListener: ['click', handleClick] });
}
```

Plain JavaScript gives you no such protection — the same handler would fire 10 times on each click.

### `e.target.update()` inside handlers

Every event handler receives an enhanced event object. `e.target` automatically has `.update()` available inside the handler, so you can modify the clicked element without querying it again:

```js
Elements.btn.update({
  addEventListener: {
    click: (e) => {
      // e.target is the element that received the click
      // .update() is available on it automatically — no getElementById needed
      e.target.update({
        textContent: 'Clicked!',
        classList: { add: 'active' },
        disabled: true
      });
    }
  }
});
```

---

## 2. Bulk Updates with `.update()` on Collections

### The plain JavaScript problem

Updating a set of elements with the same change in plain JavaScript requires a loop. For different updates per element, you need multiple loops or explicit references to each one.

```js
// Plain JS — iterating to update all cards
const cards = document.querySelectorAll('.card');

// Three separate forEach loops for three different updates
cards.forEach(card => card.classList.add('loaded'));
cards.forEach(card => card.style.opacity = '1');
cards.forEach(card => card.setAttribute('data-state', 'ready'));
```

### With the library — one `.update()` call, no loop

`Collections.ClassName.card` gives you the full set of `.card` elements as an enhanced collection. Calling `.update()` once applies the changes to every element in the collection — no iteration in your code at all.

```js
// Library — all three changes in a single call
Collections.ClassName.card.update({
  classList:    { add: 'loaded' },
  style:        { opacity: '1' },
  setAttribute: { 'data-state': 'ready' }
});
```

The library iterates internally and applies every update key to every element — you only describe what you want changed.

### Combining multiple update keys

`.update()` accepts any combination of supported keys at once. The library processes them in a defined order and applies only what actually changed (fine-grained change detection):

```js
// Update a collection of notification elements after an action
Collections.ClassName.notification.update({
  textContent: 'Done!',
  style:       { backgroundColor: '#28a745', color: '#fff' },
  classList:   { add: 'visible', remove: 'pending' },
  dataset:     { state: 'success' }
});
```

### Collections by class, tag, and name

Three access patterns — all return enhanced collections with `.update()`:

```js
// By CSS class
Collections.ClassName.card.update({ style: { border: '1px solid #ccc' } });

// By tag name
Collections.TagName.input.update({ disabled: true });

// By name attribute (useful for form groups)
Collections.Name.option.update({ classList: { remove: 'selected' } });
```

---

## 3. Unified API Across Single Elements and Collections

### Why this matters

In plain JavaScript you think differently depending on whether you have one element or many:

```js
// One element
const btn = document.getElementById('submit');
btn.textContent = 'Send';
btn.style.backgroundColor = '#007bff';

// Many elements — different pattern entirely
const btns = document.querySelectorAll('.btn');
btns.forEach(b => {
  b.textContent = 'Send';
  b.style.backgroundColor = '#007bff';
});
```

### With the library — same API, always

`.update()` works identically on a single element and on a collection. You write the same update object regardless of how many elements you are working with:

```js
// One element — via Elements (ID-based access)
Elements.submitBtn.update({
  textContent:  'Send',
  style:        { backgroundColor: '#007bff' }
});

// Many elements — via Collections (class-based access)
// Identical update object — no loop, no pattern change
Collections.ClassName.btn.update({
  textContent:  'Send',
  style:        { backgroundColor: '#007bff' }
});
```

The library internally decides whether to call the single-element path or the collection path based on what you are working with.

---

## 4. `Elements.update()` — Bulk Updates Across Multiple IDs

When you need to update several distinct elements at once, `Elements.update()` accepts a map of element IDs to update objects. All updates happen in a single call:

```js
// Plain JS — one block of code per element
document.getElementById('title').textContent = 'Welcome back!';
document.getElementById('subtitle').textContent = 'You have 3 new messages';
document.getElementById('avatar').setAttribute('src', user.avatarUrl);
document.getElementById('navBtn').classList.add('active');
document.getElementById('badge').style.display = 'inline';
```

```js
// Library — one structured call for all five elements
Elements.update({
  title:    { textContent: 'Welcome back!' },
  subtitle: { textContent: 'You have 3 new messages' },
  avatar:   { setAttribute: { src: user.avatarUrl } },
  navBtn:   { classList: { add: 'active' } },
  badge:    { style: { display: 'inline' } }
});
```

Missing elements are handled gracefully — if an ID does not exist in the DOM, that entry is skipped with a warning and the rest proceed normally. No try/catch required in your code.

---

## 5. Fine-Grained Change Detection

`.update()` does not blindly set every property on every call. It compares the new value against the previous value it set and against the element's current state. A property is only written to the DOM if something actually changed.

```js
// Called every 100ms from a timer — only re-renders what differs
function tick(currentTime) {
  Elements.clock.update({
    textContent: currentTime,          // written only when time string changes
    style: { color: isLate ? 'red' : 'black' }  // written only when colour changes
  });
}
```

This is powered by a `WeakMap` that stores the last-set value per element per property. No external libraries, no virtual DOM — just a lightweight diff before each write.

---

## 6. Full `.update()` Key Reference

All supported keys in a single example:

```js
Elements.myElement.update({
  // ── Content ──────────────────────────────────────────────
  textContent: 'Hello',
  innerHTML:   '<strong>Hello</strong>',

  // ── Properties ───────────────────────────────────────────
  value:       'input value',
  checked:     true,
  disabled:    false,
  placeholder: 'Enter text…',

  // ── Styles ───────────────────────────────────────────────
  style: {
    color:           'white',
    backgroundColor: '#007bff',
    padding:         '10px 20px',
    borderRadius:    '4px'
  },

  // ── Classes ──────────────────────────────────────────────
  classList: {
    add:     ['btn', 'btn-primary'],   // string or array
    remove:  'old-class',
    toggle:  'active',
    replace: ['btn-secondary', 'btn-primary']  // [from, to]
  },

  // ── Attributes ───────────────────────────────────────────
  setAttribute: {
    'aria-label': 'Submit form',
    'data-id':    '42'
  },
  removeAttribute: ['disabled', 'readonly'],   // string or array

  // ── Dataset ──────────────────────────────────────────────
  dataset: {
    userId: '123',
    role:   'admin'
  },

  // ── Events ───────────────────────────────────────────────
  addEventListener: {
    click:      (e) => e.target.update({ disabled: true }),
    mouseenter: () => Elements.tooltip.update({ style: { display: 'block' } }),
    mouseleave: () => Elements.tooltip.update({ style: { display: 'none' } })
  },
  removeEventListener: ['click', previousHandler],

  // ── DOM Methods ──────────────────────────────────────────
  focus:           [],
  blur:            [],
  scrollIntoView:  [{ behavior: 'smooth', block: 'center' }],
  click:           []
});
```

---

## 7. Enhanced Element Creation with `createElement`

### Plain JavaScript

Creating a button with styles, classes, attributes, and an event handler requires six separate calls minimum:

```js
const btn = document.createElement('button');
btn.textContent = 'Save Changes';
btn.className = 'btn btn-primary';
btn.setAttribute('aria-label', 'Save changes');
btn.dataset.action = 'save';
btn.style.marginTop = '16px';
btn.addEventListener('click', handleSave);
```

### With `createElement`

Everything lives in one configuration object. The returned element already has `.update()` attached:

```js
const btn = createElement('button', {
  textContent:  'Save Changes',
  className:    'btn btn-primary',
  setAttribute: { 'aria-label': 'Save changes' },
  dataset:      { action: 'save' },
  style:        { marginTop: '16px' },
  addEventListener: {
    click: handleSave
  }
});

// The returned element is enhanced — .update() is available immediately
btn.update({ disabled: true });
document.getElementById('form-actions').appendChild(btn);
```

### Building a UI component in one pass

```js
const card = createElement('div', {
  className: 'card',
  dataset:   { id: item.id, category: item.category },
  style:     { padding: '16px', borderRadius: '8px' }
});

const title = createElement('h3', {
  textContent: item.title,
  className:   'card-title'
});

const description = createElement('p', {
  textContent: item.description,
  className:   'card-body'
});

const actionBtn = createElement('button', {
  textContent: 'View Details',
  className:   'btn btn-sm',
  addEventListener: {
    click: () => Router.go(`/item/${item.id}`)
  }
});

card.appendChild(title);
card.appendChild(description);
card.appendChild(actionBtn);
document.getElementById('grid').appendChild(card);
```

---

## 8. Smart Selector System

`Selector` wraps `querySelector` and `querySelectorAll` with caching, enhanced return values, and an async waiting utility.

### Scoped queries

Run a query within the subtree of a specific element instead of searching the entire document:

```js
// Plain JS
const form = document.getElementById('signup-form');
const inputs = form.querySelectorAll('input[required]');
inputs.forEach(inp => inp.classList.add('highlight'));

// Library — Selector.within() returns a scoped helper
const inputs = Selector.within(Elements.signupForm).queryAll('input[required]');
inputs.update({ classList: { add: 'highlight' } });
```

### Result caching

`Selector.query()` and `Selector.queryAll()` cache results. Repeated calls for the same selector return the cached result without touching the DOM again:

```js
// First call — DOM query runs
const nav = Selector.query('#main-nav');

// Second call — cache hit, zero DOM access
const nav2 = Selector.query('#main-nav');
```

The cache is invalidated automatically when the matched elements are removed from the DOM (MutationObserver-backed).

### Waiting for dynamically added elements

`Selector.waitFor()` is an async utility that polls until the element appears — useful for content injected after page load (modals, lazy-loaded sections, third-party widgets):

```js
// Plain JS — manual polling
function waitForModal(selector, timeout = 5000) {
  return new Promise((resolve, reject) => {
    const interval = setInterval(() => {
      const el = document.querySelector(selector);
      if (el) { clearInterval(interval); resolve(el); }
    }, 100);
    setTimeout(() => { clearInterval(interval); reject(); }, timeout);
  });
}

// Library — built in
const modal = await Selector.waitFor('#payment-modal');
modal.update({ classList: { add: 'visible' } });
```

### `.queryAll()` returns an enhanced collection

Every result from `Selector.queryAll()` already has `.update()` attached:

```js
const inputs = Selector.queryAll('input[type="text"]');

inputs.update({
  style:        { border: '2px solid red' },
  setAttribute: { 'aria-invalid': 'true' }
});
```

---

## 9. Elements — Intelligent ID-Based Access with Live Cache

`Elements` is a `Proxy` over a `Map`-backed cache. Every property access (`Elements.myId`) runs through a lookup chain:

1. Check the in-memory `Map` cache — if hit and element is still in the DOM, return it.
2. If miss or element was removed, run `document.getElementById()` and cache the result.
3. If not found, return `null`.

A `MutationObserver` watches `document.body` for added and removed elements and keeps the cache current automatically — you never call `.clearCache()` manually.

```js
// Always returns an enhanced element (or null) — no try/catch needed
const btn = Elements.submitBtn;
if (btn) btn.update({ disabled: true });

// Or use the safe accessor with a fallback
const btn = Elements.get('submitBtn', null);
```

### Waiting for elements not yet in the DOM

```js
// Await up to 5 seconds for the element to appear
const { modal, overlay } = await Elements.waitFor('modal', 'overlay');
modal.update({ classList: { add: 'open' } });
overlay.update({ style: { display: 'block' } });
```

### Guaranteed element access

```js
// Throws immediately if any of the IDs are missing — fail fast
const { form, submitBtn, errorMsg } = Elements.getRequired('form', 'submitBtn', 'errorMsg');
```

---

## 10. Real-World Comparison Summary

| Task | Plain JavaScript | Library |
|---|---|---|
| Attach 4 events to an element | 4 `addEventListener` calls | 1 `.update({ addEventListener: {...} })` |
| Update 10 elements with the same change | `forEach` + multiple property assignments | 1 `.update()` on a collection |
| Update 5 elements by ID with different data | 5 element lookups + assignments each | 1 `Elements.update({ id1: {...}, id2: {...} })` |
| Create an element with styles, classes, data, events | 6–10 separate statements | 1 `createElement('tag', { ... })` |
| Apply styles only when they changed | Manual comparison logic | Built in — `.update()` diffs automatically |
| Wait for a dynamic element | Manual `setInterval` + `setTimeout` | `await Selector.waitFor('#selector')` |
| Prevent duplicate event listeners | Manual `removeEventListener` before re-adding | Built in — library tracks and deduplicates |
| Get a cached element reference | Manual `const cache = {}` object | Built in — `Elements` caches + auto-invalidates |

---

*For the full API, see the [main README](../README.md). For CDN links and loading patterns, see [ALL-CDN-LINKS.md](./ALL-CDN-LINKS.md) and [loading-approaches.md](./loading-approaches.md).*
