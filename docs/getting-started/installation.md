[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Installation

DOM Helpers JS is delivered via CDN — no build tools, no npm, no bundler required. Copy a script tag and you're done.

---

## Recommended — Module Loader (ESM)

The loader is the preferred approach. It resolves dependencies automatically, loads only what the page needs, and keeps your HTML clean.

```html
<script type="module">
  import { load } from 'https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.loader.esm.min.js';
  await load('reactive', 'enhancers');
  // All globals (Elements, Collections, Selector, createElement, ReactiveUtils) are ready here
</script>
```

Load only what you need — the loader handles dependencies for you:

```js
await load('core');                         // Elements, Collections, Selector, createElement
await load('reactive');                     // core + ReactiveUtils
await load('reactive', 'enhancers');        // core + ReactiveUtils + bulk helpers
await load('reactive', 'conditions');       // core + ReactiveUtils + Conditions
await load('native-enhance');               // core + enhancers + native patches
await load('reactive', 'animation', 'form'); // multiple at once — order doesn't matter
```

> The loader auto-injects missing dependencies. You never need to manually list `core` when loading `reactive` — it's handled.

---

## Recommended — Module Loader (Classic Script)

No `type="module"` required. Works in all browsers including IE11 (with a Promise polyfill).

```html
<script src="https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.loader.min.js"></script>
<script>
  DOMHelpersLoader.load('reactive', 'enhancers').then(function() {
    // globals ready
  });
</script>
```

---

## Alternative — Deferred Script Tags

Load specific modules directly with `<script type="module">`. Respects dependency order — always load Core before dependents.

```html
<!-- Core only -->
<script type="module" src="https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.core.esm.min.js"></script>

<!-- Core + Reactive -->
<script type="module" src="https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.core.esm.min.js"></script>
<script type="module" src="https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.reactive.esm.min.js"></script>

<!-- Core + Enhancers + Reactive + Conditions -->
<script type="module" src="https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.core.esm.min.js"></script>
<script type="module" src="https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.enhancers.esm.min.js"></script>
<script type="module" src="https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.reactive.esm.min.js"></script>
<script type="module" src="https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.conditions.esm.min.js"></script>
```

---

## Alternative — Full Bundle

Loads every module in one request. Best when you need most features.

```html
<!-- Deferred (recommended) -->
<script type="module" src="https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.full-spa.esm.min.js"></script>

<!-- Classic (render-blocking) -->
<script src="https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/dom-helpers.full-spa.min.js"></script>
```

**Globals exposed:** `Elements`, `Collections`, `Selector`, `createElement`, `ReactiveUtils`, `ReactiveState`, `StorageUtils`, `Forms`, `Animation`, `AsyncHelpers`, `Router`

---

## Available Modules

| Module | Globals | Standalone? |
|--------|---------|-------------|
| `core` | `Elements`, `Collections`, `Selector`, `createElement` | Yes |
| `reactive` | `ReactiveUtils`, `ReactiveState` | Requires core |
| `enhancers` | *(extends Core objects)* | Requires core |
| `conditions` | *(extends Core objects)* | Requires core |
| `storage` | `StorageUtils` | Yes |
| `form` | `Forms` | Requires core |
| `animation` | `Animation` | Requires core |
| `async` | `AsyncHelpers` | Requires core |
| `native-enhance` | *(patches native DOM methods)* | Requires core + enhancers |
| `spa` | `Router` | Yes |

---

## Load Order Rules

- **Core, Storage, SPA Router** — standalone, no dependencies
- **Enhancers, Reactive, Conditions, Form, Animation, Async** — require Core
- **Load Reactive before Conditions** — Conditions detects Reactive at init time
- **Native Enhance** — requires Core + Enhancers
- **The loader handles all of this automatically** — just name what you want

---

## CDN Providers

All modules are available on both jsDelivr and unpkg:

```
jsDelivr: https://cdn.jsdelivr.net/npm/dom-helpers-js@2.10.0/dist/
unpkg:    https://unpkg.com/dom-helpers-js@2.10.0/dist/
```

See [All CDN Links](/ALL-CDN-LINKS) for the complete file list and [Module Loader](/module-loader) for detailed loader documentation.
