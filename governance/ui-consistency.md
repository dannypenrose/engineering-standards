# UI Consistency

> Authoritative standards for keeping the same thing looking and behaving the same way: within one application, and across every application in a monorepo.

## Purpose

People learn an interface once. When Share sits under the title on one page and at the bottom of the next, or search looks one way in one product and another way in its sibling, every page becomes a small puzzle and every app feels like a different company built it. Inconsistency is also how bugs hide: a pattern implemented five times is fixed in four.

This standard defines what must be consistent, where the single implementation lives, and how to check it. It is mandatory for all web frontends, and for every app in a monorepo that shares a UI package.

Related standards: [Code Sharing](/standards/development/code-sharing), [Notifications & Confirmations](/standards/governance/notifications-and-confirmations), [Global Search](/standards/governance/global-search), [Rich Text Editors](/standards/governance/rich-text-editors), [Accessibility Standards](/standards/governance/accessibility-standards).

## The Two Levels

| Level | Rule | Where the single implementation lives |
| ----- | ---- | ------------------------------------- |
| **Within an app** | A pattern that appears on more than one page is one component, used everywhere it appears | The app's shared components folder (e.g. `components/<app>/`) |
| **Across a monorepo** | A pattern that more than one app needs is one component in the shared UI package, configured per app | The shared package (e.g. `@scope/ui`, `@scope/search`) |

Before writing a UI pattern, ask the shared-first question: *could another page, or another app, show this?* If yes, it goes in the shared place first and the page uses it. Never copy a component between apps to change one detail; add a prop.

## Page Anatomy

Every detail page (one record: an article, an event, a course, a project) follows the same order, top to bottom:

1. **Breadcrumb** back to the listing.
2. **Editor tools bar**: status and management actions (Edit, Publish, Archive, Delete), shown only to people who can use them. It is an affordance, never a security boundary; the API enforces permission.
3. **Header**: chips (type, status, audience), the title as the page's single `h1`, a one-line summary, then a row of details (date, author, reading time).
4. **Action row**: the last line of the header. See below.
5. **Body**: content, then related items.

### The action row

One component renders it (e.g. `DetailActions`), and every detail page uses it. The order is fixed:

1. The page's own primary action (Add to calendar, Visit website, Download).
2. **Share**.
3. **Pin / favourite**.
4. Onward links (My events, All bookings).

Rules:

- Share and Pin appear **nowhere else** on a detail page: not in the title row, not at the bottom, not as icon-only variants on some pages and buttons on others.
- Use the same button style for every item in the row. Do not mix icon-only and labelled buttons.
- Editor actions never go in the action row. They serve a different audience and live in the editor tools bar.
- Workflow controls that are the point of the page (an RSVP control, a booking button per session) stay where the content puts them; the action row is for the actions every page has.

### Listing pages and cards

- The create button sits in the listing's header, right-aligned, labelled with the noun ("New course", not "Create").
- Per-card actions are one compact component (edit, delete, a "more" menu), placed outside the card's link area. A button inside an anchor is invalid HTML and fires the navigation.

## Shared Behaviours

These must behave identically everywhere they appear. Each has one implementation.

| Behaviour | Standard |
| --------- | -------- |
| Toasts, confirm dialogs, inline alerts | [Notifications & Confirmations](/standards/governance/notifications-and-confirmations) |
| Destructive confirmation | Confirm dialog naming the consequence; type-to-confirm for permanent deletion only, never for reversible actions |
| Share | One Share menu (see below) |
| Global search | [Global Search](/standards/governance/global-search) |
| Comments, replies, descriptions, emoji and reactions | [Rich Text Editors](/standards/governance/rich-text-editors) |
| Empty states | Say what will appear and how to make it appear; never a blank area |
| Loading | Skeletons shaped like the content, not spinners in the middle of the page |
| Dates and times | One formatter per kind (date, date-time, range), in the product's locale and time zone |

### Sharing

A single **Share** button opens a menu. It never opens a third-party site directly.

- **Options**: Copy link (always works, confirms with a toast), the team's messaging tool (e.g. Microsoft Teams), and Email. Add a device share sheet only if product asks for it.
- **The link is the production URL**, never the sharer's own origin, so a link copied in development or staging still works for the recipient.
- **The pre-filled message** is the title, the date or time if the item has one (never cut), the product name, and as much summary as fits the target's limit, cut on a word with an ellipsis. Summaries passed in must be plain text, never stored HTML.
- **Third-party dialogs open in a centred popup**, keeping the person on the page, and fall back to an ordinary link in a new tab when the popup is blocked or the click is modified.
- **No third-party scripts** for share buttons, and nothing is recorded about who shares what.

## Cross-App Consistency in a Monorepo

- **Shared patterns live in shared packages.** The header, sidebar footer, search palette, share menu, confirm dialog and snackbar provider are package components. Apps pass configuration (their modules, icons, labels, colours), never fork the component.
- **Configuration, not copies.** When an app needs a variation, add an optional prop with a default that leaves every other app unchanged. Example: a logo blend mode that defaults to the old behaviour.
- **Brand through the theme, not the component.** Apps differ in logo, colour and name; spacing, radius, typography scale and interaction patterns stay the same.
- **A shared change reaches every app.** Before changing a shared component, list its consumers (`grep -rl` the export across `apps/`), type-check each, and say in the PR which apps it changes. A package change may not deploy anything by itself; note which apps need a deploy to pick it up.
- **Keep the props API stable.** Additive changes only, unless every consumer is updated in the same change.

## Checking Consistency

On every UI change:

- [ ] Does this pattern already exist in the app or a shared package? Use it.
- [ ] If it appears on more than one page, is it one component?
- [ ] Do detail pages follow the page anatomy, with actions in the action row only?
- [ ] Does a shared-component change leave every other consumer working (type-check each app)?
- [ ] Are toasts, confirms, empty states and loading states the shared ones?
- [ ] UK English in copy; no em dashes.

When reviewing, grep for the anti-patterns: a share or pin component imported outside the action row component's call sites, more than one implementation of the same menu, inline `window.confirm`, page-local date formatters that duplicate a shared one.
