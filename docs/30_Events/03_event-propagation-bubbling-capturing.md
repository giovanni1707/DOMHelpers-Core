[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Event Propagation, Bubbling & Capturing

When you click something on a page, the browser doesn't just notify that one element. The event *travels* — first down through the DOM tree to find the element you clicked, then back up again through every ancestor.

This behavior is called **event propagation**, and once you understand it, a lot of things that used to feel like workarounds start making sense — including the powerful delegation pattern that lets one listener cover hundreds of elements.

---

## The Journey of an Event

Let's say you have this HTML structure:

```html
<div id="grid">
  <div class="card">
    <button id="pay-btn">Pay Now</button>
  </div>
</div>
```

When the user clicks `#pay-btn`, the browser doesn't skip straight to firing the button's click handler. The event goes on a journey through three phases:

### Phase 1 — Capturing (travels down)

The event starts at the very top (`document`) and travels *down* through every ancestor until it reaches the target element:

```
document → html → body → #grid → .card → #pay-btn
```

Any listener registered with `{ capture: true }` fires during this downward journey. This is called the **capture phase**.

### Phase 2 — Target

The event arrives at `#pay-btn`. Listeners on the target itself fire here — both capturing and non-capturing ones, in the order they were registered.

### Phase 3 — Bubbling (travels back up)

After the target, the event travels *back up* through every ancestor:

```
#pay-btn → .card → #grid → body → html → document
```

This is called the **bubbling phase**, and it's what most event listeners use by default. When you call `addEventListener` without any options, your listener fires during bubbling.

Here's the full picture:

```
          ┌─────────────────────────────────────┐
          │         CAPTURING PHASE             │
          │  document                           │
          │     ↓                               │
          │  body                               │
          │     ↓                               │
          │  #grid                              │
          │     ↓                               │
          │  .card                              │
          │     ↓                               │
          │  #pay-btn     ← TARGET PHASE        │
          │     ↑                               │
          │  .card                              │
          │     ↑                               │
          │  #grid                              │
          │     ↑                               │
          │  body                               │
          │     ↑                               │
          │  document                           │
          │         BUBBLING PHASE              │
          └─────────────────────────────────────┘
```

The important takeaway: **every event passes through every ancestor twice** — once going down, once coming up. This is what makes delegation possible.

---

## Bubbling — The Default Behavior

Bubbling is the phase you're already using whenever you call `addEventListener` without options. It's also the default in DOM Helpers.

The reason bubbling is so useful is that events from *any* descendant bubble up to every ancestor. A click on a `<button>` inside a `<div>` will bubble up to that `<div>`.

That means you can listen on a container and react to events from all of its children — without attaching anything to the children themselves.

### The problem with attaching listeners to every child

```js
// Plain JS — one listener per card
document.querySelectorAll('.card').forEach(card => {
  card.addEventListener('click', handleCardClick);
});
```

This works initially — but if new cards are added to the page later (loaded from an API, added by the user), they get no listener. You'd have to re-run the setup every time the DOM changes.

### Using bubbling intentionally — the delegation pattern

Instead of listening on every card, listen on the container. When any card is clicked, the event bubbles up to the container and your listener fires:

```js
// One listener on the parent — works for ALL current and future cards
Elements.grid.update({
  addEventListener: {
    click: (e) => {
      // e.target is the element the user actually clicked
      // It might be the .card itself, or a <span> or <p> inside it
      // .closest() walks up the tree to find the nearest matching ancestor
      const card = e.target.closest('.card');
      if (!card) return; // User clicked something outside a card — ignore

      card.update({
        classList: { toggle: 'selected' }
      });
    }
  }
});
```

Because `card` is a real DOM element that DOM Helpers has enhanced, you can call `.update()` on it directly.

This pattern — **event delegation** — is one of the most useful techniques in DOM programming. One listener, infinite elements, no re-registration.

---

## `e.target` vs `e.currentTarget`

When you're inside a delegated listener, there are two properties on the event object that sound similar but mean different things:

| Property | What It Points To |
|---|---|
| `e.target` | The element the user actually clicked — could be deep inside the tree |
| `e.currentTarget` | The element your listener is attached to — the one you put `addEventListener` on |

```js
Elements.userList.update({
  addEventListener: {
    click: (e) => {
      // e.currentTarget is always #userList — your listener lives here
      // e.target is whatever the user actually clicked inside the list
      //   — could be a <button>, a <span>, an <img>, anything

      const item = e.target.closest('li.user-item');
      if (!item) return;

      const action = e.target.dataset.action; // Read from the clicked element

      if (action === 'edit')   openEditModal(item.dataset.userId);
      if (action === 'delete') confirmDelete(item.dataset.userId);
      if (action === 'view')   navigateTo(`/users/${item.dataset.userId}`);
    }
  }
});
```

The `if (!item) return` guard is important — it makes sure we only react when the click came from inside a list item, not from the whitespace around them.

---

## Stopping Propagation

Sometimes you want an event to stop traveling. The browser gives you three methods for this, and all of them work normally inside DOM Helpers handlers.

### `e.stopPropagation()` — stop here, don't go further

This stops the event from continuing to bubble up (or travel down during capture). Other listeners registered on the same element will still fire — only the propagation to other elements is stopped.

```js
Elements.card.update({
  addEventListener: {
    click: (e) => {
      e.stopPropagation(); // The click will NOT bubble up to #grid or any parent

      e.target.update({
        classList: { toggle: 'selected' }
      });
    }
  }
});

// Because stopPropagation() was called on .card, this will NOT fire
Elements.grid.update({
  addEventListener: {
    click: () => {
      console.log('grid was clicked'); // Never runs when a card is clicked
    }
  }
});
```

### `e.stopImmediatePropagation()` — stop here AND block other handlers on this element

This does everything `stopPropagation()` does, plus it prevents any other listeners on the *same* element from firing. Use it when you have multiple handlers on one element and the first one should cancel the rest:

```js
// This runs first and checks if the form is valid
Elements.submitBtn.update({
  addEventListener: {
    click: (e) => {
      if (!isFormValid()) {
        // Invalid — stop everything: don't propagate, and don't run the submit handler below
        e.stopImmediatePropagation();
        Elements.errorMsg.update({ style: { display: 'block' } });
      }
    }
  }
});

// This runs second — but only if the validation above didn't stop things
Elements.submitBtn.update({
  addEventListener: ['click', submitForm]
});
```

### `e.preventDefault()` — cancel the browser's default behavior

This has nothing to do with propagation. It tells the browser not to do what it normally would for this event — like navigating to a link's `href`, or submitting a form.

```js
// Prevent the form from doing a full page reload on submit
Elements.signupForm.update({
  addEventListener: {
    submit: (e) => {
      e.preventDefault(); // Don't reload the page
      handleSignup(e);    // Handle it ourselves
    }
  }
});

// Prevent a link from navigating, so we can handle routing ourselves
Elements.navLink.update({
  addEventListener: {
    click: (e) => {
      e.preventDefault();
      Router.go(e.target.getAttribute('href'));
    }
  }
});
```

### When to use which

| Method | Stops traveling to other elements? | Stops other handlers on same element? | Cancels browser default? |
|---|---|---|---|
| `stopPropagation()` | ✅ Yes | ❌ No | ❌ No |
| `stopImmediatePropagation()` | ✅ Yes | ✅ Yes | ❌ No |
| `preventDefault()` | ❌ No | ❌ No | ✅ Yes |

They are completely independent. You can call multiple of them together when needed.

---

## Capturing — Listening Before the Event Arrives

By default, listeners fire during the bubbling phase (on the way *up*). If you want your listener to fire during the capturing phase (on the way *down*, before the event reaches the target), add `{ capture: true }`:

```js
Elements.overlay.update({
  addEventListener: ['click', handleClick, { capture: true }]
});
```

This listener fires *before* any click handler on any element inside the overlay.

### When is capturing useful?

**Click-outside to close a dropdown:**

You want to close the dropdown when the user clicks anywhere outside of it. With capture on the document, you catch every click before they reach anything else:

```js
document.addEventListener('click', (e) => {
  // If the click wasn't inside the dropdown, close it
  if (!Elements.dropdown.element.contains(e.target)) {
    Elements.dropdown.update({ classList: { remove: 'open' } });
  }
}, { capture: true });
```

**Blocking an entire disabled panel:**

You want to prevent all interaction with a panel while it's in a disabled state:

```js
Elements.disabledPanel.update({
  addEventListener: ['click', (e) => {
    // Stop and prevent everything — nothing inside gets to handle this click
    e.stopPropagation();
    e.preventDefault();
  }, { capture: true }]
});
```

**Global analytics tracking:**

You want to log every click in the entire app, even clicks on elements that call `stopPropagation()`. Because capture fires before bubbling, a listener that calls `stopPropagation()` can't block your capture-phase listener:

```js
document.addEventListener('click', (e) => {
  analytics.track('click', {
    tag:     e.target.tagName,
    id:      e.target.id,
    classes: e.target.className
  });
}, { capture: true });
// This fires for every click — even clicks "blocked" by stopPropagation elsewhere
```

---

## Delegation in Practice — A Todo List

Here's a complete example of event delegation. The HTML structure looks like this:

```html
<ul id="todo-list">
  <li data-id="1">
    <span>Buy groceries</span>
    <button data-action="complete">✓</button>
    <button data-action="delete">✗</button>
  </li>
  <!-- More items added dynamically -->
</ul>
```

Instead of attaching listeners to every button, we attach one listener to the list and let bubbling do the work:

```js
Elements.todoList.update({
  addEventListener: {
    click: (e) => {
      // What action button was clicked?
      const action = e.target.dataset.action;
      if (!action) return; // The click wasn't on an action button — ignore it

      // Find the list item this button belongs to
      const item = e.target.closest('li');
      const id   = item?.dataset.id;
      if (!item || !id) return;

      if (action === 'complete') {
        // Mark the item as done visually and update the data layer
        item.update({
          classList:    { add: 'completed' },
          setAttribute: { 'aria-checked': 'true' }
        });
        markTodoDone(id);
      }

      if (action === 'delete') {
        // Remove the item from the DOM
        item.update({ remove: [] });
        deleteTodo(id);
      }
    }
  }
});
```

Items added to the list after this code runs are handled automatically — no re-registration needed, no loops, no `querySelectorAll` calls when the data updates.

---

## Putting It All Together — A Modal

This example combines bubbling, delegation, `stopPropagation`, and a capture-phase keyboard listener into a complete modal system:

```js
// 1. Open the modal when the trigger button is clicked
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

// 2. Close if the user clicks the dark backdrop behind the modal
//    (but NOT if they click inside the modal content)
Elements.modal.update({
  addEventListener: {
    click: (e) => {
      // e.target === e.currentTarget means the click was on the backdrop itself
      // not on a child element inside the modal
      if (e.target === e.currentTarget) {
        closeModal();
      }
    }
  }
});

// 3. Stop clicks inside the modal content from reaching the backdrop above
Elements.modalContent.update({
  addEventListener: {
    click: (e) => e.stopPropagation()
  }
});

// 4. Close button inside the modal
Elements.modalCloseBtn.update({
  addEventListener: { click: closeModal }
});

// 5. Press Escape anywhere to close — capture phase ensures it fires first
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

Each technique has a specific role:
- **Bubbling** — the backdrop click listener catches bubbled clicks (step 2)
- **`stopPropagation()`** — prevents content clicks from reaching the backdrop (step 3)
- **Capture** — the Escape key listener fires before any other handler (step 5)

---

## Summary

| Concept | What to Know |
|---|---|
| **Bubbling** | Events travel up from the target through every ancestor — the default behavior |
| **Capturing** | Events travel down from `document` to the target — opt in with `{ capture: true }` |
| **Target phase** | Handlers on the target element itself fire for both phases |
| **`e.target`** | The element the user interacted with |
| **`e.currentTarget`** | The element your listener is attached to |
| **`stopPropagation()`** | Stop the event traveling to other elements (other handlers on current element still fire) |
| **`stopImmediatePropagation()`** | Stop traveling AND prevent other handlers on this element |
| **`preventDefault()`** | Cancel the browser's default action — has no effect on propagation |
| **Delegation** | One listener on a parent handles events for all children via `e.target.closest()` |

---

Next: **[Advanced Patterns & Best Practices](./04_advanced-patterns-and-best-practices)**
