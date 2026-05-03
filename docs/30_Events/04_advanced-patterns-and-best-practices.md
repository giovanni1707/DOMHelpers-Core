[![Sponsor](https://img.shields.io/badge/Sponsor-💖-pink)](https://github.com/sponsors/giovanni1707)

[![Sponsor](https://img.shields.io/badge/Sponsor-PayPal-blue?logo=paypal)](https://paypal.me/GiovanniSylvain)

# Advanced Patterns & Best Practices

This page brings together the event system into real-world compositions — accessible interactions, keyboard handling, form validation, custom events, and cleanup — showing how DOM Helpers' event model fits naturally into production-quality code.

---

## Accessible Interactive Components

### Toggle button (ARIA-aware)

A button that expands and collapses a panel, maintaining correct ARIA state throughout:

```js
let isExpanded = false;

Elements.toggleBtn.update({
  addEventListener: {
    click: () => {
      isExpanded = !isExpanded;

      Elements.update({
        toggleBtn: {
          setAttribute: { 'aria-expanded': String(isExpanded) },
          textContent:  isExpanded ? 'Collapse' : 'Expand'
        },
        panel: {
          setAttribute: { 'aria-hidden': String(!isExpanded) },
          style:        { display: isExpanded ? 'block' : 'none' }
        }
      });
    }
  }
});
```

### Focus trap inside a modal

When a modal opens, keyboard navigation should be constrained inside it. The capture phase on `keydown` intercepts Tab before any focusable child handles it:

```js
function trapFocus(e) {
  if (e.key !== 'Tab') return;

  const focusable = Elements.modal.element.querySelectorAll(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  const first = focusable[0];
  const last  = focusable[focusable.length - 1];

  if (e.shiftKey && document.activeElement === first) {
    e.preventDefault();
    last.focus();
  } else if (!e.shiftKey && document.activeElement === last) {
    e.preventDefault();
    first.focus();
  }
}

function openModal() {
  Elements.modal.update({
    classList:    { add: 'open' },
    setAttribute: { 'aria-hidden': 'false' }
  });
  document.addEventListener('keydown', trapFocus, { capture: true });
  Elements.modalCloseBtn.element?.focus();
}

function closeModal() {
  Elements.modal.update({
    classList:    { remove: 'open' },
    setAttribute: { 'aria-hidden': 'true' }
  });
  document.removeEventListener('keydown', trapFocus, { capture: true });
  Elements.openModalBtn.element?.focus(); // Return focus to trigger
}

Elements.openModalBtn.update({ addEventListener: { click: openModal } });
Elements.modalCloseBtn.update({ addEventListener: { click: closeModal } });
```

---

## Keyboard Handling

### Global hotkeys

```js
const keyMap = {
  '/':       () => Elements.searchInput.element?.focus(),
  'Escape':  () => closeActiveModal(),
  '?':       () => Elements.helpDialog.update({ classList: { toggle: 'open' } })
};

document.addEventListener('keydown', (e) => {
  // Don't fire hotkeys when user is typing in a field
  if (e.target.matches('input, textarea, select, [contenteditable]')) return;

  const handler = keyMap[e.key];
  if (handler) {
    e.preventDefault();
    handler();
  }
});
```

### Arrow-key navigation within a list

```js
Elements.optionList.update({
  addEventListener: {
    keydown: (e) => {
      const items    = [...Elements.optionList.element.querySelectorAll('[role="option"]')];
      const current  = document.activeElement;
      const index    = items.indexOf(current);

      if (e.key === 'ArrowDown') {
        e.preventDefault();
        items[Math.min(index + 1, items.length - 1)]?.focus();
      }
      if (e.key === 'ArrowUp') {
        e.preventDefault();
        items[Math.max(index - 1, 0)]?.focus();
      }
      if (e.key === 'Home') { e.preventDefault(); items[0]?.focus(); }
      if (e.key === 'End')  { e.preventDefault(); items[items.length - 1]?.focus(); }

      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        current.click();
      }
    }
  }
});
```

### Modifier keys

```js
Elements.canvas.update({
  addEventListener: {
    click: (e) => {
      if (e.metaKey || e.ctrlKey) {
        // Ctrl/Cmd + click → multi-select
        e.target.update({ classList: { toggle: 'multi-selected' } });
      } else if (e.shiftKey) {
        // Shift + click → range select
        selectRange(e.target);
      } else {
        // Plain click → single select
        clearSelection();
        e.target.update({ classList: { add: 'selected' } });
      }
    }
  }
});
```

---

## Form Validation

### Real-time field validation

```js
const validators = {
  email:    (val) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(val),
  password: (val) => val.length >= 8,
  username: (val) => /^[a-z0-9_]{3,20}$/.test(val)
};

function validateField(e) {
  const field    = e.target;
  const name     = field.name;
  const isValid  = validators[name]?.(field.value) ?? true;
  const errorEl  = document.getElementById(`${field.id}-error`);

  field.update({
    classList:    { add: isValid ? 'valid' : 'invalid', remove: isValid ? 'invalid' : 'valid' },
    setAttribute: { 'aria-invalid': String(!isValid) }
  });

  if (errorEl) {
    errorEl.update({
      textContent: isValid ? '' : getErrorMessage(name),
      style:       { display: isValid ? 'none' : 'block' }
    });
  }
}

// Validate on blur; clear error state on focus
Collections.TagName.input.update({
  addEventListener: {
    blur:  validateField,
    focus: (e) => e.target.update({ classList: { remove: 'invalid', add: 'touched' } })
  }
});
```

### Form submission with validation gate

```js
Elements.signupForm.update({
  addEventListener: {
    submit: (e) => {
      e.preventDefault();

      const inputs   = [...e.currentTarget.querySelectorAll('input[required]')];
      const allValid = inputs.every(input => input.value.trim().length > 0);

      if (!allValid) {
        inputs
          .filter(input => !input.value.trim())
          .forEach(input => {
            input.update({
              classList:    { add: 'invalid' },
              setAttribute: { 'aria-invalid': 'true' }
            });
          });

        Elements.formError.update({
          textContent: 'Please fill in all required fields.',
          style:       { display: 'block' }
        });
        return;
      }

      submitSignup(new FormData(e.currentTarget));
    }
  }
});
```

---

## Custom Events

### Dispatching custom events

```js
// Dispatch a custom event from a button handler
Elements.addItemBtn.update({
  addEventListener: {
    click: (e) => {
      const newItem = createItem();

      // Notify the rest of the app
      e.target.element?.dispatchEvent(
        new CustomEvent('item:added', {
          bubbles: true,
          detail:  { item: newItem }
        })
      );
    }
  }
});
```

Using `.update()` with `dispatchEvent` as a DOM method call:

```js
Elements.addItemBtn.update({
  dispatchEvent: [new CustomEvent('item:added', { bubbles: true, detail: { id: 42 } })]
});
```

### Listening for custom events

Custom events bubble like any native event, so you can delegate on a container:

```js
Elements.appRoot.update({
  addEventListener: {
    'item:added': (e) => {
      renderItem(e.detail.item);
      Elements.itemCount.update({ textContent: String(++itemCount) });
    },
    'item:removed': (e) => {
      removeItemFromUI(e.detail.id);
      Elements.itemCount.update({ textContent: String(--itemCount) });
    }
  }
});
```

---

## Pointer and Touch Events

### Drag detection

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
      e.currentTarget.setPointerCapture(e.pointerId);
    },
    pointermove: (e) => {
      if (!isDragging) return;
      const dx = e.clientX - startX;
      const dy = e.clientY - startY;
      e.target.update({
        style: { transform: `translate(${dx}px, ${dy}px)` }
      });
    },
    pointerup: (e) => {
      isDragging = false;
      e.target.update({
        classList: { remove: 'dragging' }
      });
    }
  }
});
```

### Preventing scroll interference on touch

```js
Elements.slider.update({
  addEventListener: ['touchmove', (e) => {
    e.preventDefault(); // Prevent page scroll during slider drag
    handleSliderMove(e);
  }, { passive: false }]  // passive: false is required to call preventDefault on touch
});
```

---

## Listener Lifecycle and Cleanup

### Pattern: setup and teardown functions

Group related listeners so they can be added and removed as a unit:

```js
function handleKeydown(e) { /* ... */ }
function handleClick(e)   { /* ... */ }
function handleResize()   { /* ... */ }

function attachListeners() {
  document.addEventListener('keydown', handleKeydown, { capture: true });
  window.addEventListener('resize', handleResize, { passive: true });
  Elements.panel.update({
    addEventListener: { click: handleClick }
  });
}

function detachListeners() {
  document.removeEventListener('keydown', handleKeydown, { capture: true });
  window.removeEventListener('resize', handleResize);
  Elements.panel.update({
    removeEventListener: ['click', handleClick]
  });
}

// Attach when a section becomes active, detach when it's hidden
Elements.openSectionBtn.update({
  addEventListener: { click: attachListeners }
});
Elements.closeSectionBtn.update({
  addEventListener: { click: detachListeners }
});
```

### Pattern: AbortController for grouped cleanup

When many listeners share the same lifecycle, a single `AbortController` can remove all of them at once:

```js
let sessionController = null;

function startSession() {
  sessionController = new AbortController();
  const signal = sessionController.signal;

  Elements.pauseBtn.update({
    addEventListener: ['click', pauseSession, { signal }]
  });
  Elements.stopBtn.update({
    addEventListener: ['click', stopSession, { signal }]
  });
  document.addEventListener('visibilitychange', handleVisibility, { signal });
  window.addEventListener('beforeunload', saveSessionState, { signal });
}

function endSession() {
  sessionController?.abort(); // Removes ALL listeners registered with this signal
  sessionController = null;
}
```

### Pattern: `{ once: true }` for self-removing listeners

When a handler should fire exactly once and never again, `{ once: true }` is cleaner than manually removing inside the handler:

```js
// Runs setup animation exactly once, then removes itself
Elements.hero.update({
  addEventListener: ['transitionend', () => {
    Elements.hero.update({ classList: { add: 'settled' } });
  }, { once: true }]
});

// One-time consent dialog
Elements.acceptBtn.update({
  addEventListener: ['click', () => {
    localStorage.setItem('consent', 'granted');
    Elements.consentBanner.update({ style: { display: 'none' } });
  }, { once: true }]
});
```

---

## Performance Guidelines

### Use `passive: true` for scroll and touch

Marking scroll and touch listeners as passive tells the browser it can run them in parallel with painting, eliminating jank caused by waiting to see if `preventDefault()` will be called:

```js
// Always passive for scroll tracking
Elements.scrollArea.update({
  addEventListener: ['scroll', updateScrollIndicator, { passive: true }]
});

// Always passive for touch tracking (unless you need preventDefault)
Elements.swipeArea.update({
  addEventListener: ['touchstart', startSwipe, { passive: true }]
});
```

### Delegate instead of attaching to every child

| Approach | Listeners created | Dynamic elements covered |
|---|---|---|
| One per child | N (one per element) | No — must re-run setup |
| Delegation on parent | 1 | Yes — automatically |

```js
// ❌ One listener per item — does not scale, breaks on dynamic additions
document.querySelectorAll('.product').forEach(el => {
  el.addEventListener('click', handleProductClick);
});

// ✅ One delegated listener on the container
Elements.productGrid.update({
  addEventListener: {
    click: (e) => {
      const product = e.target.closest('.product');
      if (product) handleProductClick({ target: product });
    }
  }
});
```

### Debounce high-frequency events

For events that fire many times per second (resize, scroll, input), run the heavy work less frequently:

```js
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

const onResize = debounce(() => {
  Elements.layout.update({ dataset: { width: String(window.innerWidth) } });
}, 150);

window.addEventListener('resize', onResize, { passive: true });
```

---

## Event Handling — Decision Guide

Use this to pick the right approach for each situation:

```
Need to listen on one known element?
  └─ Elements.id.update({ addEventListener: ... })

Need to listen on multiple elements of the same class/tag/name?
  └─ Collections.ClassName.x.update({ addEventListener: ... })

Elements added to the DOM dynamically?
  └─ Delegate on a stable ancestor using e.target.closest()

Need to intercept before any child handler fires?
  └─ { capture: true }

Handler should fire exactly once?
  └─ { once: true }

Scroll or touch handler that doesn't call preventDefault()?
  └─ { passive: true }

Many listeners share a lifecycle (page, session, modal)?
  └─ AbortController with { signal }

Need to remove a listener later?
  └─ Store the handler in a named variable — never inline arrows
```

---

## Summary

| Scenario | Recommended Pattern |
|---|---|
| Multiple events on one element | Object format `{ click: fn, focus: fn }` |
| Events on a group of elements | `Collections.ClassName.x.update({ addEventListener: ... })` |
| Dynamic child elements | Delegate on ancestor + `e.target.closest()` |
| One-time listeners | `{ once: true }` option |
| Scroll / touch performance | `{ passive: true }` option |
| Blocking child subtree | `{ capture: true }` + `e.stopPropagation()` |
| Removing listeners by lifecycle | `AbortController` + `{ signal }` |
| Accessible keyboard nav | `keydown` + phase-aware `e.key` checks |
| Preventing default browser action | `e.preventDefault()` inside the handler |
| Custom event system | `dispatchEvent` + bubbling delegation on root |
