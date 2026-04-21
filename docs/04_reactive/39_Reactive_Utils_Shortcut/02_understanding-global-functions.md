[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Understanding Global Functions

## Your choice: namespace or global style

Before diving in, one thing to understand clearly: **you have two equivalent ways to call every reactive function**, and you can pick whichever feels right for your project.

```javascript
// Option A — Namespace style (ReactiveUtils prefix)
const user = ReactiveUtils.state({ name: 'Alice' });
ReactiveUtils.effect(() => console.log(user.name));
ReactiveUtils.batch(() => { user.name = 'Bob'; });

// Option B — Global style (Standalone API loaded)
const user = state({ name: 'Alice' });
effect(() => console.log(user.name));
batch(() => { user.name = 'Bob'; });
```

Both options call the **exact same functions** and produce the **exact same result**. There is no performance difference. There is no behavior difference. It is purely a stylistic choice.

| Style | When to prefer it |
|-------|------------------|
| `ReactiveUtils.state()` | Large codebases, teams that value explicitness, avoiding global name collisions |
| `state()` (global) | Frequent reactive code, prototyping, cleaner scripts, familiar feel |

> **Note:** Throughout the rest of the reactive documentation, **global style is used by default** — `state()`, `effect()`, `batch()`, etc. — because it's shorter and easier to read in examples. If you prefer the namespace style, just prepend `ReactiveUtils.` to any function name and it works identically.

---

## What is a "global function"?

A **global function** is a function you can call from anywhere in your JavaScript code without importing it or reaching into an object.

You've already used global functions your entire JavaScript career:

```javascript
// These are all global functions built into JavaScript:
console.log('hello');
setTimeout(() => {}, 1000);
parseInt('42');
JSON.stringify({ a: 1 });
```

Notice: no prefix, no import, no `something.methodName()`. Just the function name, called directly.

The Standalone API makes DOMHelpers reactive functions work the **same way**.

---

## The two calling styles side by side

Once you load the Standalone API, every reactive function is available both ways:

```javascript
// Style 1: Namespace style (always works)
const user = state({ name: 'Alice' });
effect(() => console.log(user.name));

// Style 2: Global style (works after loading shortcut)
const user = state({ name: 'Alice' });
effect(() => console.log(user.name));
```

**These two blocks are identical.** The output is the same. The behavior is the same.

---

## How the Standalone API creates global functions

Here's the mechanic (simplified):

```javascript
// ReactiveUtils shortcut API (part of the reactive module):
global.state  = state;
global.effect = effect;
global.batch  = batch;
global.watch  = watch;
// ... and so on for every function
```

`global` is the window object in a browser. So `global.state = ReactiveUtils.state` is the same as `window.state = ReactiveUtils.state`.

After this runs, `state` becomes a globally available name — just like `console` or `setTimeout`.

---

## "Always available" vs "conditionally available"

Not every function is unconditionally assigned. The Standalone API checks whether advanced modules were loaded before exposing their functions:

### Always available (core modules)

```javascript
// These are always set — no checks needed:
global.state    = state;
global.effect   = effect;
global.batch    = batch;
global.computed = computed;
global.watch    = watch;
global.ref      = ref;
global.refs     = refs;
```

These come from the core reactive module (Module 01), which is always required.

### Conditionally available (optional modules)

```javascript
// These are only set IF the enhanced module is loaded:
if (safeEffect) {
  global.safeEffect = safeEffect;
}

if (asyncEffect) {
  global.asyncEffect = asyncEffect;
}

if (collector) {
  global.collector = collector;
}
```

**Why the check?** Because optional modules (cleanup, enhancements, storage) might not be included in every project. The check prevents assigning `undefined` to a global name.

---

## The "always available" diagram

```
Required modules loaded → Always available as globals:
┌─────────────────────────────────────┐
│ state()       ← state │
│ effect()      ← effect│
│ batch()       ← batch │
│ computed()    ← ...                 │
│ watch()       ← ...                 │
│ effects()     ← ...                 │
│ ref()         ← ...                 │
│ refs()        ← ...                 │
│ store()       ← ...                 │
│ component()   ← ...                 │
│ reactive()    ← ...                 │
│ bindings()    ← ...                 │
│ isReactive()  ← ...                 │
│ toRaw()       ← ...                 │
│ notify()      ← ...                 │
│ pause()       ← ...                 │
│ resume()      ← ...                 │
│ untrack()     ← ...                 │
└─────────────────────────────────────┘

Optional modules loaded → Conditionally available:
┌─────────────────────────────────────┐
│ safeEffect()  ← Module 06          │
│ safeWatch()   ← Module 06          │
│ asyncEffect() ← Module 06          │
│ asyncState()  ← Module 06          │
│ ErrorBoundary ← Module 06          │
│ DevTools      ← Module 06          │
│ collector()   ← Module 05          │
│ scope()       ← Module 05          │
│ autoSave()    ← Module 07          │
│ reactiveStorage() ← Module 07      │
│ watchStorage()    ← Module 07      │
└─────────────────────────────────────┘
```

---

## A real example: from namespace to global

Let's take a real piece of reactive code and convert it from namespace style to global style.

### The scenario

A simple counter that automatically logs when the count changes.

### With namespace style

```html
<script>
  const counter = state({ count: 0 });

  effect(() => {
    console.log('Count is now:', counter.count);
  });

  function increment() {
    batch(() => {
      counter.count++;
    });
  }
</script>
```

### With global style (Standalone API loaded)

```html
<script>
  const counter = state({ count: 0 });

  effect(() => {
    console.log('Count is now:', counter.count);
  });

  function increment() {
    batch(() => {
      counter.count++;
    });
  }
</script>
```

**Exactly the same behavior.** The only difference is removing `ReactiveUtils.` from each line.

---

## Understanding the "alias" concept

When you write:
```javascript
global.state = state;
```

You are not **copying** the function. You are creating a **second name for the same function**.

Think of it like a person having a nickname:

```
Alice Johnson ←────────── both names refer to the same person
    Ali       ←───────────────────────────────────────────────
```

Similarly:
```
state  ←── both names point to the same function
     state           ←───────────────────────────────────────
```

Calling either one does the exact same thing. There's one function, two names.

---

## What happens if you call a function that wasn't loaded?

If you try to call `safeEffect()` but Module 06 wasn't loaded, you'll get an error because `safeEffect` was never assigned:

```javascript
// Module 06 NOT loaded:
safeEffect(() => {});
// ❌ ReferenceError: safeEffect is not defined
```

vs.

```javascript
// Module 06 WAS loaded:
safeEffect(() => {});
// ✅ Works perfectly
```

**How to avoid this:** Load the modules you need. If you want `safeEffect`, include `06_dh-reactive-enhancements.js` before the shortcut file.

---

## The "safe overwrite" pattern

For functions that might already exist globally (from Module 08 or other sources), the Standalone API uses a safety check:

```javascript
// Only set if not already defined:
if (typeof global.set === 'undefined' && ReactiveUtils.set) {
  global.set = ReactiveUtils.set;
}
```

This means:
- If `set` already exists globally (e.g., from Module 08), it is **not overwritten**
- If `set` doesn't exist yet, it gets assigned

This prevents accidental conflicts between modules.

---

## Your first standalone reactive script

Here's a complete working example using only global functions:

**HTML:**
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <div>
      <h2 id="greeting">Loading...</h2>
      <p id="status">Please wait</p>
      <input id="name" placeholder="Enter your name" />
      <button id="updateBtn">Update</button>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.loader.min.js"></script>

    <!-- Load modules -->

    <script>
      DOMHelpersLoader.load('reactive').then(function () {
        // 1. Create state (global function — no ReactiveUtils prefix!)

        const app = state({
          userName: '',
          isReady: false,
        });

        // 2. React to changes (global function)
        effect(() => {
          Elements.greeting.update({
            textContent: app.isReady
              ? 'Hello, ' + app.userName + '!'
              : 'Loading...',
          });
          Elements.status.update({
            textContent: app.isReady ? 'Ready' : 'Please wait',
          });
        });

        // 3. Update state on button click
        Elements.updateBtn.addEventListener('click', () => {
          const name = Elements.name.value;
          batch(() => {
             app.userName = name;
            app.isReady = true;         
          });
        });
      });
    </script>
  </body>
</html>

```

**What happens:**
1. Page loads — effect runs immediately, showing "Loading..."
2. User clicks button — `batch()` updates both values at once
3. Effect runs again — shows "Hello, Alice!" and "Ready"

All done with `state()`, `effect()`, and `batch()` — no namespace prefix needed.

---

## The mental shift

**Before the Standalone API**, you mentally say:
> "I need the state function from ReactiveUtils."

**After the Standalone API**, you mentally say:
> "I need the state function."

It's a small shift, but it makes code feel lighter and more natural — especially when you're writing lots of reactive logic in one file.

---

## Key takeaways

1. **Global functions** are callable from anywhere without a prefix — just like `console.log`
2. **Same functions, new names** — the Standalone API creates aliases, not copies
3. **Always vs conditionally available** — core functions are always exposed, optional module functions only if that module was loaded
4. **Safe assignment** — some functions use `typeof x === 'undefined'` checks to avoid overwriting
5. **Load order matters** — the shortcut file must come after all other reactive modules
6. **Nothing changes functionally** — it's pure syntactic convenience

---

## What's next?

Now let's go through each group of functions in detail — what they do, how to use them, and practical examples:

- The core state functions: `state()`, `effect()`, `batch()`, `computed()`, `watch()`
- Refs and collections: `ref()`, `refs()`, `collection()`, `list()`
- Utilities: `isReactive()`, `toRaw()`, `notify()`, `pause()`, `resume()`, `untrack()`

Let's dive into the core functions! 🚀