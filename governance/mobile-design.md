# Mobile Design

> Authoritative standard for how our phone apps look, move and behave: the bar is the best apps on each platform, not "the web app, smaller".

## Purpose

People judge a phone app in seconds, and they judge it against the best apps they already use: Apple's own apps, Things, Linear, Arc Search, Airbnb, Monzo, StoryGraph. Those apps share a handful of habits. They feel native to the phone they run on. They show one thing at a time and keep the rest a tap away. They answer every touch at once, with motion and a small haptic tick. They never block the screen with a spinner or a system alert when something softer will do. An app built from a component library's defaults has none of these habits. It works, but it feels like a prototype.

This standard sets out those habits as rules, with a review checklist. It is mandatory for every mobile app (Expo / React Native), and for the mobile layouts of tablet apps. [Mobile Development](/standards/development/mobile-development) covers how mobile apps are built, configured and shipped; this standard covers what the person holding the phone sees and feels.

Related standards: [UI Consistency](/standards/governance/ui-consistency), [Accessibility Standards](/standards/governance/accessibility-standards), [Settings Pages](/standards/governance/settings-pages), [Notifications & Confirmations](/standards/governance/notifications-and-confirmations), [Performance Budgets](/standards/quality/performance-budgets).

## Core Principles

1. **Native on each platform.** An iPhone app follows Apple's [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines); an Android app follows [Material 3](https://m3.material.io). Brand comes from colour, type, imagery and voice, never from overriding the platform's navigation, gestures or controls.
2. **One thing at a time.** Each screen has one job and one primary action. Everything else is collapsed, in a sheet, or behind a menu (progressive disclosure).
3. **Instant response.** Every touch gets visible feedback within 100 ms. Changes apply on screen straight away and sync behind the scenes (optimistic UI); the network is never something the person waits on to see their own action.
4. **Soft, not blocking.** No centred spinners on a blank page, no system alerts for routine messages, no full-screen modals for small choices.
5. **Content first.** Covers, photos and the person's own data are the interface. Chrome (borders, boxes, labels, dividers) is the minimum that makes the content readable.
6. **Thumb-friendly.** Frequent actions sit in the bottom half of the screen, targets are large, and one-handed use is the default case.
7. **Accessible by construction.** Dynamic Type, VoiceOver and TalkBack labels, contrast, and Reduce Motion are part of the first build, not a later pass.

## Platform Conventions

| Area | iOS | Android |
| ---- | --- | ------- |
| **Navigation** | Tab bar at the bottom (3–5 tabs), stack navigation with a back swipe from the left edge | Bottom navigation bar (3–5 destinations), system back gesture |
| **Screen titles** | Large title that collapses into the bar on scroll for top-level screens; inline title on detail screens | Top app bar; medium or large app bar on top-level screens |
| **Search** | In the navigation bar (pull down to reveal on long lists), not a custom box in the content | Search bar or search view at the top |
| **Primary action** | A bar button (Add, top right) or a prominent button in the content | Floating action button, bottom right |
| **More actions** | Native context menu on long press, or a pull-down menu from an ellipsis button | Overflow menu (three dots), long press for selection mode |
| **Row actions** | Swipe left for destructive actions (Delete), swipe right for the positive one (Mark as read) | Swipe where it fits the content; otherwise a menu |
| **Choices** | Segmented control (2–4 options), menus, sheets | Segmented buttons, chips, menus, sheets |
| **Icons** | SF Symbols (`expo-symbols`) | Material Symbols |
| **Haptics** | Selection, light impact, success and warning feedback | The equivalent haptic constants |

One codebase still ships both: use the platform's control through a library that renders the native one (native stack headers, native context menus, platform-specific icon sets). Where no native control exists, build one component that follows the platform's look on each (`Platform.select`), never an Android look on an iPhone.

## Layout and Spacing

- **8-point grid**: spacing is 4, 8, 12, 16, 24, 32 or 48. Screen side margins are 16 (20 on larger phones is acceptable). No other values.
- **Safe areas**: content respects the notch, Dynamic Island and home indicator; lists scroll under translucent bars and the tab bar, never stop short of them.
- **Touch targets**: at least 44 × 44 pt on iOS and 48 × 48 dp on Android, including icon buttons, chips and the clear button in a search field. Add hit slop rather than shrinking the target to fit the drawing.
- **Reach**: put frequent actions in the bottom half. Destructive actions never sit where a thumb rests by accident (bottom corners next to the primary action).
- **Density**: a list row shows title, one line of detail and at most one status. A second status, a progress bar and a rating on the same row is too much; move the rest to the detail screen.
- **Cards** are for content that is a thing in its own right (a book, a session). Settings, forms and plain lists use grouped lists, not a column of cards.

## Typography

- One type scale, from the design tokens, mapped to the platform's text styles (iOS: Large Title, Title 1–3, Headline, Body, Callout, Subheadline, Footnote, Caption; Material 3: Display, Headline, Title, Body, Label).
- **Dynamic Type**: text scales with the system setting. Layouts reflow at the largest accessibility sizes instead of truncating; never set `allowFontScaling={false}` on body text.
- Body text is never smaller than 15 pt on iOS or 14 sp on Android. Captions (11–13) are for metadata only.
- Hierarchy comes from size and weight, not colour or capitals. No paragraph in all caps; no more than two weights on one screen.
- Numbers that change (timers, counters, prices) use tabular figures so they do not jitter.

## Colour and Theme

- Colours come only from the design tokens (`nativeColors()` from `@axiomstudio/design` in The Forge). No hex, rgb or hsl literal in a screen; status colours (priority, success, warning) are tokens too.
- **Light and dark** both ship, following the system setting by default with a manual override in Settings.
- Use the platform's semantic layers: background, grouped background, surface, separators. Elevation is shown with tonal surfaces (Material) or subtle grouping (iOS), not heavy drop shadows.
- Accent colour is for the primary action, selection and links. A screen where everything is accent-coloured has no hierarchy.
- Text contrast meets WCAG AA (4.5:1, 3:1 for large text) in both themes.

## Progressive Disclosure

The rule behind most of the others: show what the person needs for the current step, and keep the rest one tap away.

| Pattern | Rule |
| ------- | ---- |
| **Lists of options** (categories, tags, authors, filters) | Never render an open-ended list in full. Each group starts collapsed, showing only the current choice ("Any", the selected names, or "3 selected"). An open group shows the first 6–8 options, selected ones first, with "Show all N" and a search field once there are more than 8 |
| **Filters** | Two or three quick filters (the most used) stay on screen as segmented controls or chips; the rest live in a filter sheet behind a filter button, which shows a badge with the number active. "Clear" is always one tap from the list |
| **Forms** | Required fields first; optional and advanced fields under a collapsed "More details" group |
| **Detail screens** | Summary at the top (cover, title, status, the primary action); long text truncated with "More"; secondary data in grouped sections below |
| **Destructive and rare actions** | In a menu (ellipsis or long press), never as a row of buttons on every list item |

## Sheets, Menus and Dialogs

| Need | Use | Never |
| ---- | --- | ----- |
| A small choice, a filter set, a quick edit, a picker | **Bottom sheet** (native sheet on iOS, modal bottom sheet on Android) with a grabber, detents (medium and large), swipe to dismiss | A box centred on the screen |
| A few actions on one item | **Native context menu** (long press) or a pull-down menu | A system alert listing the actions |
| A task with several steps or fields (add a book, edit a session) | A pushed screen, or a large sheet with Cancel and Save in its header | A dialog with scrolling content |
| Confirming a destructive action | Native action sheet (iOS) or a dialog (Android), with the destructive button styled as destructive; undo instead wherever the action can be reversed | Confirming routine, reversible actions |
| An error or problem the person must act on | A dialog | A dialog for errors they cannot act on |

Sheets keep their state when dismissed by accident only if reopening would otherwise lose typed input; otherwise they reset. A sheet never opens another sheet on top of itself; push a screen inside it instead.

## Feedback and Status

- **Pressed state** on everything tappable: opacity or highlight on iOS, ripple on Android.
- **Haptics** (`expo-haptics`): selection feedback for pickers and toggles, light impact for a completed drag or pull to refresh, success when a task finishes (session saved, book added), warning before a destructive confirmation. Haptics accompany a visual change, never replace one, and are never used on every scroll or keystroke.
- **Toasts / snackbars** for confirmations, with Undo for anything destructive that can be reversed ("Session deleted. Undo"). One at a time, at the bottom above the tab bar, 3–4 seconds. Routine success needs no message at all when the screen already shows the result.
- **Errors** appear where the problem is: next to the field, at the top of the list ("Couldn't refresh. Showing saved data."), or as a retry row. Never a system alert for a failed refresh.
- **Offline** is a quiet banner or status, not an error. Actions taken offline look done and show a small "Waiting to sync" marker until they sync.

## Loading, Empty and Error States

Every screen designs all four states, not just the happy one:

| State | Rule |
| ----- | ---- |
| **Loading (first time)** | Skeletons shaped like the content (rows, covers, cards), not a spinner in the middle of a blank screen |
| **Loading (refresh)** | Pull to refresh with the native control; the existing content stays on screen |
| **Empty** | An illustration or large symbol, one line saying what goes here, and one button to add the first item. A filtered list with no matches says so and offers "Clear filters" |
| **Error** | What went wrong in plain words, what still works ("Showing your saved books"), and Retry |
| **Partial** | Pagination loads the next page before the end of the list (prefetch), with a footer skeleton, never a "Load more" button |

## Motion

- Motion explains: where a screen came from, what changed, what can be dragged. It is never decoration.
- Use spring animations (`react-native-reanimated`) for interactive movement; 200–350 ms for transitions; never longer than 500 ms.
- Animate list changes: a deleted row collapses, an added row slides in, a reordered row moves (layout animations).
- Shared element transitions (a book cover growing into the detail screen) are encouraged for the app's main object.
- **Reduce Motion**: when the system setting is on, replace movement with fades and skip parallax and springs.
- 60 frames per second on a mid-range phone, at least; animations run on the UI thread, never in JavaScript timers.

## Lists and Images

- Long lists use a recycling list (`@shopify/flash-list`, or `FlatList` with fixed item sizes) and memoised rows.
- Rows support the platform's gestures: swipe actions, long-press menu, and drag to reorder where order matters.
- Section headers are sticky on grouped lists (by date, by status).
- Images use `expo-image`: cached on disk, sized to the slot they fill, with a blurhash or tinted placeholder and a short fade-in. A missing image shows a designed placeholder (a generated cover with the title), never a broken box or a grey square.
- Covers, avatars and thumbnails keep their aspect ratio and corner radius from the tokens.

## Forms and Input

- The right keyboard for each field (`number-pad`, `email-address`, `url`), the right `textContentType` / `autoComplete` for autofill, and a return key that moves to the next field.
- The keyboard never covers the focused field (`KeyboardAvoidingView` or a keyboard-aware scroll view); tapping outside dismisses it.
- Native pickers for dates and times. Steppers or sliders for small numbers (pages read, rating) instead of typing.
- Validate as the person types after the first blur, with the message under the field. The Save button stays enabled and explains what is missing when pressed, rather than sitting greyed out with no reason.
- Changes that can be applied immediately (a toggle, a rating, a status) save immediately; only multi-field edits need a Save.

## Phone-Only Capabilities

A phone app earns its place on the home screen by doing what a website cannot. Consider each of these for every app, and record why it was or was not adopted:

| Capability | Typical use |
| ---------- | ----------- |
| **Widgets** (home screen, lock screen) | Today's tasks, the book being read, a streak |
| **Live Activities / ongoing notifications** | A running timer (reading or focus session) on the lock screen and Dynamic Island |
| **Push and local notifications** | Reminders the person asked for, streak at risk, a shared item; always with in-app controls per type |
| **Share extension** | Save a link, a book or a quote from another app |
| **Quick actions** (long press on the app icon) and **App Shortcuts / Siri** | Start a session, quick add, scan a barcode |
| **Camera** | Barcode scanning, photographing a cover or a page |
| **Biometrics** | Unlock for sensitive data, never as the only way in |
| **Offline** | The app opens and works without a connection; see [Mobile Development](/standards/development/mobile-development) |

## Accessibility on Mobile

On top of [Accessibility Standards](/standards/governance/accessibility-standards):

- Every interactive element has an accessible label and role; icon-only buttons always have a label.
- Related elements in a row are grouped so a screen reader reads one sentence per row ("The Hobbit, J. R. R. Tolkien, reading, 45 percent").
- Custom actions (`accessibilityActions`) expose swipe and long-press actions to screen reader users.
- State is announced: selected, expanded, disabled, progress.
- Test each new screen with VoiceOver and TalkBack, at the largest text size, in dark mode and with Reduce Motion on.

## Recommended Stack (Expo)

| Need | Library |
| ---- | ------- |
| Navigation with native headers, large titles and header search | React Navigation native stack (`headerLargeTitle`, `headerSearchBarOptions`) |
| Bottom sheets | `@gorhom/bottom-sheet`, or the native stack's `formSheet` presentation |
| Native context and pull-down menus | `@react-native-menu/menu` (or `zeego`) |
| Haptics | `expo-haptics` |
| Images | `expo-image` |
| Long lists | `@shopify/flash-list` |
| Animation and gestures | `react-native-reanimated`, `react-native-gesture-handler` (swipeable rows) |
| Icons | `expo-symbols` on iOS, Material Symbols on Android |
| Blur and materials | `expo-blur` |
| Widgets and Live Activities | Expo config plugins with native targets (needs a native build) |

A component library (react-native-paper and similar) may supply buttons, text fields and theming, but its defaults are not the design: replace a library dialog with a sheet, a library menu with a native menu, and a library app bar with the native header wherever this standard calls for it. Adding any of these libraries is a native change: it needs a new native build, not an over-the-air update.

## Review Checklist

Run this for every new or changed screen, on a real phone, in light and dark:

- [ ] Follows the platform: native header and title behaviour, back gesture, native menus, platform icons.
- [ ] One primary action, easy to reach; destructive actions in a menu or swipe, with Undo where reversible.
- [ ] Open-ended lists collapsed with a preview, Show all and search; filters behind a filter button with a count.
- [ ] Sheets, not centred dialogs, for choices and quick edits; no system alert used as a menu.
- [ ] Spacing on the 8-point grid; touch targets at least 44 pt / 48 dp; content clear of safe areas.
- [ ] Type from the scale, scales with Dynamic Type, nothing truncated at large sizes.
- [ ] Colours only from tokens; correct in light and dark; contrast AA.
- [ ] Loading (skeleton), empty, error and offline states designed and reachable.
- [ ] Every touch has a pressed state; haptics on completion and selection; changes appear instantly.
- [ ] Motion purposeful, under 350 ms for transitions, honours Reduce Motion, smooth at 60 fps.
- [ ] Images cached with placeholders; long lists recycle.
- [ ] Screen reader labels, grouping and actions; tested with VoiceOver or TalkBack.
- [ ] Phone-only capabilities considered for the feature (widget, notification, shortcut, share).

## Anti-patterns

| Anti-pattern | Instead |
| ------------ | ------- |
| A system alert as an action menu ("Edit / Delete / Cancel") | Native context menu or swipe actions |
| A centred modal box holding a long form or list | Bottom sheet with detents, or a pushed screen |
| Every option of a long list rendered at once | Collapsed groups with a preview, Show all and search |
| A spinner in the middle of a blank screen | Skeletons, and cached content first |
| "Are you sure?" for reversible actions | Do it, then offer Undo |
| Material components and look on an iPhone | Platform-specific controls and icons |
| Hardcoded colours for status or priority | Design tokens |
| A column of cards for settings or a form | Grouped lists |
| A row of three icon buttons on every list item | Swipe actions and a long-press menu |
| Typing numbers into a text field for small values | Stepper, slider or picker |
