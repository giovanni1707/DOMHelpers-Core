[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Async Effects and Enhanced Async State

## Part 1: asyncEffect — Async Effects with Cancellation

### What is it?

`asyncEffect` is a reactive effect designed for async operations like `fetch`. It provides:

- An **AbortSignal** for cancelling in-flight requests
- **Automatic cancellation** when the effect re-runs (because a dependency changed)
- **Cleanup function support** — return a function from your async work to clean up

---

### Basic Example

```javascript
const state = state({ query: 'cats' });

const stop = asyncEffect(async (signal) => {
  const response = await fetch(`/api/search?q=${state.query}`, { signal });
  const data = await response.json();
  Elements.results.update({
    innerHTML: data.map(item => `<div>${item.name}</div>`).join('')
  });
});
```

**What happens:**

```
1️⃣ Effect runs immediately
   ↓
   fetch('/api/search?q=cats', { signal }) starts
   ↓
2️⃣ User types a new query → state.query = 'dogs'
   ↓
   Previous fetch is ABORTED (signal.abort())
   ↓
   New fetch('/api/search?q=dogs', { signal }) starts
   ↓
3️⃣ Response arrives → UI updated with dog results
```

Without `asyncEffect`, the first request might finish after the second one, showing stale "cats" results when the user expects "dogs." This is a **race condition**. The `AbortSignal` prevents it by cancelling the old request.

---

### The signal Parameter

Your effect function receives an `AbortSignal` as its first argument. Pass it to `fetch` or any other API that supports cancellation:

```javascript
asyncEffect(async (signal) => {
  // Pass signal to fetch — it will abort if the effect re-runs
  const response = await fetch(url, { signal });

  // Check if aborted before doing work
  if (signal.aborted) return;

  // Process the response
  const data = await response.json();
  renderData(data);
});
```

---

### Cleanup Function

If your async effect returns a function, it will be called as cleanup before the next run:

```javascript
asyncEffect(async (signal) => {
  const ws = new WebSocket(`wss://api.example.com/${state.channel}`);

  ws.onmessage = (e) => {
    Elements.messages.update({ innerHTML: Elements.messages.innerHTML + `<p>${e.data}</p>` });
  };

  // Return a cleanup function
  return () => {
    ws.close();  // Close the WebSocket when the effect re-runs
  };
});
```

**Flow:**

```
First run:
   ↓
WebSocket opens for channel "general"
   ↓
state.channel = 'random' (dependency changes)
   ↓
Cleanup runs → ws.close() (closes old WebSocket)
   ↓
AbortController.abort() (cancels any pending fetch)
   ↓
New run → WebSocket opens for channel "random"
```

---

### Error Handling

Aborted requests are handled gracefully — `AbortError` is silently caught:

```javascript
asyncEffect(async (signal) => {
  const response = await fetch(url, { signal });
  // If aborted, the fetch throws AbortError
  // asyncEffect catches it silently — no error logged
  const data = await response.json();
  renderData(data);
}, {
  onError: (error) => {
    // Only called for NON-abort errors
    console.error('Fetch failed:', error.message);
  }
});
```

---

### Disposing an asyncEffect

The `asyncEffect` returns a dispose function that:
1. Disposes the reactive effect
2. Runs the cleanup function (if any)
3. Aborts the current request

```javascript
const stop = asyncEffect(async (signal) => {
  const response = await fetch(url, { signal });
  // ...
});

// Later:
stop();
// Effect disposed + any in-flight request aborted
```

---

## Part 2: Enhanced Async State

### What is it?

`asyncState` is a reactive state designed for async data fetching. It wraps data, loading, and error states into one object and prevents race conditions using a **request ID** pattern.

---

### Creating an Async State

```javascript
const users = asyncState(null);
// or
const users = ReactiveEnhancements.asyncState(null, { /* options */ });
```

**What you get:**

```
asyncState(null)
   ↓
Returns a reactive state:
{
  data: null,           // The fetched data
  loading: false,       // Is a request in progress?
  error: null,          // Last error (or null)
  requestId: 0,         // Tracks request ordering

  // Computed properties:
  isSuccess: false,     // !loading && !error && data !== null
  isError: false,       // !loading && error !== null
  isIdle: true,         // !loading && data === null && error === null

  // Methods:
  execute(fn),         // Run an async function
  abort(),             // Cancel current request
  reset(),             // Reset to initial state
  refetch()            // Re-run the last execute function
}
```

---

### execute — Running Async Functions

```javascript
const users = asyncState([]);

await users.execute(async (signal) => {
  const response = await fetch('/api/users', { signal });
  return response.json();
});

console.log(users.data);       // [...users]
console.log(users.loading);    // false
console.log(users.isSuccess);  // true
```

**What happens inside execute:**

```
users.execute(fn)
   ↓
1️⃣ Abort previous request (if any)
   ↓
2️⃣ requestId++ (now 1)
   ↓
3️⃣ Create new AbortController
   ↓
4️⃣ Set loading = true, error = null
   ↓
5️⃣ await fn(signal)
   ↓
6️⃣ Success?
   ├── YES → Is requestId still 1? (still the latest request?)
   │         ├── YES → data = result, return { success: true, data }
   │         └── NO  → return { success: false, stale: true }
   │
   └── ERROR → Is it AbortError?
              ├── YES → return { success: false, aborted: true }
              └── NO  → Is requestId still 1?
                        ├── YES → error = error, return { success: false, error }
                        └── NO  → Ignore (stale request)
   ↓
7️⃣ finally: if requestId still matches → loading = false
```

---

### Race Condition Prevention

The `requestId` pattern prevents stale responses from overwriting fresh data:

```javascript
const users = asyncState([]);

// Request 1: fetch page 1
users.execute(async (signal) => {
  const res = await fetch('/api/users?page=1', { signal });
  return res.json();  // Takes 3 seconds
});

// 500ms later — user clicks "page 2"
// Request 2: fetch page 2
users.execute(async (signal) => {
  const res = await fetch('/api/users?page=2', { signal });
  return res.json();  // Takes 1 second
});
```

**What happens:**

```
Request 1 starts (requestId = 1)
   ↓ 500ms
Request 2 starts:
   1. Aborts request 1 (signal.abort())
   2. requestId becomes 2
   ↓ 1 second
Request 2 finishes:
   requestId is 2, current requestId is 2 → MATCH ✅
   data = page 2 results
   ↓
Request 1 would have finished later, but it was aborted
   Even if it somehow resolved, requestId is 1 ≠ 2 → STALE, ignored
```

---

### abort — Cancelling a Request

```javascript
const users = asyncState([]);

users.execute(async (signal) => {
  const res = await fetch('/api/users', { signal });
  return res.json();
});

// Cancel the request
users.abort();

console.log(users.loading);  // false
console.log(users.data);     // [] (unchanged)
```

---

### reset — Back to Initial State

```javascript
const users = asyncState([]);

await users.execute(async () => {
  return [{ name: 'Alice' }];
});

console.log(users.data);      // [{ name: 'Alice' }]
console.log(users.isSuccess); // true

users.reset();

console.log(users.data);      // [] (initial value)
console.log(users.loading);   // false
console.log(users.error);     // null
console.log(users.isIdle);    // true
console.log(users.requestId); // 0
```

`reset` also aborts any in-flight request.

---

### refetch — Re-run the Last Function

```javascript
const users = asyncState([]);

await users.execute(async (signal) => {
  const res = await fetch('/api/users', { signal });
  return res.json();
});

// Later, refresh the data with the same function:
await users.refetch();
// Re-runs the last function passed to execute
```

If no function was previously executed, `refetch` returns:
```javascript
{ success: false, error: Error('No function to refetch') }
```

---

### Options

```javascript
const users = asyncState(null, {
  onSuccess: (data) => {
    console.log('Fetched:', data.length, 'users');
  },
  onError: (error) => {
    console.error('Fetch failed:', error.message);
  }
});
```

| Option | Type | Description |
|--------|------|-------------|
| `onSuccess` | Function | Called with the data on successful fetch |
| `onError` | Function | Called with the error on failed fetch |

---

### Using Async State with Effects

```javascript
const users = asyncState([]);

// Effect re-runs whenever loading, data, or error changes
effect(() => {
  let html;
  if (users.loading)       html = '<p>Loading...</p>';
  else if (users.isError)  html = `<p>Error: ${users.error.message}</p>`;
  else if (users.isSuccess) html = users.data.map(u => `<div>${u.name}</div>`).join('');
  else                     html = '<p>Click "Load" to fetch users</p>';
  Elements.users.update({ innerHTML: html });
});

// Fetch on button click
Elements.loadBtn.update({ onclick: () => { });
  users.execute(async (signal) => {
    const res = await fetch('/api/users', { signal });
    return res.json();
  });
};

// Refresh button
Elements.refreshBtn.update({ onclick: () => { });
  users.refetch();
};
```

---

### Computed Properties on Async State

The async state comes with three computed properties:

| Property | Type | True when |
|----------|------|-----------|
| `.isSuccess` | Boolean | `!loading && !error && data !== null` |
| `.isError` | Boolean | `!loading && error !== null` |
| `.isIdle` | Boolean | `!loading && data === null && error === null` |

These update automatically and can be used in effects:

```javascript
effect(() => {
  Elements.refreshBtn.update({ disabled: users.loading });
  Elements.status.update({ textContent: users.isIdle ? 'Ready' : });
    users.loading ? 'Loading...' :
    users.isSuccess ? 'Loaded' :
    'Error';
});
```

---

## Common Mistakes

### ❌ Forgetting to pass signal to fetch

```javascript
// ❌ Request won't be cancelled when effect re-runs
asyncEffect(async (signal) => {
  const response = await fetch(url);  // No signal!
  // ...
});

// ✅ Always pass signal to fetch
asyncEffect(async (signal) => {
  const response = await fetch(url, { signal });
  // ...
});
```

### ❌ Using asyncState without await on execute

```javascript
// ❌ Can't check the result synchronously
const result = users.execute(fetchFn);
console.log(result);  // Promise, not the result

// ✅ Use await
const result = await users.execute(fetchFn);
console.log(result);  // { success: true, data: [...] }
```

### ❌ Ignoring the return value of execute

```javascript
// The return value tells you what happened
const result = await users.execute(fetchFn);

if (result.success) {
  console.log('Data:', result.data);
} else if (result.aborted) {
  console.log('Request was cancelled');
} else if (result.stale) {
  console.log('A newer request replaced this one');
} else {
  console.log('Error:', result.error);
}
```

---

## Key Takeaways

1. **asyncEffect** — reactive effect for async work with AbortSignal
2. **Automatic cancellation** — previous requests abort when the effect re-runs
3. **AbortError is silenced** — only real errors trigger error handlers
4. **asyncState** — complete async data management (data, loading, error)
5. **requestId pattern** — prevents race conditions where stale responses overwrite fresh data
6. **execute** — run async functions with cancellation and result tracking
7. **abort** — manually cancel in-flight requests
8. **refetch** — re-run the last function without re-specifying it
9. **reset** — return to initial state (also aborts)
10. **Computed properties** (`.isSuccess`, `.isError`, `.isIdle`) make UI state simple

---

## What's next?

Let's explore the DevTools, see real-world examples, and wrap up with the complete API reference.

Let's continue!