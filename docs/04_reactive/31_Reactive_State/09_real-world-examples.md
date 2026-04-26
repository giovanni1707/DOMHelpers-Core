[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Real-World Examples

Let's see reactive state solving real problems you'll encounter in web development.

---

## Example 1: Counter App

### The scenario

A counter with increment, decrement, reset, and a doubled display.

```javascript
const counter = state({ count: 0 });
computed(counter, {
  doubled: function() { return this.count * 2; },
  isNegative: function() { return this.count < 0; }
});

effect(() => {
  Elements.update({
    count:   { textContent: counter.count, style: { color: counter.isNegative ? 'red' : 'green' } },
    doubled: { textContent: counter.doubled }
  });
});

Elements.incBtn.addEventListener('click', () => counter.count++);
Elements.decBtn.addEventListener('click', () => counter.count--);
Elements.resetBtn.addEventListener('click', () => counter.count = 0);
```

---

## Example 2: Todo List

### The scenario

A todo list with add, toggle, remove, and filter functionality.

```javascript
const todos = store(
  {
    items: [],
    filter: 'all',
    newText: ''
  },
  {
    getters: {
      filtered() {
        if (this.filter === 'active') return this.items.filter(t => !t.done);
        if (this.filter === 'done') return this.items.filter(t => t.done);
        return this.items;
      },
      remaining() {
        return this.items.filter(t => !t.done).length;
      }
    },
    actions: {
      add(state) {
        if (!state.newText.trim()) return;
        state.items.push({ id: Date.now(), text: state.newText, done: false });
        state.newText = '';
      },
      toggle(state, id) {
        const item = state.items.find(t => t.id === id);
        if (item) item.done = !item.done;
      },
      remove(state, id) {
        const idx = state.items.findIndex(t => t.id === id);
        if (idx !== -1) state.items.splice(idx, 1);
      }
    }
  }
);

effect(() => {
  Elements.remaining.update({ textContent: `${todos.remaining} left` });
});
```

---

## Example 3: Form Validation

### The scenario

A signup form with real-time validation.

```javascript
const signup = form({
  name: '',
  email: '',
  password: ''
});

// Validate on change
watch(signup, 
  function() { return this.values.email; },
  (email) => {
    if (!email) signup.setError('email', null);
    else if (!email.includes('@')) signup.setError('email', 'Invalid email');
    else signup.setError('email', null);
  }
);

watch(signup, 
  function() { return this.values.password; },
  (password) => {
    if (!password) signup.setError('password', null);
    else if (password.length < 8) signup.setError('password', '8+ characters required');
    else signup.setError('password', null);
  }
);

// Keep UI in sync
effect(() => {
  Elements.update({
    emailError: { textContent: signup.errors.email || '' },
    passwordError: { textContent: signup.errors.password || '' },
    submitBtn: { disabled: !signup.isValid || !signup.isDirty }
  });
});
```

---

## Example 4: Dashboard with Live Metrics

### The scenario

A dashboard that fetches and displays live metrics.

```javascript
const dashboard = state({
  users: 0,
  revenue: 0,
  orders: 0,
  lastUpdated: null
});

computed(dashboard, {
  avgOrderValue: function() {
    return this.orders > 0 ? this.revenue / this.orders : 0;
  }
});

// Bind to DOM
effect(() => {
  Elements.update({
    userCount:  { textContent: dashboard.users.toLocaleString() },
    revenue:    { textContent: `$${dashboard.revenue.toLocaleString()}` },
    orderCount: { textContent: dashboard.orders.toLocaleString() },
    avgOrder:   { textContent: `$${dashboard.avgOrderValue.toFixed(2)}` },
    lastUpdate: {
      textContent: dashboard.lastUpdated
        ? new Date(dashboard.lastUpdated).toLocaleTimeString()
        : 'Never'
    }
  });
});

// Fetch data periodically
async function refreshMetrics() {
  const data = await fetch('/api/metrics').then(r => r.json());
  batch(() => {
    dashboard.users = data.users;
    dashboard.revenue = data.revenue;
    dashboard.orders = data.orders;
    dashboard.lastUpdated = Date.now();
  });
}

setInterval(refreshMetrics, 30000);
refreshMetrics();
```

---

## Example 5: Shopping Cart

### The scenario

A shopping cart with add, remove, quantity updates, and total calculation.

```javascript
const cart = store(
  { items: [], discount: 0 },
  {
    getters: {
      itemCount() {
        return this.items.reduce((sum, i) => sum + i.qty, 0);
      },
      subtotal() {
        return this.items.reduce((sum, i) => sum + i.price * i.qty, 0);
      },
      total() {
        return this.subtotal - this.discount;
      }
    },
    actions: {
      addItem(state, product) {
        const existing = state.items.find(i => i.id === product.id);
        if (existing) {
          existing.qty++;
        } else {
          state.items.push({ ...product, qty: 1 });
        }
      },
      removeItem(state, id) {
        const idx = state.items.findIndex(i => i.id === id);
        if (idx !== -1) state.items.splice(idx, 1);
      },
      setQty(state, id, qty) {
        const item = state.items.find(i => i.id === id);
        if (item) item.qty = Math.max(1, qty);
      }
    }
  }
);

effect(() => {
  Elements.update({
    cartCount:   { textContent: cart.itemCount },
    subtotal:    { textContent: `$${cart.subtotal.toFixed(2)}` },
    total:       { textContent: `$${cart.total.toFixed(2)}` },
    checkoutBtn: { disabled: cart.itemCount === 0 }
  });
});
```

---

## Example 6: API Data Loader

### The scenario

Load user data from an API with loading and error states.

```javascript
const userData = asyncState(null);

// UI reacts to loading state
effect(() => {
  Elements.update({
    spinner: { hidden: !userData.loading },
    content: { hidden: !userData.isSuccess },
    error:   { hidden: !userData.isError }
  });

  if (userData.isSuccess) {
    Elements.update({
      userName:  { textContent: userData.data.name },
      userEmail: { textContent: userData.data.email }
    });
  }

  if (userData.isError) {
    Elements.error.update({ textContent: `Error: ${userData.error.message}` });
  }
});

// Load button
Elements.loadBtn.addEventListener('click', async () => {
  try {
    await userData.execute(async () => {
      const res = await fetch('/api/user/123');
      if (!res.ok) throw new Error('Failed to load user');
      return res.json();
    });
  } catch (e) {
    // Error is already stored in userData.error
  }
});
```

---

## Example 7: Theme Switcher

### The scenario

A theme system that persists to localStorage and updates the entire page.

```javascript
const theme = state({ mode: 'light', fontSize: 16 });

// Persist to localStorage automatically — loads saved values on startup
autoSave(theme, 'theme-prefs');

// Apply theme to page
effect(() => {
  document.body.update({
    setAttribute: { 'data-theme': theme.mode },
    style: { fontSize: `${theme.fontSize}px` }
  });
});

// Toggle button
Elements.themeToggle.addEventListener('click', () => {
  theme.mode = theme.mode === 'light' ? 'dark' : 'light';
});

// Font size controls
Elements.fontUp.addEventListener('click', () => {
  theme.fontSize = Math.min(24, theme.fontSize + 2);
});

Elements.fontDown.addEventListener('click', () => {
  theme.fontSize = Math.max(12, theme.fontSize - 2);
});
```

---

## Example 8: Tab System

### The scenario

A tab component with content switching.

```javascript
const tabs = state({
  active: 'home',
  content: {
    home: 'Welcome to the home page',
    about: 'Learn more about us',
    contact: 'Get in touch'
  }
});

computed(tabs, {
  currentContent: function() {
    return this.content[this.active] || 'Page not found';
  }
});

effect(() => {
  // Update tab buttons
  document.querySelectorAll('.tab-btn').forEach(btn => {
    btn.classList.toggle('active', btn.dataset.tab === tabs.active);
  });

  // Update content
  Elements.tabContent.update({ textContent: tabs.currentContent });
});

// Tab click handlers
document.querySelectorAll('.tab-btn').forEach(btn => {
  btn.addEventListener('click', () => {
    tabs.active = btn.dataset.tab;
  });
});
```

---

## Example 9: Collection with Component

### The scenario

A notification system using the component factory.

```javascript
const notifications = component({
  state: {
    items: [],
    maxVisible: 3
  },

  computed: {
    visible() {
      return this.items.slice(0, this.maxVisible);
    },
    hasMore() {
      return this.items.length > this.maxVisible;
    }
  },

  watch: {
    items(newItems) {
      if (newItems.length > 10) {
        this.items = newItems.slice(-10);
      }
    }
  },

  actions: {
    add(state, message, type = 'info') {
      state.items.push({
        id: Date.now(),
        message,
        type,
        timestamp: new Date()
      });
    },
    dismiss(state, id) {
      const idx = state.items.findIndex(n => n.id === id);
      if (idx !== -1) state.items.splice(idx, 1);
    },
    clearAll(state) {
      state.items = [];
    }
  },

  mounted() {
    console.log('Notification system ready');
  }
});

// Usage
notifications.add('Welcome!', 'success');
notifications.add('New message received', 'info');
notifications.dismiss(notifications.items[0].id);
```

---

## Example 10: Reactive Builder Pattern

### The scenario

Build a search component using the chainable builder.

```javascript
const search = reactive({
    query: '',
    results: [],
    isSearching: false
  })
  .computed({
    hasResults() { return this.results.length > 0; },
    resultCount() { return this.results.length; }
  })
  .watch({
    query(newQuery) {
      if (newQuery.length >= 3) {
        performSearch(newQuery);
      }
    }
  })
  .action('clear', (state) => {
    state.query = '';
    state.results = [];
  })
  .build();

async function performSearch(query) {
  search.isSearching = true;
  try {
    const res = await fetch(`/api/search?q=${query}`);
    search.results = await res.json();
  } finally {
    search.isSearching = false;
  }
}

effect(() => {
  Elements.resultCount.update({
    textContent: search.isSearching ? 'Searching...' : `${search.resultCount} results`
  });
});
```

---

## Common patterns summary

### Pattern 1: State + Effect

```javascript
const app = state({ value: 0 });
effect(() => {
  Elements.display.update({ textContent: app.value });
});
```

### Pattern 2: Store + Getters + Actions

```javascript
const store = store(initialState, { getters, actions });
```

### Pattern 3: Form + Validation

```javascript
const form = form(initialValues);
watch(form, /* field */, /* validate */);
```

### Pattern 4: Async + Loading UI

```javascript
const data = async();
await data.execute(fetchFunction);
```

---

## Key takeaways

1. Start simple with **state() + effect()** for basic reactive UIs
2. Use **store()** when you need organized getters and actions
3. Use **form()** for any form with validation
4. Use **async()** for API calls with loading/error states
5. Use **component()** for self-contained UI components with lifecycle
6. Use **reactive() builder** when building incrementally
7. **batch()** when making multiple state changes that should trigger one update
8. **bind()** for declarative DOM bindings without manual effects

---

## What's next?

Finally, let's look at the complete utilities and API reference:
- All available methods
- Global exports
- Integration points
- The iteration utilities (eachEntries, mapEntries)

Let's wrap up!