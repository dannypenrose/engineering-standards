# Settings Pages

> Authoritative standard for how an application's Settings are laid out, found, saved and described. One shape for every web, desktop and mobile app.

## Purpose

Settings grow. A page that starts with three options has thirty a year later, and the layouts that work at three (one long page, a row of tabs, anchor links down the side) fall apart well before thirty: options get buried, people scroll past what they came for, and every app ends up arranged differently. The products people already know (macOS System Settings, VS Code, Obsidian, Notion, Linear) converged on one pattern. This standard adopts it, so a person who has used Settings in one of our apps can use it in all of them.

It is mandatory for every application with user-facing settings: web frontends, desktop apps (Electron, Tauri) and mobile apps. Admin-only configuration screens follow it too.

Related standards: [UI Consistency](/standards/governance/ui-consistency), [Global Search](/standards/governance/global-search), [Accessibility Standards](/standards/governance/accessibility-standards), [Notifications & Confirmations](/standards/governance/notifications-and-confirmations).

## The Layout

| Part | Rule |
| ---- | ---- |
| **Entry point** | Settings opens from the top bar's settings cog and from ⌘, (Ctrl+,). It is never an item in the main navigation. |
| **Section list** | Every section listed down the left, always visible, in a fixed order. Related sections may sit under a small overline (Account, Workspace, App). Below the tablet breakpoint the list becomes a picker (a select) above the content. |
| **Search** | A search field at the top of the section list, on every Settings page. It searches every setting's name, description and keywords, plus section titles. ⌘F (Ctrl+F) focuses it, Enter opens the first result, Escape clears it. |
| **One section per page** | The right-hand side shows one section at a time, headed by its title and an optional one-line description. Never one long page of every section; never anchor links down a single page. |
| **Addresses** | Each section has its own address, so it can be linked to, bookmarked and opened from the command palette: `/settings/<section>` in apps with client routing (the Electron apps), `/settings?tab=<section>` in the Next.js apps, where the shared component manages it. A link to one setting adds `setting=<id>`: the section opens, scrolls to the setting and highlights it briefly. An unknown section opens the first one. |
| **Tabs** | Not for Settings. A row of tabs hides sections once they overflow and cannot hold search. |
| **About** | The last section. Version, platform, data folder where relevant, and Check for updates on desktop. |

### Section order

1. **General** first: the options most people change (appearance, language, start-up).
2. Feature sections in the order the features appear in the app.
3. **Account**, **Workspace** or **Admin** sections, grouped under their overline, shown only to people who can use them.
4. **About** last.

Name sections with one or two plain words (Editor, Shortcuts, Notifications). Do not create a section for one setting unless it will grow; put it in General.

## A Setting

Every setting is a **row**:

- On the left, its **label** (sentence case, a noun or a short imperative: "Open at login", "Theme") and a **one-line description** of what it changes. Longer help belongs in the documentation, linked if needed.
- On the right, its **control**: a switch for on/off, a segmented control for two to four choices, a select for more, a text field, a slider, or a button for an action. Wide controls (lists, multi-line fields) sit under the text instead.
- Related rows sit together in a **group** with a short heading. A group may carry one action for all its rows (Add, Reset all).
- A setting kept somewhere the person would not assume is **labelled**: "This device" (kept in this browser or on this computer only), "Your account" (follows the person), "This workspace" (changes it for everyone). Say nothing when it is obvious from the section.

### Saving

- **Settings save as they change.** No Save button at the bottom of a page, which people forget, and no unsaved state to lose on navigation.
- One exception: a set of values that are only valid together (warning and critical thresholds, a connection's host, port and credentials) may save as one group, with Save and Discard on that group only, never on the page.
- Text fields save when they lose focus or on Enter, never on every key; Escape restores the saved value.
- The row shows the save itself: "Saving…" while it is in flight, "Saved" for a moment, or the error (announced as an alert) with the control back at its saved value. A toast is not a substitute.
- A change that is destructive, costs money or affects other people confirms first (see [Notifications & Confirmations](/standards/governance/notifications-and-confirmations)). A change that cannot apply until restart says so in its description.
- A setting with a sensible default can be reset to it; a section with many (Shortcuts) offers Reset all.

## Search

- Every row is registered for search with an id that matches the row, its label, its description and keywords people might type instead (synonyms, the old name, the platform's word: "startup" for Open at login).
- Matching is by the start of each word typed; every word must match. Matches on the label rank above matches on the description; settings rank above sections.
- Results show the setting's label and where it lives ("Editor · Font size"). Choosing one opens its section at the setting.
- Each section is also a command in the app's command palette ("Settings: Editor"), per [Global Search](/standards/governance/global-search).

## Accessibility

- The section list is a `nav` with an accessible name; the current section carries `aria-current="page"`.
- The search field has a label (visible or screen-reader only); the result count is announced (`aria-live="polite"`).
- Every control has an accessible name matching its row's label. A switch is a real switch (`role="switch"`, `aria-checked`).
- Save status uses `role="status"`; a failed save uses `role="alert"`.
- Everything works from the keyboard, and highlighting a searched-for setting respects reduced motion.

## Implementations

Use the shared implementation for your platform; never build a Settings layout in an app.

| Platform | Use |
| -------- | --- |
| Tailwind / Radix apps (Electron apps, the Electron template) | `@axiomstudio/design/react`: `SettingsLayout`, `SettingGroup`, `SettingRow`, `SettingInput`, `useSaveStatus` |
| MUI apps (The Forge's web apps) | `@forge/ui`: `TabbedSettings` (the sidebar layout with search) |
| Zenith (web and desktop) | `@forge/zenith-editor`'s Settings plug-in, which follows this standard with its own styles |
| Mobile (Expo) | Up to one screen of settings (about a dozen rows): one Settings screen of titled groups (react-native-paper `List.Section` titles, `List.Item` rows with a description), About last. Beyond that: a list of sections as the Settings screen, each section its own screen, with search at the top of the list |

An app adds sections, never layouts. If the shared implementation lacks something an app needs, change the package.

## Anti-Patterns

| Anti-pattern | Instead |
| ------------ | ------- |
| One long page of every section, with or without anchor links | One section per page, listed on the left |
| Tabs across the top of Settings | The section list |
| Settings without search | Search above the section list, every row registered |
| A Save button for a page or a card | Save each setting as it changes, status on its row |
| Saving a text field on every key | Save on blur or Enter |
| A toast as the only sign a setting saved or failed | The row's status |
| Settings as a main navigation item | The top bar's cog and ⌘, |
| A row without a description, or a description that repeats the label | One line on what it changes |
| A section's address that cannot be linked to | `/settings/<section>` |

## Checklist

- [ ] Settings opens from the cog and ⌘, (Ctrl+,), and is not in the main navigation
- [ ] Sections listed on the left (a picker on small screens), one section per page, About last
- [ ] Search at the top of the section list; ⌘F focuses it; every row is registered with keywords
- [ ] Each section has its own address; `?setting=<id>` opens and highlights one setting
- [ ] Each section is a command in the palette
- [ ] Every setting is a row: label, one-line description, control
- [ ] Settings save as they change; the row shows Saving…, Saved or the error
- [ ] Settings kept somewhere unexpected say where ("This device", "Your account", "This workspace")
- [ ] Built on the shared implementation, not an app-local layout
