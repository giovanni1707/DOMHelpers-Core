[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Advanced Patterns & Best Practices

This page is about putting everything together. You've learned how listeners work, how events travel, and how to control that journey. Now let's look at the patterns that show up again and again in real applications.

Each section below is something you can adapt directly to your own projects.

---

## Building Accessible Interactive Components

### Expand/Collapse Toggle

A button that shows or hides a panel. The key here is keeping ARIA attributes in sync with the visual state so screen readers always know what's happening:

```js
let isExpanded = false;

Elements.toggleBtn.update({
  addEventListener: {
    click: () => {
      isExpanded = !isExpanded;

      // Update both the button and the panel in one call
      Elements.update({
        toggleBtn: {
          // aria-expanded tells screen readers whether the panel is open
          setAttribute: { 'aria-expanded': String(isExpanded) },
          textContent:  isExpanded ? 'Collapse' : 'Expand'
        },
        panel: {
          // aria-hidden removes the panel from the accessibility tree when closed
          setAttribute: { 'aria-hidden': String(!isExpanded) },
          style:        { display: isExpanded ? 'block' : 'none' }
        }
      });
    }
  }
});
```

### Focus Trap in a Modal

When a modal is open, pressing Tab should cycle through elements *inside* the modal — not jump to things behind it. This is a standard accessibility requirement.

The trick is using a capture-phase `keydown` listener on the document. Because it fires before anything else, we can intercept Tab before the browser moves focus:

```js
function trapFocus(e) {
  if (e.key !== 'Tab') return; // We only care about Tab

  // Collect every focusable element inside the modal
  const focusable = Elements.modal.element.querySelectorAll(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  const first = focusable[0];
  const last  = focusable[focusable.length - 1];

  if (e.shiftKey && document.activeElement === first) {
    // Shift+Tab on the first element — jump to the last
    e.preventDefault();
    last.focus();
  } else if (!e.shiftKey && document.activeElement === last) {
    // Tab on the last element — jump to the first
    e.preventDefault();
    first.focus();
  }
  // Otherwise let Tab move focus normally within the modal
}

function openModal() {
  Elements.modal.update({
    classList:    { add: 'open' },
    setAttribute: { 'aria-hidden': 'false' }
  });
  document.addEventListener('keydown', trapFocus, { capture: true });
  Elements.modalCloseBtn.element?.focus(); // Move focus into the modal
}

function closeModal() {
  Elements.modal.update({
    classList:    { remove: 'open' },
    setAttribute: { 'aria-hidden': 'true' }
  });
  document.removeEventListener('keydown', trapFocus, { capture: true });
  Elements.openModalBtn.element?.focus(); // Return focus to where it came from
}

Elements.openModalBtn.update({ addEventListener: { click: openModal } });
Elements.modalCloseBtn.update({ addEventListener: { click: closeModal } });
```

The listener is added when the modal opens and removed when it closes — clean and scoped to exactly when it's needed.

---

## Keyboard Handling

### Global Keyboard Shortcuts

Let users trigger common actions with keyboard shortcuts, without interfering when they're typing in a field:

```js
const shortcuts = {
  '/':      () => Elements.searchInput.element?.focus(),  // Open search
  'Escape': () => closeActiveModal(),                      // Close anything open
  '?':      () => Elements.helpPanel.update({ classList: { toggle: 'open' } })
};

document.addEventListener('keydown', (e) => {
  // If focus is inside a text field, don't hijack keypresses
  if (e.target.matches('input, textarea, select, [contenteditable]')) return;

  const action = shortcuts[e.key];
  if (action) {
    e.preventDefault(); // Don't type the character into a field that gets focus next
    action();
  }
});
```

### Arrow Key Navigation in a List

Custom dropdowns, option pickers, and menus should respond to arrow keys. Here's a pattern that works for any list of focusable items:

```js
Elements.optionList.update({
  addEventListener: {
    keydown: (e) => {
      // Collect every option in the list
      const items   = [...Elements.optionList.element.querySelectorAll('[role="option"]')];
      const current = document.activeElement;
      const index   = items.indexOf(current);

      // Move down — but don't go past the last item
      if (e.key === 'ArrowDown') {
        e.preventDefault();
        items[Math.min(index + 1, items.length - 1)]?.focus();
      }

      // Move up — but don't go past the first item
      if (e.key === 'ArrowUp') {
        e.preventDefault();
        items[Math.max(index - 1, 0)]?.focus();
      }

      // Jump to start or end
      if (e.key === 'Home') { e.preventDefault(); items[0]?.focus(); }
      if (e.key === 'End')  { e.preventDefault(); items[items.length - 1]?.focus(); }

      // Select the focused item with Enter or Space
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        current.click(); // Trigger the item's click handler
      }
    }
  }
});
```

### Modifier Keys (Ctrl, Shift, Meta)

Detecting combinations like Ctrl+Click or Shift+Click:

```js
Elements.canvas.update({
  addEventListener: {
    click: (e) => {
      if (e.metaKey || e.ctrlKey) {
        // Ctrl/Cmd + click → add this item to the selection
        e.target.update({ classList: { toggle: 'multi-selected' } });

      } else if (e.shiftKey) {
        // Shift + click → select a range
        selectRange(e.target);

      } else {
        // Plain click → clear other selections and select just this one
        clearSelection();
        e.target.update({ classList: { add: 'selected' } });
      }
    }
  }
});
```

---

## Form Validation

### Validate as the User Types (or Leaves a Field)

Showing errors on `blur` (when the user leaves a field) is friendlier than showing them immediately while typing. Clearing the error on `focus` (when they return to fix it) gives fast feedback:

```js
// Define validation rules for each field by name
const validators = {
  email:    (val) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(val),
  password: (val) => val.length >= 8,
  username: (val) => /^[a-z0-9_]{3,20}$/.test(val)
};

function validateField(e) {
  const field   = e.target;
  const isValid = validators[field.name]?.(field.value) ?? true;
  const errorEl = document.getElementById(`${field.id}-error`);

  // Update the field's appearance and accessibility state
  field.update({
    classList:    { add: isValid ? 'valid' : 'invalid', remove: isValid ? 'invalid' : 'valid' },
    setAttribute: { 'aria-invalid': String(!isValid) }
  });

  // Show or hide the error message
  if (errorEl) {
    errorEl.update({
      textContent: isValid ? '' : getErrorMessage(field.name),
      style:       { display: isValid ? 'none' : 'block' }
    });
  }
}

// Apply to every input — validate on blur, clear error highlight on focus
Collections.TagName.input.update({
  addEventListener: {
    blur:  validateField,
    focus: (e) => e.target.update({ classList: { remove: 'invalid', add: 'touched' } })
  }
});
```

### Block Submission Until the Form Is Valid

```js
Elements.signupForm.update({
  addEventListener: {
    submit: (e) => {
      e.preventDefault(); // Always prevent the default page reload

      // Find every required input and check if it has a value
      const inputs   = [...e.currentTarget.querySelectorAll('input[required]')];
      const allFilled = inputs.every(input => input.value.trim().length > 0);

      if (!allFilled) {
        // Highlight the empty fields
        inputs
          .filter(input => !input.value.trim())
          .forEach(input => {
            input.update({
              classList:    { add: 'invalid' },
              setAttribute: { 'aria-invalid': 'true' }
            });
          });

        // Show a general error at the top of the form
        Elements.formError.update({
          textContent: 'Please fill in all required fields.',
          style:       { display: 'block' }
        });
        return; // Stop here — don't submit
      }

      // All good — submit
      submitSignup(new FormData(e.currentTarget));
    }
  }
});
```

---

## Custom Events

Sometimes you want different parts of your app to communicate without tight coupling. Custom events are a great tool for this — one part dispatches an event, another part listens for it, and they don't need to know about each other.

### Dispatching a Custom Event

```js
Elements.addItemBtn.update({
  addEventListener: {
    click: () => {
      const newItem = createItem();

      // Tell the rest of the app something happened
      // bubbles: true means the event will bubble up through the DOM
      // detail carries the data you want to pass along
      Elements.addItemBtn.element?.dispatchEvent(
        new CustomEvent('item:added', {
          bubbles: true,
          detail:  { item: newItem }
        })
      );
    }
  }
});
```

You can also dispatch events directly through `.update()`:

```js
Elements.addItemBtn.update({
  dispatchEvent: [new CustomEvent('item:added', { bubbles: true, detail: { id: 42 } })]
});
```

### Listening for Custom Events

Because custom events bubble just like native ones, you can use delegation — one listener on a root element catches events from anywhere in the subtree:

```js
Elements.appRoot.update({
  addEventListener: {
    // Listen for item:added events bubbling up from anywhere inside #app-root
    'item:added': (e) => {
      renderItem(e.detail.item); // e.detail carries the data you passed
      Elements.itemCount.update({ textContent: String(++itemCount) });
    },
    'item:removed': (e) => {
      removeItemFromUI(e.detail.id);
      Elements.itemCount.update({ textContent: String(--itemCount) });
    }
  }
});
```

This pattern keeps your code loosely coupled — the button doesn't need to know about the item count display, and the display doesn't need to know what triggers updates.

---

## Pointer and Touch Events

### Drag and Drop

The Pointer Events API (`pointerdown`, `pointermove`, `pointerup`) is the modern way to handle dragging — it works consistently across mouse, touch, and stylus:

```js
let isDragging = false;
let startX, startY;

Elements.draggable.update({
  addEventListener: {
    pointerdown: (e) => {
      isDragging = true;
      startX = e.clientX;
      startY = e.clientY;
      e.target.update({ classList: { add: 'dragging' } });

      // Capture the pointer so we keep receiving events even if the cursor
      // moves outside the element before pointerup fires
      e.currentTarget.setPointerCapture(e.pointerId);
    },

    pointermove: (e) => {
      if (!isDragging) return;
      const dx = e.clientX - startX; // How far we've moved horizontally
      const dy = e.clientY - startY; // How far we've moved vertically
      e.target.update({
        style: { transform: `translate(${dx}px, ${dy}px)` }
      });
    },

    pointerup: (e) => {
      isDragging = false;
      e.target.update({ classList: { remove: 'dragging' } });
    }
  }
});
```

### Prevent Scroll Interference During Touch

For sliders, carousels, or any touch-controlled element, you need to call `preventDefault()` on `touchmove` to stop the page from scrolling while the user is dragging. But this requires `passive: false` — otherwise the browser ignores `preventDefault()` on touch events:

```js
Elements.slider.update({
  addEventListener: ['touchmove', (e) => {
    e.preventDefault(); // Stop the page from scrolling while we handle the drag
    handleSliderMove(e);
  }, { passive: false }] // passive: false is required to allow preventDefault on touch
});
```

---

## Listener Cleanup

A listener that isn't cleaned up when it's no longer needed is a memory leak waiting to happen. Here are three patterns for keeping your listeners tidy.

### Pattern 1 — Named setup and teardown functions

Group related listeners into functions you can call to attach or remove them as a set:

```js
function handleKeydown(e) { /* ... */ }
function handleClick(e)   { /* ... */ }
function handleResize()   { /* ... */ }

function attachListeners() {
  document.addEventListener('keydown', handleKeydown, { capture: true });
  window.addEventListener('resize', handleResize, { passive: true });
  Elements.panel.update({ addEventListener: { click: handleClick } });
}

function detachListeners() {
  document.removeEventListener('keydown', handleKeydown, { capture: true });
  window.removeEventListener('resize', handleResize);
  Elements.panel.update({ removeEventListener: ['click', handleClick] });
}

// Attach when a section opens, detach when it closes
Elements.openSectionBtn.update({  addEventListener: { click: attachListeners } });
Elements.closeSectionBtn.update({ addEventListener: { click: detachListeners } });
```

### Pattern 2 — AbortController for group removal

When a group of listeners all share the same lifecycle, one `AbortController` can remove all of them in a single line. Think of it like a shared off-switch:

```js
let controller = null;

function startSession() {
  controller = new AbortController();
  const { signal } = controller; // Pass this signal to every listener

  Elements.pauseBtn.update({
    addEventListener: ['click', pauseSession, { signal }]
  });
  Elements.stopBtn.update({
    addEventListener: ['click', stopSession, { signal }]
  });
  document.addEventListener('visibilitychange', handleVisibility, { signal });
  window.addEventListener('beforeunload', saveState, { signal });
}

function endSession() {
  controller?.abort(); // All four listeners removed instantly
  controller = null;
}
```

### Pattern 3 — `{ once: true }` for self-cleanup

When a handler should only ever fire one time, `{ once: true }` is the cleanest approach — the listener removes itself automatically after firing:

```js
// After the intro animation finishes, add the 'settled' class — once only
Elements.hero.update({
  addEventListener: ['animationend', () => {
    Elements.hero.update({ classList: { add: 'settled' } });
  }, { once: true }]
});

// Show the consent banner action only once
Elements.acceptCookiesBtn.update({
  addEventListener: ['click', () => {
    localStorage.setItem('cookiesAccepted', 'true');
    Elements.cookieBanner.update({ style: { display: 'none' } });
  }, { once: true }]
});
```

---

## Performance Tips

### Always use `passive: true` for scroll and touch

When you attach a scroll or touch listener, the browser has to wait and see if your handler calls `preventDefault()` before it can paint. If your handler doesn't need to block scrolling, telling the browser upfront with `passive: true` removes that wait:

```js
// Good for any scroll tracker that doesn't need to block scrolling
Elements.page.update({
  addEventListener: ['scroll', updateScrollIndicator, { passive: true }]
});

// Good for touch tracking that doesn't prevent default touch behavior
Elements.swipeArea.update({
  addEventListener: ['touchstart', startSwipe, { passive: true }]
});
```

### Prefer delegation over per-child listeners

| Approach | Number of listeners | Works for dynamically added elements? |
|---|---|---|
| One listener per child | N | ❌ No — must re-run setup |
| One delegated listener on parent | 1 | ✅ Yes — always |

```js
// ❌ Attaches a listener to every existing product — breaks when new ones are added
document.querySelectorAll('.product').forEach(el => {
  el.addEventListener('click', handleProductClick);
});

// ✅ One listener handles all products, including ones added later
Elements.productGrid.update({
  addEventListener: {
    click: (e) => {
      const product = e.target.closest('.product');
      if (product) handleProductClick(product);
    }
  }
});
```

### Debounce events that fire very frequently

Some events like `resize`, `scroll`, and `input` can fire dozens of times per second. If your handler does something expensive (like re-calculating a layout), running it on every single event will slow things down. A debounce function delays the work until the events stop coming in:

```js
// A simple debounce — waits until things settle down before running fn
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

// The actual work runs at most once every 150ms, not 60+ times per second
const onResize = debounce(() => {
  Elements.layout.update({ dataset: { width: String(window.innerWidth) } });
}, 150);

window.addEventListener('resize', onResize, { passive: true });
```

---

## Choosing the Right Approach

Use this as a quick reference when you're deciding how to handle an event:

```
One known element?
  → Elements.id.update({ addEventListener: ... })

Multiple elements of the same class, tag, or name?
  → Collections.ClassName.x.update({ addEventListener: ... })

Elements added to the DOM dynamically?
  → Delegate on a stable ancestor — one listener, e.target.closest()

Need to intercept before any child handler fires?
  → { capture: true }

Handler should only fire once?
  → { once: true }

Scroll or touch event that won't call preventDefault()?
  → { passive: true }

Group of listeners that all end at the same time?
  → AbortController with { signal }

Listener you'll need to remove later?
  → Save the handler in a named variable (never inline arrows)
```

---

## Summary

| Scenario | What to Use |
|---|---|
| Multiple events on one element | Object format: `{ click: fn, focus: fn }` |
| Events on a group of elements | `Collections.ClassName.x.update({ addEventListener: ... })` |
| Dynamically added elements | Delegate on a stable ancestor with `e.target.closest()` |
| One-time listeners | `{ once: true }` |
| Scroll / touch that doesn't block | `{ passive: true }` |
| Intercept before children see the event | `{ capture: true }` |
| Remove a group of listeners cleanly | `AbortController` + `{ signal }` |
| Keyboard shortcuts | `keydown` on `document`, check `e.key` and modifier keys |
| Preventing page reload on form submit | `e.preventDefault()` in the `submit` handler |
| Loose coupling between parts of the app | `CustomEvent` with `bubbles: true` + delegation on a root element |
