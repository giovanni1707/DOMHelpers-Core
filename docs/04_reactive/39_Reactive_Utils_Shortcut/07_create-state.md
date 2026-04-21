[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# `createState()` — Reactive State with Automatic DOM Bindings

## Quick Start (30 seconds)

```html
<!-- HTML -->
<span id="counter">0</span>
<span id="label">inactive</span>
```

```javascript
const app = createState(
  { count: 0, active: false },
  {
    '#counter': 'count',
    '#label':   () => app.active ? 'active' : 'inactive'
  }
);

app.count = 5;       // #counter shows "5" — automatically
app.active = true;   // #label shows "active" — automatically
```

State changes → DOM updates. No `effect()` needed. No manual wiring.

---

## What is `createState()`?

`createState()` combines two things into one call:

1. **`state()`** — creates a reactive proxy object
2. **DOM bindings** — declares which DOM elements should reflect which state values

Every time a bound state property changes, the connected DOM elements update automatically — as if you had written a dedicated `effect()` for each one.

### The two-step vs one-step comparison

```javascript
// Without createState() — two steps every time:
const app = state({ count: 0, label: 'off' });

effect(() => {
  Id('counter').update({ textContent: app.count });
  Id('label').update({ textContent: app.label });
});

// With createState() — one call does both:
const app = createState(
  { count: 0, label: 'off' },
  {
    '#counter': 'count',
    '#label':   'label'
  }
);
```

Both are equivalent. `createState()` is the shorthand.

---

## Syntax

```javascript
const myState = createState(initialValues, bindings);

// Namespace equivalent:
const myState = ReactiveUtils.createState(initialValues, bindings);
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `initialValues` | Object | Plain object with starting property values |
| `bindings` | Object | `{ 'css-selector': binding }` — what to update when state changes |

**Returns:** A reactive state proxy — the same kind of object as `state()`. All global reactive functions (`watch()`, `computed()`, `batch()`, etc.) work with it.

---

## Binding Definition Formats

There are three ways to define what each binding does:

### Format 1 — Simple key string

Maps the element's `textContent` directly to a state property by name:

```javascript
const app = createState(
  { score: 0, name: 'Alice' },
  {
    '#score-display': 'score',   // textContent = app.score
    '#name-display':  'name'     // textContent = app.name
  }
);

app.score = 42;   // #score-display shows "42"
app.name = 'Bob'; // #name-display shows "Bob"
```

Use this when you want to display a state value as plain text.

### Format 2 — Arrow function returning a value

Computes a derived value — can reference multiple state properties:

```javascript
const app = createState(
  { firstName: 'Alice', lastName: 'Smith', count: 0 },
  {
    '#full-name':   () => `${app.firstName} ${app.lastName}`,
    '#badge-label': () => app.count > 0 ? `${app.count} items` : 'Empty',
    '#status':      () => app.count === 0 ? 'idle' : 'active'
  }
);

app.firstName = 'Bob';  // #full-name shows "Bob Smith"
app.count = 3;          // #badge-label shows "3 items", #status shows "active"
```

The function is re-run any time any property it reads changes.

### Format 3 — Object with multiple properties

Controls multiple DOM properties on a single element at once:

```javascript
const app = createState(
  { isOnline: false, role: 'guest' },
  {
    '#status-badge': {
      textContent: () => app.isOnline ? 'Online' : 'Offline',
      className:   () => app.isOnline ? 'badge badge-green' : 'badge badge-gray',
      hidden:      () => app.role === 'admin' ? false : true
    },
    '#avatar': {
      src: () => `/avatars/${app.role}.png`,
      'aria-label': () => `${app.role} avatar`
    }
  }
);

app.isOnline = true;  // #status-badge: text → "Online", class → "badge badge-green"
app.role = 'admin';   // #status-badge hidden → false, #avatar src changes
```

Each key in the object maps to a DOM property (same properties as `.update()`).

---

## Selectors

Every key in the bindings object is a CSS selector:

```javascript
const app = createState(
  { count: 0, active: false },
  {
    '#my-id':            'count',       // by ID
    '.my-class':         'count',       // by class (updates ALL matching elements)
    'button':            () => '...',   // by tag
    '[data-role="main"]': () => '...'   // by attribute
  }
);
```

Multiple elements matching the same selector all get updated.

---

## Full Example: User Status Panel

```html
<div id="user-panel">
  <img id="user-avatar" src="" alt="" />
  <h2 id="user-name"></h2>
  <span id="user-role"></span>
  <span id="online-badge"></span>
  <p id="last-seen"></p>
</div>
```

```javascript
const user = createState(
  {
    name: 'Alice',
    role: 'admin',
    isOnline: true,
    avatarUrl: '/avatars/alice.jpg',
    lastSeen: null
  },
  {
    '#user-name': 'name',

    '#user-role': {
      textContent: () => user.role.charAt(0).toUpperCase() + user.role.slice(1),
      className:   () => `role-badge role-${user.role}`
    },

    '#user-avatar': {
      src: 'avatarUrl',
      alt: () => `${user.name}'s avatar`
    },

    '#online-badge': {
      textContent: () => user.isOnline ? 'Online' : 'Offline',
      className:   () => `badge ${user.isOnline ? 'badge-green' : 'badge-gray'}`
    },

    '#last-seen': {
      textContent: () => user.isOnline ? '' : `Last seen: ${user.lastSeen || 'never'}`,
      hidden:      () => user.isOnline
    }
  }
);

// Update state — all bound elements update automatically:
user.isOnline = false;
user.lastSeen = '2 hours ago';
// → #online-badge: "Offline", gray
// → #last-seen: "Last seen: 2 hours ago", visible
```

---

## Full Example: Counter with Derived UI

```html
<span id="count">0</span>
<span id="count-label">items</span>
<button id="reset-btn">Reset</button>
<div id="overflow-warning" hidden></div>
```

```javascript
const cart = createState(
  { count: 0, limit: 10 },
  {
    '#count': 'count',

    '#count-label': () => cart.count === 1 ? 'item' : 'items',

    '#reset-btn': {
      disabled:    () => cart.count === 0,
      textContent: () => cart.count === 0 ? 'Nothing to reset' : `Reset (${cart.count})`
    },

    '#overflow-warning': {
      hidden:      () => cart.count <= cart.limit,
      textContent: () => `Limit is ${cart.limit}. Remove ${cart.count - cart.limit} item(s).`
    }
  }
);

// Use normally:
cart.count++;      // count shows "1", label shows "item", button updates
cart.count += 9;   // count shows "10", warning hides
cart.count++;      // count shows "11", warning appears
```

---

## `createState()` vs `state()` + `effect()`

Use this table to decide which to reach for:

| Situation | Reach for |
|-----------|-----------|
| Simple text display tied directly to state | `createState()` |
| Multiple DOM properties on one element | `createState()` with object format |
| Complex logic across many elements | `effect()` — more flexible |
| Need to conditionally skip updates | `effect()` — can add `if` guards |
| Async DOM updates (`await` inside) | `asyncEffect()` |
| State that maps finite values to UI configurations | `Conditions.whenState()` |
| Want zero manual wiring for straightforward display | `createState()` |

**The core rule:**
```
State value → show it directly in DOM    →  createState()
State value → compute something complex  →  effect()
```

They are fully composable — you can use `createState()` for the static bindings and add `effect()` for the dynamic parts:

```javascript
const app = createState(
  { status: 'idle', progress: 0 },
  {
    '#status-label': 'status'   // simple: just show the value
  }
);

// effect() for the part that needs more logic:
effect(() => {
  Id('progress-bar').update({
    style: { width: `${app.progress}%` },
    'aria-valuenow': app.progress
  });
});
```

---

## The returned object

`createState()` returns a standard reactive state proxy — the same as `state()`. All global reactive functions work with it:

```javascript
const app = createState({ count: 0 }, { '#counter': 'count' });

// All of these work alongside the bindings:
watch(app, { count: (newVal) => console.log('count →', newVal) });
computed(app, { doubled: function() { return this.count * 2; } });
batch(() => { app.count = 0; });
const raw = toRaw(app);
```

The bindings run on top of whatever else you do with the state — they don't replace `watch()`, `computed()`, or effects.

---

## Cleanup

Bindings set up by `createState()` create reactive effects internally. To group them with other effects for cleanup, wrap the entire setup in `scope()` (Module 05):

```javascript
const stopAll = scope((collect) => {
  const app = createState(
    { count: 0 },
    { '#counter': 'count' }
  );

  collect(effect(() => {
    console.log('count changed:', app.count);
  }));
});

// Later — stops all effects and bindings:
stopAll();
```

---

## API Reference

### `createState(initialValues, bindings?)`

| | |
|--|--|
| **Module** | Core (Module 01) |
| **Global** | `createState()` |
| **Namespace** | `ReactiveUtils.createState()` |

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `initialValues` | `Object` | Yes | Plain object — property names become reactive state keys |
| `bindings` | `Object` | No | `{ 'selector': binding }` — see formats below |

**Binding value types:**

| Value | Behavior |
|-------|----------|
| `'propertyName'` | Maps `textContent` to `state.propertyName` |
| `() => expression` | Computes a string/value; sets `textContent` |
| `{ prop: () => val }` | Sets each DOM property independently |

**Returns:** Reactive state proxy (same as `state()`)

---

## Summary

- `createState()` = `state()` + DOM bindings in one call
- Three binding formats: key string, arrow function, or multi-property object
- Selectors support any CSS selector: `#id`, `.class`, `tag`, `[attr]`
- Returns the same reactive proxy as `state()` — all instance methods work
- Composable: add `effect()` or `watch()` alongside bindings for complex logic
- Both `createState()` (global) and `ReactiveUtils.createState()` (namespace) work identically

---

## What's next?

Continue to: [Real-World Examples and API Reference](./06_real-world-examples-and-api-reference.md)
