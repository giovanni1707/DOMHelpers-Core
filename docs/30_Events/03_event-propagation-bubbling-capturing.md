[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Event Propagation, Bubbling & Capturing

When an event fires on an element, the browser does not deliver it only to that element. It travels — first down through the DOM tree, then back up. Understanding this propagation model is essential for writing correct event handlers and unlocking the delegation pattern that makes dynamic UIs easier to build.

---

## How Events Travel Through the DOM

Every event dispatched in the browser goes through three distinct phases:

```
Document
  └─ html
       └─ body
            └─ section#content
                 └─ div.card
                      └─ button#pay-btn   ← event originates here
```

When the user clicks `#pay-btn`, the event does **not** just fire on that button. It travels in three phases:

### Phase 1 — Capturing (top → target)

The event starts at `document` and travels **down** through every ancestor of the target, all the way to the target itself.

```
document → html → body → section#content → div.card → button#pay-btn
```

Listeners registered with `{ capture: true }` fire during this phase — before the target or any bubbling listener sees the event.

### Phase 2 — Target

The event arrives at `button#pay-btn`. Both capturing and bubbling listeners on the target itself fire here (in the order they were registered).

### Phase 3 — Bubbling (target → top)

After the target, the event travels **back up** through every ancestor all the way to `document`.

```
button#pay-btn → div.card → section#content → body → html → document
```

Listeners registered without `{ capture: true }` (the default) fire during this phase.

---

## Visualizing the Three Phases

```
          ┌────────────────────────────────────────┐
          │           CAPTURING PHASE              │
          │   document                             │
          │      ↓                                 │
          │   body                                 │
          │      ↓                                 │
          │   div.card                             │
          │      ↓                                 │
          │   button#pay-btn  ← TARGET PHASE       │
          │      ↑                                 │
          │   div.card                             │
          │      ↑                                 │
          │   body                                 │
          │      ↑                                 │
          │   document                             │
          │           BUBBLING PHASE               │
          └────────────────────────────────────────┘
```

Most handlers use the **bubbling phase** — this is the default when no options are passed. The capturing phase is used for interception: catching an event before it reaches the target.

---

## Bubbling — The Default Behavior

Bubbling is what most developers encounter first, and it is the phase DOM Helpers handlers fire on by default.

### The plain JavaScript problem

Without understanding bubbling, developers often write redundant listeners:

```js
// Plain JS — attaching to every card separately
document.querySelectorAll('.card').forEach(card => {
  card.addEventListener('click', handleCardClick);
});

// If new cards are added later, they get no listener.
// Must re-run setup every time the DOM changes.
```

### With DOM Helpers — using bubbling intentionally

Since click events bubble up from any descendant, you can listen on the container and act on any card inside it — present or future:

```js
// One listener on the container catches clicks on all current and future .card children
Elements.grid.update({
  addEventListener: {
    click: (e) => {
      const card = e.target.closest('.card');
      if (!card) return;

      // e.target is the element clicked — could be a nested <span> or <p>
      // .closest() walks up to find the nearest .card ancestor
      card.update({
        classList: { toggle: 'selected' }
      });
    }
  }
});
```

Because `card` is the actual DOM element found by `.closest()`, you can call `.update()` on it directly — `card` is already enhanced.

---

## Capturing — Listening Before the Target

The capture phase fires **before** any handler on the target or any bubbling handler. You opt into it with `{ capture: true }`:

```js
// Plain JS
document.addEventListener('click', interceptClick, { capture: true });

// DOM Helpers
Elements.overlay.update({
  addEventListener: ['click', interceptClick, { capture: true }]
});
```

### When to use capturing

**Global click-outside detection** — close a dropdown when anything outside it is clicked:

```js
document.addEventListener('click', (e) => {
  const dropdown = document.getElementById('dropdown-menu');
  if (!dropdown.contains(e.target)) {
    Elements.dropdownMenu.update({ classList: { remove: 'open' } });
  }
}, { capture: true });
```

**Blocking events before they reach a subtree** — prevent all clicks inside a disabled panel:

```js
Elements.disabledPanel.update({
  addEventListener: ['click', (e) => {
    e.stopPropagation();
    e.preventDefault();
  }, { capture: true }]
});
```

**Measuring timing** — instrumenting event latency before handlers run.

---

## Stopping Propagation

Sometimes you want an event to stop traveling. The browser gives you three tools — and DOM Helpers handlers have full access to all of them through the event object `e`.

### `e.stopPropagation()`

Stops the event from moving to the next element in the propagation chain. Handlers already registered on the current element continue to fire.

```js
Elements.card.update({
  addEventListener: {
    click: (e) => {
      e.stopPropagation(); // Click will not bubble to parent elements
      e.target.update({ classList: { toggle: 'selected' } });
    }
  }
});

// This parent listener will NOT fire when .card is clicked
Elements.grid.update({
  addEventListener: {
    click: (e) => {
      console.log('Grid clicked'); // Skipped when card handles the click
    }
  }
});
```

### `e.stopImmediatePropagation()`

Stops propagation **and** prevents any other listeners on the current element from firing. Use this when you have multiple handlers on the same element and one should block the rest.

```js
Elements.btn.update({
  addEventListener: {
    click: (e) => {
      if (!isFormValid()) {
        e.stopImmediatePropagation(); // Blocks any other click listeners on #btn
        e.preventDefault();
        Elements.errorMsg.update({ style: { display: 'block' } });
      }
    }
  }
});

// If validation fails, this second handler will NOT run
Elements.btn.update({
  addEventListener: ['click', submitForm]
});
```

### `e.preventDefault()`

Prevents the browser's default action for the event — does not affect propagation at all.

```js
// Prevent form submission (so you can handle it in JS)
Elements.form.update({
  addEventListener: {
    submit: (e) => {
      e.preventDefault();
      handleFormSubmit(e);
    }
  }
});

// Prevent a link from navigating
Elements.navLink.update({
  addEventListener: {
    click: (e) => {
      e.preventDefault();
      Router.go(e.target.getAttribute('href'));
    }
  }
});

// Prevent the context menu from appearing
Elements.canvas.update({
  addEventListener: {
    contextmenu: (e) => {
      e.preventDefault();
      showCustomContextMenu(e.clientX, e.clientY);
    }
  }
});
```

### Comparison

| Method | Stops bubbling? | Stops other handlers on same element? | Stops browser default? |
|---|---|---|---|
| `stopPropagation()` | Yes | No | No |
| `stopImmediatePropagation()` | Yes | Yes | No |
| `preventDefault()` | No | No | Yes |

---

## Event Delegation — One Listener for Many Elements

Delegation is the practical application of bubbling. Instead of attaching a listener to every child, attach one listener to a common ancestor and let bubbling bring every child's event up to it.

### Why delegation matters

- **Dynamic content** — elements added after the listener is registered are automatically handled
- **Performance** — one listener vs. hundreds
- **Simplicity** — setup runs once, not once per element

### The `e.target` and `e.currentTarget` distinction

Inside a delegated listener, two event properties tell you different things:

| Property | What It Is |
|---|---|
| `e.target` | The element the user actually clicked (deepest source) |
| `e.currentTarget` | The element the listener is attached to (the delegate) |

```js
Elements.userList.update({
  addEventListener: {
    click: (e) => {
      // e.currentTarget → #userList (always the list — our listener's element)
      // e.target        → whatever was clicked (could be a <button>, <span>, or <li>)

      const item = e.target.closest('li.user-item');
      if (!item) return;

      const action = e.target.dataset.action;

      if (action === 'edit')   openEditModal(item.dataset.userId);
      if (action === 'delete') confirmDelete(item.dataset.userId);
      if (action === 'view')   navigateTo(`/users/${item.dataset.userId}`);
    }
  }
});
```

### Full delegation example — dynamic todo list

```js
// HTML (items may be added/removed dynamically):
// <ul id="todo-list">
//   <li data-id="1">
//     <span class="title">Buy groceries</span>
//     <button data-action="complete">✓</button>
//     <button data-action="delete">✗</button>
//   </li>
// </ul>

Elements.todoList.update({
  addEventListener: {
    click: (e) => {
      const action = e.target.dataset.action;
      if (!action) return;

      const item = e.target.closest('li');
      const id   = item?.dataset.id;
      if (!item || !id) return;

      if (action === 'complete') {
        item.update({
          classList:   { add: 'completed' },
          setAttribute: { 'aria-checked': 'true' }
        });
        markTodoDone(id);
      }

      if (action === 'delete') {
        item.update({ remove: [] });
        deleteTodo(id);
      }
    }
  }
});
```

New `<li>` items added to `#todo-list` later are automatically covered — no re-registration needed.

---

## Combining Capturing and Delegation

Capture-phase delegation is useful when you need to intercept events before they reach any descendant handler:

```js
// Log every click anywhere in the app — even if children stop propagation
document.addEventListener('click', (e) => {
  analytics.track('click', {
    element: e.target.tagName,
    id:      e.target.id,
    classes: e.target.className
  });
}, { capture: true });
```

Because capturing fires before any bubbling handler, even elements that call `e.stopPropagation()` in their own click handlers will still be logged here.

---

## Propagation in Practice — A Modal System

A complete example that uses bubbling, capturing, `stopPropagation`, and delegation together:

```js
// Open modal
Elements.openModalBtn.update({
  addEventListener: {
    click: () => {
      Elements.modal.update({
        style:        { display: 'flex' },
        classList:    { add: 'open' },
        setAttribute: { 'aria-hidden': 'false' }
      });
    }
  }
});

// Close on overlay click (bubbling from overlay → modal background)
Elements.modal.update({
  addEventListener: {
    click: (e) => {
      // Only close if the click landed directly on the modal backdrop — not a child
      if (e.target === e.currentTarget) {
        closeModal();
      }
    }
  }
});

// Stop clicks inside the modal content from reaching the backdrop
Elements.modalContent.update({
  addEventListener: {
    click: (e) => e.stopPropagation()
  }
});

// Close button inside the modal
Elements.modalCloseBtn.update({
  addEventListener: {
    click: closeModal
  }
});

// Global Escape key listener (capture — fires before any other handler)
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape') closeModal();
}, { capture: true });

function closeModal() {
  Elements.modal.update({
    style:        { display: 'none' },
    classList:    { remove: 'open' },
    setAttribute: { 'aria-hidden': 'true' }
  });
}
```

---

## Summary

| Concept | Key Point |
|---|---|
| **Bubbling** | Events travel up from target to document — the default phase for all handlers |
| **Capturing** | Events travel down from document to target — opt in with `{ capture: true }` |
| **Target phase** | Handlers on the target itself fire for both phases |
| **`stopPropagation()`** | Stops the event moving to the next element; other handlers on current element still fire |
| **`stopImmediatePropagation()`** | Stops propagation and cancels remaining handlers on the current element |
| **`preventDefault()`** | Cancels the browser's default action; does not affect propagation |
| **Delegation** | One ancestor listener handles events for all descendants via `e.target.closest()` |
| **`e.target`** | The element that originated the event |
| **`e.currentTarget`** | The element where the current listener is attached |

---

Next: **[Advanced Patterns & Best Practices](./04_advanced-patterns-and-best-practices)**
