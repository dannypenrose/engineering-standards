# Global Search

> Authoritative standards for the global search palette (⌘K / Ctrl K): how it looks, how it behaves, what it searches and how it stays safe.

## Purpose

Global search is how people get anywhere fast. A palette that opens empty, mixes every result into one list, or shows items the person cannot open is worse than no search, because people stop trusting it. Every app in a monorepo uses the same palette so the shortcut, layout and keys work the same everywhere.

Related standards: [UI Consistency](/standards/governance/ui-consistency), [Accessibility Standards](/standards/governance/accessibility-standards), [Code Sharing](/standards/development/code-sharing).

## One Implementation

- The palette, its header trigger and its keyboard shortcut come from the shared search package (e.g. `@scope/search`: `SearchCommand`, `SearchTrigger`, `useCommandPalette`). Apps never build their own.
- An app supplies only **what to search**: it registers modules on the shared registry, passes module icons, and registers "Jump to" actions. That code lives in one place per app (e.g. `lib/search.ts` plus a small header component).

## Opening

- **Shortcut**: ⌘K on macOS, Ctrl K elsewhere, from any page. The trigger in the header shows the shortcut for the person's platform, worked out in the browser after mount.
- **Trigger**: a search-field-shaped button on wide screens; an icon button on narrow ones. It sits in the app bar and inherits its colours.
- **Render after mount** if anything the palette offers depends on client-side state such as feature flags, so the server and browser render the same markup.

## Layout

- A centred panel near the top of the viewport (about 12% down), 640 px wide at most, with a blurred backdrop. **Full screen on phones**, with a visible close button.
- **Input**: one large borderless field with a search icon (a small spinner while searching), a clear button when there is text, and an `Esc` key hint when there is not.
- **Results grouped by module**, in the order the registry ranks them, each group headed by the module's display name.
- **Each row**: the module's icon in a tinted tile, the title with **matched words highlighted**, and one line of context (where it lives: project, board, category, status). The highlighted row shows an Enter hint.
- **Footer**: the result count on the left (announced politely to screen readers) and key hints on the right: ↑ ↓ to move, ↵ to open, Esc to close.

## States

| State | What it shows |
| ----- | ------------- |
| Nothing typed | **Recent searches**, then **Jump to**: the sections of the app the person can see, with their navigation icons |
| Searching, no results yet | Skeleton rows shaped like results |
| Results | Grouped rows as above |
| No results | An icon, "No results for …", and a short suggestion |

- Save a search to "Recent" **only when a result is opened**, never on every debounced keystroke, or half-typed words fill the list.
- Debounce input (around 200–300 ms). Never search on an empty query.

## Keyboard and Accessibility

- The input is an ARIA `combobox` controlling a `listbox`; every row is an `option` with `aria-selected`, and the highlighted row is the input's `aria-activedescendant`. Groups use `role="group"` labelled by their heading.
- ↑ and ↓ move through every row in the order shown, wrapping at the ends; Enter opens; Esc closes. Moving the mouse over a row highlights it, so keyboard and mouse never disagree.
- The highlighted row scrolls into view.
- Focus returns to the input when a recent search is chosen or the query is cleared.

## What Search May Return

- **Use the endpoints the pages already use.** A module calls the same list endpoint, with the same filters, as the page that shows that content. That way search inherits every access rule (draft status, audience, team limits, guest scopes) and cannot leak an item the person could not open. Do not add a search endpoint that bypasses them.
- **Scope by role.** A restricted user (a guest, an external client) gets only the modules they can use; disable the rest rather than let the API refuse them.
- **Small, stable lists may be filtered in the browser** (for example, the boards a person belongs to), cached briefly.
- **Cap each module** (around 5–8 rows) so one module cannot crowd out the others.
- **Never log or track** what someone searched for beyond their own browser's recent list.

## Results Open the Right Thing

- A result opens the exact item, not its listing: a card opens its board with the card showing; a checklist item opens its parent card.
- Selecting a result closes the palette and navigates in the same tab.

## Checklist

- [ ] Palette and trigger come from the shared package; the app only registers modules, icons and Jump to actions.
- [ ] Every module calls an existing page endpoint, so access rules apply.
- [ ] Restricted roles see only the modules they can use.
- [ ] Results grouped, highlighted, capped per module, with a one-line context.
- [ ] Recent searches saved on open only; Jump to follows the visible navigation.
- [ ] Combobox, listbox and option roles; arrow keys, Enter, Esc; full screen on phones.
