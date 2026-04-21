[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Reactive With Conditions

## Quick Start (30 seconds)

```javascript
const app = state({ status: 'idle' });

Conditions.whenState(
  () => app.status,
  {
    'idle':     { '#status-panel': { textContent: 'Waiting…',    className: 'panel' } },
    'loading':  { '#status-panel': { textContent: 'Loading…',    className: 'panel loading' } },
    'success':  { '#status-panel': { textContent: 'Done!',       className: 'panel success' } },
    'error':    { '#status-panel': { textContent: 'Failed.',     className: 'panel error' } }
  }
);

app.status = 'loading';
// → #status-panel shows "Loading…" with class "panel loading" — automatically ✨
```

One call. Every state handled. DOM always matches.

---

## What is Conditions?

**Conditions** is a declarative conditional-rendering module. It lets you describe **what the DOM should look like for each possible state value** as a plain object — a lookup table — and automatically applies the right configuration whenever the reactive value changes.

Think of it as replacing a chain of `if/else` blocks with a **table**: each row is one state, each column is one element, and reading across a row shows the complete UI for that state at a glance.

---

## The Problem It Solves

Without Conditions, switching between visual states requires imperative `if/else` logic inside effects:

```javascript
// Imperative — logic buried in branching, hard to read
effect(() => {
  const s = op.status;
  if (s === 'idle') {
    Id('icon').update({ className: 'icon icon-gray' });
    Id('label').update({ textContent: 'Waiting' });
    Id('detail').update({ hidden: true });
    Id('action-btn').update({ textContent: 'Start', disabled: false, className: 'btn btn-primary' });
  } else if (s === 'loading') {
    Id('icon').update({ className: 'icon icon-blue spinning' });
    Id('label').update({ textContent: 'Processing…' });
    Id('detail').update({ hidden: true });
    Id('action-btn').update({ textContent: 'Cancel', disabled: false, className: 'btn btn-secondary' });
  } else if (s === 'success') {
    Id('icon').update({ className: 'icon icon-green' });
    Id('label').update({ textContent: 'Complete' });
    Id('detail').update({ hidden: false, textContent: 'All done.' });
    Id('action-btn').update({ textContent: 'Start Again', disabled: false, className: 'btn btn-primary' });
  } else if (s === 'error') {
    Id('icon').update({ className: 'icon icon-red' });
    Id('label').update({ textContent: 'Failed' });
    Id('detail').update({ hidden: false });
    Id('action-btn').update({ textContent: 'Retry', disabled: false, className: 'btn btn-danger' });
  }
});
```

**What's wrong with this:**
- "What does the UI look like when status is `error`?" requires mentally tracing scattered lines across four branches
- Adding a new element (`#progress`) means touching every branch
- Adding a new status (`'paused'`) means a new dense block
- The logic lives inside an effect — can't glance at the configuration as data

### The Conditions Solution

```javascript
// Declarative — each state is a complete, self-contained block
const op = state({ status: 'idle', error: null });

Conditions.whenState(
  () => op.status,
  {
    'idle': {
      '#icon':       { className: 'icon icon-gray' },
      '#label':      { textContent: 'Waiting' },
      '#detail':     { hidden: true, textContent: '' },
      '#action-btn': { textContent: 'Start', disabled: false, className: 'btn btn-primary' }
    },
    'loading': {
      '#icon':       { className: 'icon icon-blue spinning' },
      '#label':      { textContent: 'Processing…' },
      '#detail':     { hidden: true, textContent: '' },
      '#action-btn': { textContent: 'Cancel', disabled: false, className: 'btn btn-secondary' }
    },
    'success': {
      '#icon':       { className: 'icon icon-green' },
      '#label':      { textContent: 'Complete' },
      '#detail':     { hidden: false, textContent: 'All done.' },
      '#action-btn': { textContent: 'Start Again', disabled: false, className: 'btn btn-primary' }
    },
    'error': {
      '#icon':       { className: 'icon icon-red' },
      '#label':      { textContent: 'Failed' },
      '#detail':     { hidden: false },
      '#action-btn': { textContent: 'Retry', disabled: false, className: 'btn btn-danger' }
    }
  }
);

// Dynamic error text alongside static conditions
effect(() => {
  if (op.status === 'error') {
    Id('detail').update({ textContent: op.error || 'An unexpected error occurred.' });
  }
});

op.status = 'loading';  // All four elements update in one go
op.status = 'error';    // All four elements switch to error layout
```

✅ "What does the error state look like?" — read the `'error'` block, one self-contained object  
✅ Adding a new element = one line per state block  
✅ Adding a new status = one new block, nothing else changes  
✅ No manual function calls — Conditions + Reactive handles everything

---

## Mental Model: A Lookup Table

```
When status is…    The UI looks like:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
'idle'     →  icon: gray,  label: "Waiting",    btn: "Start"
'loading'  →  icon: spin,  label: "Processing…", btn: "Cancel"
'success'  →  icon: green, label: "Complete",   btn: "Start Again"
'error'    →  icon: red,   label: "Failed",     btn: "Retry"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

You write the table. Conditions reads it and applies the right row automatically whenever the value changes.

---

## Syntax

```javascript
Conditions.whenState(
  getValue,   // () => reactive value to watch
  conditions  // { 'matcher': { 'selector': updateObject } }
);
```

### The Conditions Object Structure

```javascript
{
  // Each key is a condition matcher
  'someValue': {
    // Each inner key is a CSS selector
    '#element-id':  { textContent: 'New text', className: 'active' },
    '.badge':       { hidden: false },
    'button':       { disabled: true }
  },
  'otherValue': {
    '#element-id':  { textContent: 'Different text', className: 'inactive' }
  },
  default: {
    // Runs when no other condition matched
    '#element-id':  { textContent: 'Fallback text', className: '' }
  }
}
```

### Selector Types Inside Conditions

Every key inside a condition block is a CSS selector:

| Selector | Matches |
|----------|---------|
| `'#my-id'` | Element with `id="my-id"` |
| `'.my-class'` | All elements with `class="my-class"` |
| `'button'` | All `<button>` elements |
| `'[data-role="admin"]'` | Attribute selector |
| `'body'` | The document body |

---

## How It Works Internally

When Reactive is loaded, `whenState()` wraps everything in an `effect()`:

```
Conditions.whenState(() => app.status, conditions)
                  ↓
effect(() => {
  const value = getValue()          // Reads reactive state — tracked
                  ↓
  For each key in conditions object:
    matchesCondition(value, key)?
                  ↓
    YES → getElements(selector) for each selector in that block
          applyConfig(element, updateObject)
          break — only first match applies
                  ↓
    NO  → try next key
})
                  ↓
app.status changes → effect re-runs → new row matched → DOM updates
```

Only the **first matching condition** in the object applies. Order matters when multiple matchers could match the same value.

---

## Condition Matchers

Every key in the conditions object goes through the **matcher system** — a registry of strategies tried in order. Here is every built-in matcher:

### Exact String Match

```javascript
Conditions.whenState(() => state.mode, {
  'light':  { 'body': { className: 'theme-light' } },
  'dark':   { 'body': { className: 'theme-dark' } },
  'system': { 'body': { className: 'theme-system' } }
});
```

### Boolean Match

```javascript
Conditions.whenState(() => state.isOpen, {
  'true':  { '#drawer': { className: 'drawer open' } },
  'false': { '#drawer': { className: 'drawer closed' } }
});
```

### Truthy / Falsy

```javascript
Conditions.whenState(() => state.user, {
  'truthy': { '#profile': { hidden: false } },   // any truthy value
  'falsy':  { '#profile': { hidden: true } }     // null, undefined, 0, '', false
});
```

### Null / Undefined / Empty

```javascript
Conditions.whenState(() => state.errorMessage, {
  'empty':  { '#error-bar': { hidden: true } },                // null, undefined, or ''
  'truthy': { '#error-bar': { hidden: false } }
});

Conditions.whenState(() => state.data, {
  'null':      { '#results': { hidden: true, textContent: '' } },
  'undefined': { '#results': { hidden: true, textContent: '' } },
  'truthy':    { '#results': { hidden: false } }
});
```

`'empty'` matches: `null`, `undefined`, `''`, `[]`, `{}`, and any other falsy value.

### Numeric Comparisons

```javascript
// Exact number
Conditions.whenState(() => state.score, {
  '0':   { '#rank': { textContent: 'No score yet' } },
  '100': { '#rank': { textContent: 'Perfect!' } }
});

// Range (inclusive)
Conditions.whenState(() => state.score, {
  '0-49':   { '#rank': { className: 'rank bronze' } },
  '50-79':  { '#rank': { className: 'rank silver' } },
  '80-100': { '#rank': { className: 'rank gold' } }
});

// Comparisons
Conditions.whenState(() => state.count, {
  '0':   { '#badge': { hidden: true } },
  '>0':  { '#badge': { hidden: false } },
  '>=10': { '#badge': { className: 'badge badge-overflow', textContent: '9+' } }
});

// All numeric operators:
// '5'        exact equals 5
// '1-10'     range 1 to 10 inclusive
// '>5'       greater than 5
// '>=5'      greater than or equal to 5
// '<5'       less than 5
// '<=5'      less than or equal to 5
```

### String Pattern Matching

```javascript
// includes: — value contains substring
Conditions.whenState(() => state.statusCode, {
  'includes:error': { '#status': { className: 'status-error' } },
  'includes:warn':  { '#status': { className: 'status-warn' } }
});

// startsWith: — value begins with prefix
Conditions.whenState(() => state.statusCode, {
  'startsWith:2': { '#api-status': { className: 'success' } },  // 200, 201, 204…
  'startsWith:4': { '#api-status': { className: 'client-error' } },  // 400, 404…
  'startsWith:5': { '#api-status': { className: 'server-error' } }   // 500, 503…
});

// endsWith: — value ends with suffix
Conditions.whenState(() => state.filename, {
  'endsWith:.jpg':  { '#file-icon': { className: 'icon icon-image' } },
  'endsWith:.pdf':  { '#file-icon': { className: 'icon icon-pdf' } },
  'endsWith:.json': { '#file-icon': { className: 'icon icon-code' } },
  default:          { '#file-icon': { className: 'icon icon-file' } }
});
```

### Regex Matching

```javascript
Conditions.whenState(() => state.email, {
  '/^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$/': {
    '#email-status': { className: 'valid', textContent: '✓' }
  },
  'falsy': {
    '#email-status': { className: '', textContent: '' }
  },
  default: {
    '#email-status': { className: 'invalid', textContent: '✗' }
  }
});
```

Format: `/pattern/flags` — standard regex syntax inside strings.

### The `default` Key

`default` matches any value not explicitly caught by a previous key. Place it last:

```javascript
Conditions.whenState(() => state.tier, {
  'gold':   { '#badge': { className: 'badge gold',   textContent: '★ Gold' } },
  'silver': { '#badge': { className: 'badge silver', textContent: '◆ Silver' } },
  'bronze': { '#badge': { className: 'badge bronze', textContent: '● Bronze' } },
  default:  { '#badge': { className: 'badge',        textContent: 'Member' } }
});
```

---

## Update Object: What Each Selector Can Receive

The update object for each selector supports the same properties as `.update()`:

```javascript
{
  '#my-element': {
    // Content
    textContent: 'Hello',
    innerHTML: '<span>Hi</span>',

    // Visibility
    hidden: true,

    // Class
    className: 'active highlighted',
    classList: { add: 'active', remove: 'inactive' },
    classList: { toggle: 'open' },

    // Style
    style: { color: 'red', opacity: '0.5' },

    // Attributes
    setAttribute: { 'aria-label': 'Close menu', 'data-state': 'open' },
    removeAttribute: ['disabled'],

    // Properties
    disabled: true,
    checked: false,
    value: 'default text',

    // Dataset
    dataset: { userId: '42', role: 'admin' },

    // Events (applied when condition activates)
    addEventListener: {
      click: () => handleClick()
    }
  }
}
```

---

## Examples

### Example 1 — Traffic Light

```javascript
const light = state({ color: 'red' });

Conditions.whenState(
  () => light.color,
  {
    'red': {
      '#traffic-light': { className: 'light light-red' },
      '#light-label':   { textContent: 'STOP' },
      '#light-red':     { style: { opacity: '1' } },
      '#light-yellow':  { style: { opacity: '0.2' } },
      '#light-green':   { style: { opacity: '0.2' } }
    },
    'yellow': {
      '#traffic-light': { className: 'light light-yellow' },
      '#light-label':   { textContent: 'CAUTION' },
      '#light-red':     { style: { opacity: '0.2' } },
      '#light-yellow':  { style: { opacity: '1' } },
      '#light-green':   { style: { opacity: '0.2' } }
    },
    'green': {
      '#traffic-light': { className: 'light light-green' },
      '#light-label':   { textContent: 'GO' },
      '#light-red':     { style: { opacity: '0.2' } },
      '#light-yellow':  { style: { opacity: '0.2' } },
      '#light-green':   { style: { opacity: '1' } }
    }
  }
);

// Cycle automatically
const sequence = ['red', 'green', 'yellow'];
let i = 0;
setInterval(() => { light.color = sequence[++i % sequence.length]; }, 2000);
```

### Example 2 — Async Loading States

```javascript
const data = state({ status: 'idle' });

Conditions.whenState(
  () => data.status,
  {
    'idle': {
      '#loader':       { hidden: true },
      '#content':      { hidden: true },
      '#error-panel':  { hidden: true },
      '#empty-state':  { hidden: false }
    },
    'loading': {
      '#loader':       { hidden: false },
      '#content':      { hidden: true },
      '#error-panel':  { hidden: true },
      '#empty-state':  { hidden: true }
    },
    'success': {
      '#loader':       { hidden: true },
      '#content':      { hidden: false },
      '#error-panel':  { hidden: true },
      '#empty-state':  { hidden: true }
    },
    'error': {
      '#loader':       { hidden: true },
      '#content':      { hidden: true },
      '#error-panel':  { hidden: false },
      '#empty-state':  { hidden: true }
    }
  }
);

async function fetchData() {
  data.status = 'loading';
  try {
    const res = await fetch('/api/data');
    if (!res.ok) throw new Error('Failed');
    data.status = 'success';
  } catch {
    data.status = 'error';
  }
}
```

Four possible states. Every element's visibility is declared per state. No logic, no branching.

### Example 3 — User Role Permissions

```javascript
const auth = state({ role: 'guest' });

Conditions.whenState(
  () => auth.role,
  {
    'guest': {
      '#nav-dashboard': { hidden: true },
      '#nav-settings':  { hidden: true },
      '#nav-admin':     { hidden: true },
      '#login-btn':     { hidden: false },
      '#welcome-msg':   { textContent: 'Please log in to continue' }
    },
    'user': {
      '#nav-dashboard': { hidden: false },
      '#nav-settings':  { hidden: false },
      '#nav-admin':     { hidden: true },
      '#login-btn':     { hidden: true },
      '#welcome-msg':   { textContent: 'Welcome to your dashboard' }
    },
    'admin': {
      '#nav-dashboard': { hidden: false },
      '#nav-settings':  { hidden: false },
      '#nav-admin':     { hidden: false },
      '#login-btn':     { hidden: true },
      '#welcome-msg':   { textContent: 'Welcome, Administrator' }
    }
  }
);

function loginAsUser()  { auth.role = 'user'; }
function loginAsAdmin() { auth.role = 'admin'; }
function logout()       { auth.role = 'guest'; }
```

### Example 4 — Form Submission States

```javascript
const form = state({ status: 'idle' });

Conditions.whenState(
  () => form.status,
  {
    'idle': {
      '#submit-btn': { textContent: 'Submit',       disabled: false, className: 'btn btn-primary' },
      '#spinner':    { hidden: true },
      '#success-msg': { hidden: true },
      '#error-msg':  { hidden: true }
    },
    'submitting': {
      '#submit-btn': { textContent: 'Submitting…',  disabled: true,  className: 'btn btn-primary btn-loading' },
      '#spinner':    { hidden: false },
      '#success-msg': { hidden: true },
      '#error-msg':  { hidden: true }
    },
    'success': {
      '#submit-btn': { textContent: 'Submitted ✓',  disabled: true,  className: 'btn btn-success' },
      '#spinner':    { hidden: true },
      '#success-msg': { hidden: false },
      '#error-msg':  { hidden: true }
    },
    'error': {
      '#submit-btn': { textContent: 'Try Again',    disabled: false, className: 'btn btn-danger' },
      '#spinner':    { hidden: true },
      '#success-msg': { hidden: true },
      '#error-msg':  { hidden: false }
    }
  }
);

Id('my-form').addEventListener('submit', async (e) => {
  e.preventDefault();
  form.status = 'submitting';
  try {
    await submitData();
    form.status = 'success';
  } catch {
    form.status = 'error';
  }
});
```

### Example 5 — Numeric Range: Score Tier

```javascript
const game = state({ score: 0 });

Conditions.whenState(
  () => game.score,
  {
    '0':      { '#rank': { textContent: 'No score yet', className: 'rank' } },
    '1-49':   { '#rank': { textContent: 'Beginner',     className: 'rank bronze' } },
    '50-79':  { '#rank': { textContent: 'Intermediate', className: 'rank silver' } },
    '80-99':  { '#rank': { textContent: 'Advanced',     className: 'rank gold' } },
    '100':    { '#rank': { textContent: 'Perfect!',     className: 'rank diamond' } }
  }
);
```

### Example 6 — Pattern Matching: HTTP Status Codes

```javascript
const api = state({ lastStatus: '' });

Conditions.whenState(
  () => api.lastStatus,
  {
    'startsWith:2': {
      '#api-indicator': { className: 'indicator success', textContent: 'OK' }
    },
    'startsWith:3': {
      '#api-indicator': { className: 'indicator redirect', textContent: 'Redirected' }
    },
    'startsWith:4': {
      '#api-indicator': { className: 'indicator client-error', textContent: 'Client Error' }
    },
    'startsWith:5': {
      '#api-indicator': { className: 'indicator server-error', textContent: 'Server Error' }
    },
    default: {
      '#api-indicator': { className: 'indicator', textContent: '—' }
    }
  }
);
```

---

## Combining Conditions With `effect()`

Conditions handles the **mode switching** (which layout). Effects handle **dynamic values within a mode**. Use both together:

```javascript
const app = state({ status: 'loading', progress: 0, errorMessage: '' });

// Conditions: which panels are visible
Conditions.whenState(
  () => app.status,
  {
    'loading': {
      '#content':      { hidden: true },
      '#loader-panel': { hidden: false },
      '#error-panel':  { hidden: true }
    },
    'loaded': {
      '#content':      { hidden: false },
      '#loader-panel': { hidden: true },
      '#error-panel':  { hidden: true }
    },
    'error': {
      '#content':      { hidden: true },
      '#loader-panel': { hidden: true },
      '#error-panel':  { hidden: false }
    }
  }
);

// effect(): dynamic values that change continuously
effect(() => {
  Id('progress-bar').update({ style: { width: `${app.progress}%` } });
  Id('progress-text').update({ textContent: `${app.progress}%` });
});

effect(() => {
  Id('error-message').update({ textContent: app.errorMessage });
});
```

**The pattern:**
- `Conditions.whenState()` → "which visual mode am I in?"
- `effect()` → "what are the specific values within this mode?"

---

## Multiple Independent `whenState()` Calls

Each `whenState()` is independent. You can have as many as you need, each watching a different part of state:

```javascript
const dashboard = state({
  loadStatus: 'idle',
  authStatus: 'guest',
  sidebarState: 'open',
  theme: 'light'
});

Conditions.whenState(
  () => dashboard.loadStatus,
  {
    'idle':    { '#page-loader': { hidden: true },  '#main-content': { hidden: false }, '#error-banner': { hidden: true } },
    'loading': { '#page-loader': { hidden: false }, '#main-content': { hidden: true },  '#error-banner': { hidden: true } },
    'loaded':  { '#page-loader': { hidden: true },  '#main-content': { hidden: false }, '#error-banner': { hidden: true } },
    'error':   { '#page-loader': { hidden: true },  '#main-content': { hidden: true },  '#error-banner': { hidden: false } }
  }
);

Conditions.whenState(
  () => dashboard.authStatus,
  {
    'guest': { '#user-nav': { hidden: true },  '#admin-nav': { hidden: true },  '#guest-banner': { hidden: false } },
    'user':  { '#user-nav': { hidden: false }, '#admin-nav': { hidden: true },  '#guest-banner': { hidden: true } },
    'admin': { '#user-nav': { hidden: false }, '#admin-nav': { hidden: false }, '#guest-banner': { hidden: true } }
  }
);

Conditions.whenState(
  () => dashboard.sidebarState,
  {
    'open':   { '#sidebar': { className: 'sidebar open' },   '#main': { className: 'main shifted' } },
    'closed': { '#sidebar': { className: 'sidebar closed' }, '#main': { className: 'main' } },
    'mini':   { '#sidebar': { className: 'sidebar mini' },   '#main': { className: 'main shifted-mini' } }
  }
);

Conditions.whenState(
  () => dashboard.theme,
  {
    'light': { 'body': { className: 'theme-light' } },
    'dark':  { 'body': { className: 'theme-dark' } }
  }
);

// Each state variable controls its own slice of the UI independently
dashboard.authStatus  = 'admin';   // Only nav updates
dashboard.sidebarState = 'mini';   // Only sidebar updates
dashboard.theme       = 'dark';    // Only body class updates
```

---

## `Conditions.apply()` — One-Shot, No Reactivity

Apply a condition once for a specific value without setting up a reactive watcher:

```javascript
// Apply once — no reactive tracking, no effect
Conditions.apply('error', {
  '#status-icon': { className: 'icon icon-red' },
  '#status-text': { textContent: 'Error' }
});
```

Use this for initial render or imperative one-off updates.

---

## `Conditions.watch()` — Explicit Reactive Mode

Identical to `whenState()` in reactive mode — explicit alias when you want to be clear about intent:

```javascript
const stopWatching = Conditions.watch(
  () => app.status,
  { /* conditions */ }
);

// Later — stop the reactive watcher
stopWatching();
```

---

## Cleanup — Stopping a `whenState()`

`whenState()` returns the dispose function from the underlying `effect()`. Call it to stop watching:

```javascript
const stopCondition = Conditions.whenState(
  () => modal.isOpen,
  {
    'true':  { '#modal': { hidden: false }, 'body': { className: 'no-scroll' } },
    'false': { '#modal': { hidden: true },  'body': { className: '' } }
  }
);

// Stop watching when the component is removed
function destroyModal() {
  stopCondition();
}
```

---

## Extending: Custom Matchers

Register your own condition matching logic:

```javascript
// Match values in a specific set
Conditions.registerMatcher('oneOf', {
  test: (condition) => condition.startsWith('oneOf:'),
  match: (value, condition) => {
    const options = condition.slice(6).split(',').map(s => s.trim());
    return options.includes(String(value));
  }
});

// Usage
Conditions.whenState(() => state.country, {
  'oneOf:US,CA,MX': { '#region-label': { textContent: 'North America' } },
  'oneOf:GB,FR,DE': { '#region-label': { textContent: 'Europe' } },
  default:          { '#region-label': { textContent: 'Other' } }
});
```

---

## Extending: Custom Property Handlers

Register your own property application logic:

```javascript
// Custom handler for a "pulse" animation
Conditions.registerHandler('pulse', {
  test: (key) => key === 'pulse',
  apply: (element, val) => {
    if (val) {
      element.classList.add('pulse-animation');
      setTimeout(() => element.classList.remove('pulse-animation'), 600);
    }
  }
});

// Usage in conditions
Conditions.whenState(() => state.alert, {
  'true':  { '#notification': { hidden: false, pulse: true, className: 'notification active' } },
  'false': { '#notification': { hidden: true, className: 'notification' } }
});
```

---

## API Summary

| Method | Description |
|--------|-------------|
| `Conditions.whenState(getFn, conditions)` | Primary method — reactive when Reactive is loaded |
| `Conditions.apply(value, conditions)` | One-shot, no reactivity |
| `Conditions.watch(getFn, conditions)` | Explicit reactive mode alias of `whenState` |
| `Conditions.batch(fn)` | Batch DOM updates inside a function |
| `Conditions.registerMatcher(name, { test, match })` | Add a custom condition matcher |
| `Conditions.registerHandler(name, { test, apply })` | Add a custom property handler |
| `Conditions.getMatchers()` | List all registered matcher names |
| `Conditions.getHandlers()` | List all registered handler names |
| `Conditions.hasReactivity` | `true` if Reactive is loaded |
| `Conditions.mode` | `'reactive'` or `'static'` |

### Built-in Condition Matchers

| Key format | Matches when… |
|------------|---------------|
| `'active'` | `value === 'active'` (string equality) |
| `'true'` | `value === true` (boolean) |
| `'false'` | `value === false` (boolean) |
| `'truthy'` | `!!value === true` |
| `'falsy'` | `!value === true` |
| `'null'` | `value === null` |
| `'undefined'` | `value === undefined` |
| `'empty'` | `null`, `undefined`, `''`, `[]`, `{}` |
| `'5'` | `value === 5` (numeric exact) |
| `'1-10'` | `value >= 1 && value <= 10` (range) |
| `'>5'` | `value > 5` |
| `'>=5'` | `value >= 5` |
| `'<5'` | `value < 5` |
| `'<=5'` | `value <= 5` |
| `'includes:text'` | `value.includes('text')` |
| `'startsWith:pre'` | `value.startsWith('pre')` |
| `'endsWith:suf'` | `value.endsWith('suf')` |
| `'/regex/flags'` | regex test against value |
| `'"quoted"'` | `value === 'quoted'` (explicit string) |
| `default` | catch-all, always matches if last |

---

## Summary

- `Conditions.whenState()` maps reactive values to DOM configurations — a **lookup table**, not logic
- Each condition block is **self-contained**: read one block to see the complete UI for that state
- Supports a rich matcher system: strings, booleans, truthy/falsy, null checks, numeric ranges and comparisons, string patterns, and regex
- Selectors inside conditions can be `#id`, `.class`, any CSS selector, or a direct element reference
- Works **on top of the reactive system** — no additional wiring
- Use alongside `effect()`: Conditions handles **which mode**, effects handle **dynamic values within a mode**
- Returns a dispose function — call it to stop watching
- Extensible via `registerMatcher()` and `registerHandler()`

**The rule:**
```
Finite, named states  (idle | loading | error)  →  Conditions.whenState()
Continuously changing values (count, text, url) →  effect() + .update()
```

---

## What's Next?

Continue to: [14 — Full Power: Reactive + DOM Helpers Together](./14_full_power_reactive_dom_helpers.md)
