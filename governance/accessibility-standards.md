# Accessibility Standards

> Authoritative accessibility standards aligned with WCAG 2.1 AA and industry best practices.

## Purpose

Establish accessibility practices that ensure applications are usable by people with disabilities, comply with legal requirements, and provide a better experience for all users.

## Core Principles

1. **Perceivable** - Information must be presentable in ways users can perceive
2. **Operable** - Interface components must be operable by all users
3. **Understandable** - Information and UI operation must be understandable
4. **Robust** - Content must be robust enough for assistive technologies

## WCAG 2.1 AA Compliance Overview

| Principle | Guidelines | Key Requirements |
| --------- | ---------- | ---------------- |
| Perceivable | Text alternatives, captions, adaptable content | Alt text, transcripts, responsive |
| Operable | Keyboard access, enough time, no seizures | Tab navigation, focus management |
| Understandable | Readable, predictable, input assistance | Clear language, consistent UI |
| Robust | Compatible with assistive tech | Valid HTML, ARIA when needed |

## Keyboard Navigation

### Requirements

```typescript
// All interactive elements must be keyboard accessible
// Focus order must be logical
// Focus must be visible

// Good: Native elements are keyboard accessible by default
<button onClick={handleClick}>Submit</button>
<a href="/page">Link</a>
<input type="text" />

// Bad: Div with click handler - NOT keyboard accessible
<div onClick={handleClick}>Submit</div>

// If you must use non-semantic elements, add keyboard support:
<div
  role="button"
  tabIndex={0}
  onClick={handleClick}
  onKeyDown={(e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      handleClick();
    }
  }}
>
  Submit
</div>
```

### Focus Management

```typescript
// Skip links for main content
<a href="#main-content" className="skip-link">
  Skip to main content
</a>

// Focus trap for modals
function Modal({ isOpen, onClose, children }) {
  const modalRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (isOpen) {
      // Save previous focus
      const previousFocus = document.activeElement;

      // Focus modal
      modalRef.current?.focus();

      return () => {
        // Restore focus on close
        (previousFocus as HTMLElement)?.focus();
      };
    }
  }, [isOpen]);

  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      tabIndex={-1}
    >
      {children}
    </div>
  );
}
```

### Focus Visibility

```css
/* Never remove focus outline without replacement */
/* Bad */
*:focus {
  outline: none;
}

/* Good - Custom visible focus */
*:focus {
  outline: 2px solid #005fcc;
  outline-offset: 2px;
}

/* Better - Use focus-visible for keyboard-only focus */
*:focus:not(:focus-visible) {
  outline: none;
}

*:focus-visible {
  outline: 2px solid #005fcc;
  outline-offset: 2px;
}
```

## Semantic HTML

### Use Correct Elements

```html
<!-- Good: Semantic structure -->
<header>
  <nav aria-label="Main navigation">
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/about">About</a></li>
    </ul>
  </nav>
</header>

<main id="main-content">
  <article>
    <h1>Page Title</h1>
    <section aria-labelledby="section-heading">
      <h2 id="section-heading">Section Title</h2>
      <p>Content...</p>
    </section>
  </article>
</main>

<footer>
  <p>&copy; 2024</p>
</footer>

<!-- Bad: Div soup -->
<div class="header">
  <div class="nav">
    <div class="nav-item">Home</div>
  </div>
</div>
```

### Heading Hierarchy

```html
<!-- Good: Logical heading hierarchy -->
<h1>Main Page Title</h1>
  <h2>Section 1</h2>
    <h3>Subsection 1.1</h3>
    <h3>Subsection 1.2</h3>
  <h2>Section 2</h2>

<!-- Bad: Skipped levels -->
<h1>Title</h1>
<h4>Subsection</h4>  <!-- Skipped h2, h3 -->
```

## Images and Media

### Alt Text

```typescript
// Informative images - describe the content
<img
  src="/chart.png"
  alt="Sales increased 25% from Q1 to Q2 2024"
/>

// Decorative images - empty alt
<img src="/decoration.png" alt="" />

// Complex images - provide detailed description
<figure>
  <img
    src="/infographic.png"
    alt="Company growth infographic"
    aria-describedby="infographic-desc"
  />
  <figcaption id="infographic-desc">
    Detailed description of the infographic...
  </figcaption>
</figure>

// Icons with meaning - provide label
<button aria-label="Close dialog">
  <CloseIcon aria-hidden="true" />
</button>

// Icons that are decorative
<span>
  <StarIcon aria-hidden="true" />
  4.5 out of 5 stars
</span>
```

### Video and Audio

```html
<!-- Video with captions and audio description -->
<video controls>
  <source src="video.mp4" type="video/mp4" />
  <track
    kind="captions"
    src="captions.vtt"
    srclang="en"
    label="English"
    default
  />
  <track
    kind="descriptions"
    src="descriptions.vtt"
    srclang="en"
    label="English audio descriptions"
  />
</video>

<!-- Audio with transcript -->
<audio controls>
  <source src="podcast.mp3" type="audio/mpeg" />
</audio>
<a href="transcript.html">Read transcript</a>
```

## Forms

### Labels and Instructions

```typescript
// Every input needs a label
<label htmlFor="email">Email address</label>
<input
  type="email"
  id="email"
  name="email"
  aria-describedby="email-hint"
  required
/>
<span id="email-hint" className="hint">
  We'll never share your email
</span>

// Group related fields
<fieldset>
  <legend>Shipping Address</legend>
  <label htmlFor="street">Street</label>
  <input type="text" id="street" name="street" />
  {/* More fields... */}
</fieldset>

// Required field indication
<label htmlFor="name">
  Name <span aria-hidden="true">*</span>
  <span className="visually-hidden">(required)</span>
</label>
<input type="text" id="name" required aria-required="true" />
```

### Error Handling

```typescript
// Accessible form validation
function FormField({ label, error, ...props }) {
  const inputId = useId();
  const errorId = `${inputId}-error`;

  return (
    <div>
      <label htmlFor={inputId}>{label}</label>
      <input
        id={inputId}
        aria-invalid={!!error}
        aria-describedby={error ? errorId : undefined}
        {...props}
      />
      {error && (
        <span id={errorId} role="alert" className="error">
          {error}
        </span>
      )}
    </div>
  );
}

// Error summary at form level
<div role="alert" aria-live="polite">
  <h2>Please fix the following errors:</h2>
  <ul>
    <li><a href="#email">Email is required</a></li>
    <li><a href="#password">Password must be at least 8 characters</a></li>
  </ul>
</div>
```

## Color and Contrast

### Contrast Requirements

| Element | Minimum Ratio (AA) | Enhanced Ratio (AAA) |
| ------- | ------------------ | -------------------- |
| Normal text (< 18px) | 4.5:1 | 7:1 |
| Large text (≥ 18px or 14px bold) | 3:1 | 4.5:1 |
| UI components | 3:1 | - |
| Non-text content | 3:1 | - |

### Color Independence

```css
/* Don't rely solely on color */

/* Bad: Only color indicates state */
.error { color: red; }
.success { color: green; }

/* Good: Color + icon/text */
.error {
  color: #d32f2f;
  &::before {
    content: "⚠ Error: ";
  }
}

.success {
  color: #388e3c;
  &::before {
    content: "✓ Success: ";
  }
}
```

## ARIA Usage

### ARIA Principles

1. **First rule**: Don't use ARIA if native HTML works
2. **Second rule**: Don't change native semantics unless necessary
3. **Third rule**: All ARIA controls must be keyboard accessible
4. **Fourth rule**: Don't use `role="presentation"` on focusable elements
5. **Fifth rule**: All interactive elements must have accessible names

### Common ARIA Patterns

```typescript
// Tabs
<div role="tablist" aria-label="Settings tabs">
  <button
    role="tab"
    id="tab-1"
    aria-selected={activeTab === 0}
    aria-controls="panel-1"
  >
    General
  </button>
  <button
    role="tab"
    id="tab-2"
    aria-selected={activeTab === 1}
    aria-controls="panel-2"
  >
    Security
  </button>
</div>
<div
  role="tabpanel"
  id="panel-1"
  aria-labelledby="tab-1"
  hidden={activeTab !== 0}
>
  General settings content
</div>

// Live regions for dynamic content
<div
  role="status"
  aria-live="polite"
  aria-atomic="true"
>
  {statusMessage}
</div>

// Alert for important messages
<div role="alert">
  Your session will expire in 5 minutes
</div>
```

## Testing Accessibility

### Automated Testing

```typescript
// Jest + jest-axe
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

test('page has no accessibility violations', async () => {
  const { container } = render(<MyComponent />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});

// Cypress + cypress-axe
describe('Accessibility', () => {
  it('has no detectable violations', () => {
    cy.visit('/');
    cy.injectAxe();
    cy.checkA11y();
  });
});
```

### Manual Testing Checklist

- [ ] Navigate entire page with keyboard only
- [ ] Test with screen reader (VoiceOver, NVDA, JAWS)
- [ ] Check zoom to 200% - content still usable
- [ ] Disable CSS - content still makes sense
- [ ] Test with high contrast mode
- [ ] Check all color contrast ratios

## Component Checklist

### Buttons

- [ ] Uses `<button>` element (or has `role="button"`)
- [ ] Has accessible name (text content or `aria-label`)
- [ ] Disabled state uses `disabled` attribute (not just styling)
- [ ] Loading state announced to screen readers

### Links

- [ ] Uses `<a>` element with `href`
- [ ] Link text describes destination (not "click here")
- [ ] Opens in new tab? Include warning
- [ ] Visited/unvisited states visually distinct

### Forms

- [ ] All inputs have associated labels
- [ ] Required fields indicated
- [ ] Error messages associated with inputs
- [ ] Focus moves logically through form

### Modals

- [ ] Focus trapped within modal
- [ ] Can close with Escape key
- [ ] Focus returns to trigger on close
- [ ] Has `role="dialog"` and `aria-modal="true"`

## Checklist

### Development Process

- [ ] Accessibility requirements in acceptance criteria
- [ ] Semantic HTML used throughout
- [ ] Keyboard navigation tested
- [ ] Screen reader tested
- [ ] Automated accessibility tests in CI

### Compliance

- [ ] WCAG 2.1 AA criteria met
- [ ] Color contrast verified
- [ ] All images have appropriate alt text
- [ ] Forms are fully accessible
- [ ] Dynamic content announced

### Documentation

- [ ] Accessibility guidelines documented
- [ ] Component accessibility notes
- [ ] Testing procedures documented

## References

- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [MDN Accessibility](https://developer.mozilla.org/en-US/docs/Web/Accessibility)
- [Deque University](https://dequeuniversity.com/)
- [A11y Project Checklist](https://www.a11yproject.com/checklist/)
- [Inclusive Components](https://inclusive-components.design/)
- [axe DevTools](https://www.deque.com/axe/)
