[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Instance Methods

> **This is not the recommended approach.**
>
> The recommended way to use the reactive system is through the **global shortcut functions** — `computed()`, `watch()`, `batch()`, `notify()`, `toRaw()` — which are cleaner, more composable, and consistent with how all documentation examples are written.
>
> Instance methods exist because they are part of the library and some developers prefer an object-oriented style where everything lives on the state object itself. If you prefer this approach, it works — but for modern, idiomatic DOM Helpers code, use the standalone functions.

---

## What are instance methods?

Every reactive object created with `state()` automatically gets a set of **instance methods** — methods you can call directly on the state object. They all start with `$` to distinguish them from your own data properties.

---

## Recommended vs instance-method comparison

| What you want to do | Recommended (global) | Instance method (alternative) |
|---|---|---|
| Add a computed property | `computed(app, { key: fn })` | `app.$computed('key', fn)` |
| Watch a property | `watch(app, 'key', fn)` | `app.$watch('key', fn)` |
| Batch multiple changes | `batch(() => { ... })` | `app.$batch(function() { ... })` |
| Manually trigger effects | `notify(app, 'key')` | `app.$notify('key')` |
| Get the raw unwrapped object | `toRaw(app)` | `app.$raw` |
| Update state + DOM together | — | `app.$update({})` (instance-only) |
| Functional updates | — | `app.$set({})` (instance-only) |
| Bind state to DOM elements | — | `app.$bind({})` (instance-only) |

`$update`, `$set`, and `$bind` have no direct global equivalent — they are instance-only features.

---

## Quick reference

| Method | What it does |
|--------|-------------|
| `.$computed(key, fn)` | Add a computed property to this state object |
| `.$watch(keyOrFn, callback)` | Watch a property for changes on this state object |
| `.$batch(fn)` | Group changes into one update, `this` = state |
| `.$update(updates)` | Update state + DOM together in one call |
| `.$set(updates)` | Functional updates using transformer functions |
| `.$bind(bindings)` | Connect state properties to DOM elements |
| `.$notify(key?)` | Manually trigger effects watching a property |
| `.$raw` | Access the original unwrapped plain object |

---

## $computed() and $watch()

The recommended alternatives are the global `computed()` and `watch()` functions.

```javascript
const app = state({ price: 100, taxRate: 0.1 });

// ✅ Recommended — global functions
const derived = computed(app, {
  total: function() { return this.price * (1 + this.taxRate); }
});

watch(app, 'price', (newVal, oldVal) => {
  console.log(`Price: ${oldVal} → ${newVal}`);
});
```

```javascript
// 🔵 Instance method alternative — works, but less idiomatic
app.$computed('total', function() {
  return this.price * (1 + this.taxRate);
});

app.$watch('price', (newVal, oldVal) => {
  console.log(`Price: ${oldVal} → ${newVal}`);
});
```

---

## $batch() — Group changes

The recommended alternative is the global `batch()` function.

```javascript
// ✅ Recommended — global batch()
batch(() => {
  app.firstName = 'Bob';
  app.lastName = 'Jones';
  app.age = 31;
});
```

```javascript
// 🔵 Instance method alternative
// Inside $batch(), "this" refers to the state object
app.$batch(function() {
  this.firstName = 'Bob';
  this.lastName = 'Jones';
  this.age = 31;
});
// Effects run once with all three changes applied
```

### Why use it?

Without batching, each assignment triggers effects immediately:

```javascript
const app = state({ firstName: 'Alice', lastName: 'Smith' });

effect(() => {
  console.log(`${app.firstName} ${app.lastName}`);
});

// Without batch — effect runs twice
app.firstName = 'Bob';    // logs "Bob Smith"
app.lastName = 'Jones';   // logs "Bob Jones"

// With batch — effect runs once
batch(() => {
  app.firstName = 'Bob';
  app.lastName = 'Jones';
});
// logs "Bob Jones" — only the final state
```

---

## $update() — Mixed state + DOM updates

> **Instance-only.** There is no global equivalent. This method is unique to the instance API.

### What is it?

`$update()` lets you update **state properties** and **DOM elements** in a single call. It automatically detects whether each key is a state property or a CSS selector.

### Syntax

```javascript
app.$update({
  // State properties (plain keys)
  count: 5,
  name: 'Alice',

  // DOM elements (keys that look like selectors)
  '#counter': { textContent: '5' },
  '.status':  { style: { color: 'green' } }
});
```

### How it detects selectors vs state

```
Key starts with '#' → DOM selector (ID)
Key starts with '.' → DOM selector (class)
Key contains '[' or '>' → DOM selector (complex CSS)
Everything else → State property
```

### Example

```javascript
const app = state({
  userName: 'Alice',
  score: 0
});

// Update state AND DOM in one call
app.$update({
  // State updates
  userName: 'Bob',
  score: 100,

  // DOM updates
  '#playerName':   { textContent: 'Bob' },
  '#scoreDisplay': { textContent: '100' },
  '.status-badge': { style: { color: 'gold' } }
});
```

### Nested state with dot notation

```javascript
const app = state({
  user: { profile: { name: 'Alice' } }
});

app.$update({
  'user.profile.name': 'Bob'  // Updates nested property
});

console.log(app.user.profile.name);  // 'Bob'
```

All updates inside `$update()` are automatically batched — effects run once after all changes.

---

## $set() — Functional updates

> **Instance-only.** There is no global equivalent.

### What is it?

`$set()` lets you update properties using **functions** that receive the current value and return the new value. Useful when the new value depends on the old one.

### Syntax

```javascript
app.$set({
  count: (current) => current + 1,
  name: 'Alice'  // Static values work too
});
```

### Why use it?

```javascript
const app = state({ count: 0, scores: [80, 90] });

// Without $set — must read and write separately
app.count = app.count + 1;

// With $set — function receives current value, cleaner for derived updates
app.$set({
  count: (current) => current + 1,
  name: 'Updated'  // Mix functions and static values
});
```

### Nested properties

```javascript
const app = state({
  settings: { volume: 50 }
});

app.$set({
  'settings.volume': (current) => Math.min(100, current + 10)
});

console.log(app.settings.volume);  // 60
```

Like `$update()`, all changes in `$set()` are automatically batched.

---

## $bind() — Connect state to DOM

> **Instance-only.** There is no global equivalent.
>
> For the recommended declarative DOM-binding approach, use `effect()` with `Elements.update({})` instead.

### What is it?

`$bind()` creates **reactive bindings** between state properties and DOM elements. When state changes, the DOM updates automatically — without needing a manual `effect()`.

### Syntax

```javascript
const cleanup = app.$bind({
  '#elementId': 'propertyName',               // Simple binding
  '#another': function() { return this.x; },  // Computed binding
  '.class': {                                  // Multiple properties
    textContent: 'propertyName',
    style: function() { return { color: this.isActive ? 'green' : 'red' }; }
  }
});

// Remove all bindings when done
cleanup();
```

### Simple string binding

Map a state property directly to an element's text:

```javascript
const app = state({ count: 0 });

app.$bind({
  '#counter': 'count'  // #counter always shows app.count
});

app.count = 42;  // #counter automatically shows "42"
```

### Function binding

Use a function for computed displays:

```javascript
const app = state({ firstName: 'Alice', lastName: 'Smith' });

app.$bind({
  '#fullName': function() {
    return `${this.firstName} ${this.lastName}`;
  }
});

app.firstName = 'Bob';  // #fullName shows "Bob Smith"
```

### Multi-property binding

Bind different DOM properties of the same element:

```javascript
const app = state({ name: 'Alice', isOnline: true });

app.$bind({
  '#userCard': {
    textContent: 'name',
    style: function() {
      return { borderColor: this.isOnline ? 'green' : 'gray' };
    }
  }
});
```

### Nested property binding

```javascript
const app = state({
  user: { profile: { name: 'Alice' } }
});

app.$bind({
  '#userName': 'user.profile.name'
});
```

### $bind() vs effect()

```javascript
// 🔵 $bind() — instance method, declarative bindings object
app.$bind({
  '#counter': 'count',
  '#label':   function() { return `Total: ${this.count}`; }
});

// ✅ Recommended equivalent — effect() + Elements
effect(() => {
  Elements.update({
    counter: { textContent: app.count },
    label:   { textContent: `Total: ${app.count}` }
  });
});
```

Both achieve the same result. The `effect()` approach is preferred because it is explicit, composable, and consistent with the rest of the documentation.

---

## $notify() — Manual trigger

The recommended alternative is the global `notify()` function.

```javascript
// ✅ Recommended
notify(app, 'items');

// 🔵 Instance method alternative
app.$notify('items');
```

### When to use it

Array mutations like `push()` modify the array **in place** without reassigning the property. The reactive system detects reassignment, not in-place mutation:

```javascript
const app = state({ items: [1, 2, 3] });

effect(() => {
  console.log('Items:', app.items.length);
});

// push() mutates in place — effect does NOT re-run automatically
app.items.push(4);

// Manually notify the system that "items" changed
notify(app, 'items');
// Effect re-runs: "Items: 4"
```

> **Note:** If you use `collection()` instead of a plain array in state, you never need `$notify` — collections track mutations automatically.

---

## $raw — Access the original object

The recommended alternative is the global `toRaw()` function.

```javascript
// ✅ Recommended
const original = toRaw(app);

// 🔵 Instance method alternative
const original = app.$raw;
```

### When to use it

```javascript
// Serializing state
const json = JSON.stringify(toRaw(app));

// Sending to an API
fetch('/api/save', {
  method: 'POST',
  body: JSON.stringify(toRaw(app))
});

// Passing to a library that doesn't need reactivity
someLibrary.init(toRaw(app));
```

---

## Common patterns

### Pattern 1: Initialize and bind (instance style)

```javascript
// 🔵 Instance method style
const app = state({ count: 0, name: 'World' });

app.$bind({
  '#count':    'count',
  '#greeting': function() { return `Hello, ${this.name}!`; }
});
```

```javascript
// ✅ Recommended equivalent
const app = state({ count: 0, name: 'World' });

effect(() => {
  Elements.update({
    count:    { textContent: app.count },
    greeting: { textContent: `Hello, ${app.name}!` }
  });
});
```

### Pattern 2: Batch update (instance style)

```javascript
// 🔵 Instance method style
app.$batch(function() {
  this.firstName = data.firstName;
  this.lastName  = data.lastName;
  this.email     = data.email;
  this.role      = data.role;
});
```

```javascript
// ✅ Recommended equivalent
batch(() => {
  app.firstName = data.firstName;
  app.lastName  = data.lastName;
  app.email     = data.email;
  app.role      = data.role;
});
```

### Pattern 3: Functional increment

```javascript
// 🔵 Instance method style
app.$set({
  count:       (c) => c + 1,
  lastUpdated: new Date().toISOString()
});
```

---

## Key takeaways

1. **These are not the recommended patterns.** For modern, idiomatic code use global `computed()`, `watch()`, `batch()`, `notify()`, and `toRaw()` paired with `effect()` and `Elements.update({})`.
2. **`$update`, `$set`, and `$bind` are instance-only** — they have no global equivalent and can still be used when their specific behaviour is useful.
3. **If you prefer the `$` style**, it is fully supported — all instance methods work correctly and are not deprecated.
4. **All `$` methods are non-enumerable** — they will not appear in `Object.keys()` or `for...in` loops, so they never interfere with your data.
5. **`$notify`** is most useful with plain array mutations; prefer `collection()` to avoid needing it.

---

## What's next?

Now let's explore the specialized state factories:
- `ref()` for single values
- `collection()` for reactive lists
- `form()` for form state management
- `asyncState()` for loading/error/data patterns

Let's continue!
