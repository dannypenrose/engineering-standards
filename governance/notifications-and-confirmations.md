# Notifications & Confirmations

> Authoritative standards for user feedback: toasts, confirmation dialogs, and inline alerts across all web applications.

## Purpose

Every user-facing action must produce timely, consistent, and accessible feedback. Inconsistent patterns (native `alert()`, stray `window.confirm()`, snackbars in different corners) confuse users, erode trust, and hide bugs — if a user can't tell whether their action succeeded, they'll repeat it or escalate.

This document defines which tool to reach for, how to style it, and what to say. It is mandatory for all web frontends.

## The Three Tools

Use exactly one of these patterns for user feedback. Do not mix, and do not invent alternatives.

| Tool | Purpose | Blocking? | Lifetime |
| ---- | ------- | --------- | -------- |
| **Toast** (top-centre, below header, auto-dismiss) | Confirm a completed action or surface a non-blocking error | No | 3 seconds |
| **Confirm dialog** (centre modal) | Ask the user to approve a destructive or irreversible action | Yes | Until user responds |
| **Inline alert** (in-page banner) | Persistent state the user needs to see until they fix it (validation summary, degraded service, permission denied) | No | Until condition clears |

Native `window.alert()`, `window.confirm()`, and `window.prompt()` are **forbidden**. They block the main thread, can't be styled, fail accessibility audits, and look like browser chrome rather than the application.

## Toasts

### Library & Mount

- **Library**: `react-hot-toast` (default) or `notistack` if migrating from an existing notistack setup. Pick one per app and stick to it.
- **Mount**: a single `<Toaster />` (or `<SnackbarProvider />`) in the root layout, inside the theme provider.
- **Position**: `top-centre`, with a vertical offset that places the toast clear of the application header and navigation (≈96 px on a standard-height top bar). This keeps the toast visible without colliding with the menu icons in the top-right of the header. No exceptions — do not use top-right (collides with header controls), bottom-centre (users don't look there), or per-feature overrides.
- **Duration**: 3000 ms (3 seconds). Long enough to read a short message, short enough not to annoy.
- **Max visible**: 5. Prevents stack overflow during batch operations.
- **Dedupe**: identical messages must not stack — collapse duplicates.

```tsx
// app/layout.tsx (Next.js) or equivalent root layout
import { Toaster } from 'react-hot-toast';

<Toaster
  position="top-center"
  containerStyle={{ top: 96 }}
  toastOptions={{ duration: 3000 }}
/>
```

Adjust the `top` offset to match the app's header height. The goal is for the toast to land just below the header, centred, so it never overlaps navigation controls.

### Variants

| Variant | Use for | Examples |
| ------- | ------- | -------- |
| `success` | CRUD action completed, export generated, toggle flipped | "Client updated", "Report exported" |
| `error` | Server error, operation failed, validation rejection | "Failed to delete contract", "Session expired" |
| `warning` | Degraded state, rate limit approaching, partial success | "Saved as draft — connection unstable" |
| `info` | Neutral status, tip, background job queued | "Sync started in background" |

Do **not** default everything to `success`. An error path that shows a green toast is worse than no feedback.

### Message Format

- **Short and specific.** One noun + one verb in past tense, optionally qualified.
- **Sentence case**, no trailing period, no shouting.
- **Action + Result**. `"<Thing> <action>ed"` for success, `"Failed to <action> <thing>"` for errors.
- **No technical details.** Never expose stack traces, error codes, or backend error strings verbatim.

#### Good

```
Client updated
Report "Q3 Sales" deleted
Failed to export PDF
Invitation email sent
```

#### Bad

```
Success!                          // too vague
Error: 500 Internal Server Error  // technical leak
The operation completed.          // unnecessary period, vague
CLIENT DELETED                    // shouting
You have successfully updated…    // too verbose
```

### Required After CRUD

Every create, update, delete, and toggle action must emit a toast on both the success and error paths. A silent CRUD action is a bug, not an aesthetic choice — users need to know the click took effect. If the action navigates away on success, the toast still fires before the navigation.

```tsx
try {
  await userApi.deleteUser(userId);
  toast.success('User deleted');
  refresh();
} catch (err) {
  toast.error(err instanceof Error ? err.message : 'Failed to delete user');
}
```

## Confirmation Dialogs

### When Required

A centred confirmation dialog is **mandatory** before any of the following:

- Deleting a record (user, org, report, template, merger, contract, section, etc.)
- Discarding unsaved changes
- Revoking access, permissions, or subscriptions
- Any action with the word "permanent", "cascade", or "cannot be undone" in its description
- Bulk destructive actions (delete-many, purge, reset)

For trivial reversible toggles (favourite, pin, hide) — no dialog needed. Just the toast.

### Implementation

Build a reusable `ConfirmProvider` + `useConfirm()` hook. Mount the provider in the root layout alongside the toast provider. The hook returns a promise-based function so calling code reads top-to-bottom:

```tsx
const askConfirm = useConfirm();

const handleDelete = async (id: string) => {
  const ok = await askConfirm({
    title: 'Delete user',
    description: 'This user will be permanently removed. This action cannot be undone.',
    confirmLabel: 'Delete',
    destructive: true,
  });
  if (!ok) return;
  await deleteUser(id);
  toast.success('User deleted');
};
```

### Dialog Requirements

- **Centred modal** (MUI Dialog, Radix Dialog, or equivalent). Never a drawer, never a popover, never inline.
- **Backdrop click dismisses** (treated as cancel).
- **Cancel on the left, confirm on the right.** Confirm button is `variant="contained"` with `color="error"` for destructive actions, `color="primary"` otherwise.
- **Destructive confirm label** is specific: `Delete`, `Remove`, `Revoke` — not `OK` or `Yes`.
- **Autofocus the confirm button** (users should be able to press Enter to proceed after reading).
- **Escape dismisses** as cancel.
- **Always title + description.** Title is the verb-noun (`Delete report`); description is the consequence (`"Q3 Sales" will be permanently removed. This action cannot be undone.`).

### Message Format

| Part | Rule | Example |
| ---- | ---- | ------- |
| Title | Verb + noun, no question mark | `Delete user` |
| Description | One sentence, states the consequence | `This user will be permanently removed. This action cannot be undone.` |
| Confirm label | Imperative verb | `Delete`, `Remove`, `Discard` |
| Cancel label | Default `Cancel`; for discard-changes use `Keep editing` | `Cancel` / `Keep editing` |

## Inline Alerts

Use an in-page banner (MUI Alert, Radix Alert, custom Banner) when the condition is persistent and the user needs to see it until they act on it:

- Form validation summary at top of form
- Service degradation ("Live prices are 30+ minutes stale")
- Permission denied in place of content
- Unsaved changes indicator in an edit page

Do **not** use an inline alert for transient feedback — that's a toast. Do **not** use a toast for a persistent condition — the user will miss it when it times out.

## Accessibility

- Toasts must render inside an `aria-live="polite"` region (libraries do this by default; verify).
- Errors should use `aria-live="assertive"`.
- Confirmation dialogs must trap focus, return focus to the trigger on close, and be labelled via `aria-labelledby`.
- Close buttons on toasts must be keyboard reachable and have an accessible label (`aria-label="Dismiss"`).
- Colour alone must not carry meaning — include an icon per variant (✓, ✕, ⚠, ℹ).
- Never auto-dismiss confirm dialogs.

## Architectural Rules

1. **Single provider.** Exactly one `<Toaster />` and one `<ConfirmProvider />` in the app, both mounted in the root layout inside the theme provider.
2. **Re-export pattern.** If a wrapper module exists (e.g. `components/common/snackbar.ts`), feature code imports from the wrapper — not directly from `react-hot-toast` / `notistack`. Swapping libraries then touches one file instead of hundreds.
3. **No per-page snackbars.** Using `<Snackbar>` + local `useState` in a feature component is a violation — it bypasses the position/duration/dedupe standards and creates drift.
4. **Promise-based confirm hook.** `useConfirm()` returns `(options) => Promise<boolean>`. Callers `await` it; no render-prop APIs, no `onConfirm`/`onCancel` callback forests.

## Enforcement Checklist

Before merging, verify:

- [ ] No `window.alert(`, `window.confirm(`, or `window.prompt(` calls in the diff (grep the whole repo once per release).
- [ ] No bare `<Snackbar>` components outside the root provider.
- [ ] Every new CRUD action emits a toast on success AND error.
- [ ] Every destructive action is gated by `useConfirm()` with `destructive: true`.
- [ ] Toast messages follow the Action + Result format, sentence case, no trailing period.
- [ ] Confirm dialogs have title + description + specific verb label + destructive colour.
- [ ] Toast position is `top-centre` with a vertical offset that clears the header. Duration is 3000 ms.
- [ ] Icons are present for each variant; colour is not the sole cue.

## Anti-Patterns

| Anti-pattern | Why it's wrong | Fix |
| ------------ | -------------- | --- |
| `if (confirm('Delete?'))` | Blocks main thread, can't style, inconsistent with app chrome | `useConfirm()` dialog |
| `alert('Saved!')` | Modal native dialog, looks like a browser warning | `toast.success('Saved')` |
| Per-page `useState` snackbar with `anchorOrigin="bottom-center"` | Drifts from platform convention | Delete local snackbar, use global toast |
| Toast says `"Error: 500"` | Technical leak, user can't act on it | `toast.error('Failed to save report')` |
| Destructive action with `color="primary"` confirm button | Red is the universal "dangerous" signal | `color="error"` + specific verb label |
| Silent delete (no toast after success) | User can't tell if the action worked | Always emit a toast |

## Rationale

Consistency in notifications is a trust signal. A platform where every delete asks in a different style, every success lands in a different corner, and half the errors are swallowed silently reads as unfinished. The three-tool model (toast / confirm / inline) covers every feedback case without overlap, and the positional and timing defaults let users build muscle memory — they stop reading the text and start trusting the behaviour.

When in doubt: **transient success or error → top-centre toast below the header**; **destructive decision → centred confirm dialog**; **persistent condition → inline banner**. Everything else is a violation.
