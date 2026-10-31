# Rich Text Editors

> Authoritative standards for WYSIWYG (rich-text) editing: which editor to use, what it offers, how the HTML it writes is stored, shown and sanitised, and how emoji and reactions work.

## Purpose

People write comments, replies and descriptions in every product. When one app has bold and emoji, another a bare text box, and a third a different editor with different buttons, people relearn the basics on every page, and each copy is a separate place for an XSS hole. This standard makes rich text one thing: one editor, one way to show what it wrote, one sanitiser on the server, one emoji picker and one reaction row.

It is mandatory for every web frontend and API that accepts or shows user-written formatted text, and for every app in a monorepo that shares a UI package.

Related standards: [UI Consistency](/standards/governance/ui-consistency), [Security Guidelines](/standards/governance/security-guidelines), [Accessibility Standards](/standards/governance/accessibility-standards), [Code Sharing](/standards/development/code-sharing), [File Uploads](/standards/development/file-uploads).

## Which Field Gets Which Editor

| Field | Editor | Examples |
| ----- | ------ | -------- |
| Conversation | **Comment editor** | Comments, replies, ticket messages, internal notes |
| Descriptions of a record people work on | **Comment editor** | A ticket's description, a task or card's description |
| Long-form published content | **Article editor** (headings, tables, embeds) | Blog posts, knowledge-base articles, course and event pages |
| Short single-purpose text | Plain text field | Titles, summaries and excerpts, a recognition message, an RSVP note, a leave reason, anything that goes into an SMS, a Teams message or a subject line |
| Text sent to an AI model or parsed by code | Plain text field | Prompts, pasted source text, search queries |

Rules:

- **Never a plain multi-line text box for conversation.** Comments and replies always use the comment editor, so people can format, link and add emoji the same way everywhere.
- **Short text stays plain.** Rich text in a field that is shown on one line, sent where HTML does not render, or limited to a few hundred characters adds markup without adding meaning.
- **Two editors, no more.** The comment editor and the article editor are the only two. A third (a different library, a Markdown box, a "lite" variant) needs an ADR.

## The Comment Editor

One component in the shared UI package (e.g. `RichTextEditor` in `@scope/ui`), configured per field, never copied.

### Toolbar

The same buttons, in the same order, everywhere:

1. Bold, Italic, Strikethrough
2. Bullet list, Numbered list, Quote, Inline code
3. Link, Image (only when the field can store an upload), **Emoji**
4. Undo, Redo

- Groups are separated by a divider. Every button has a tooltip and an `aria-label`; toggles set `aria-pressed`.
- **Links** are edited in a small popover with a text field, never `window.prompt`. An address without a scheme gets `https://`. An empty address removes the link.
- **Images** are uploaded through the owning record's attachment or upload endpoint, then inserted by URL. Hide the button where there is nowhere to store one yet (a form for a record that does not exist), and point people at the form's own attachment field instead.
- **Emoji** opens the shared emoji picker (below) and inserts at the caret.
- **@mentions** are offered only when the field passes the people who may be mentioned; the server decides whom to notify, and checks each id against who may see the record.

### Behaviour

- **Grows with its content.** A minimum height suited to the field (about four lines for a reply, six for a description); no fixed height, no inner scroll bar.
- **Ctrl+Enter / ⌘+Enter sends** a reply or saves an edit, where the form has a single obvious submit.
- **Fields saved on blur** (a card description) save when focus leaves the editor, not when it moves into the editor's own popovers.
- **An emptied editor is empty.** The editor reports `''`, not `<p></p>`, so "required" checks and disabled Send buttons work.
- **Accessible name.** The editable area has `role="textbox"`, `aria-multiline="true"` and an `aria-label` naming the field ("Reply", "Description").
- **Placeholder** says what to write ("Type your reply here…"), not "Enter text".

## Storage

- Store the editor's **HTML** in a `TEXT` column (never a bare `VARCHAR(191)`), up to 15,000 characters: an emoji is four bytes, so that always fits 65,535 bytes.
- **Sanitise every write on the server**, whatever sent it: the REST DTO, an MCP tool, an import, a script. The editor reshaping input is not a security boundary.
  - Allow only what the toolbar produces: `p br strong b em i s strike del u code pre blockquote ul ol li a img h2 h3 hr span`.
  - Attributes: `a[href target rel]`, `img[src alt]`, `span[class="mention" data-mention-id]`. Nothing else: no `style`, no event handlers, no `class` except `mention`.
  - Schemes: `http`, `https`, `mailto` for links; `http`, `https` or a relative upload path for images. No `javascript:`, no `data:`, no protocol-relative URLs.
  - A link with `target="_blank"` always gets `rel="noopener noreferrer"`.
- **Plain text in, paragraphs out.** A write that is plain text (an AI agent, an older client) is escaped and turned into paragraphs, a blank line starting a new one and a newline becoming `<br>`, so everything newly stored is HTML.
- **Validate the text, not the markup.** "Required" means the visible text, or an image, is not empty. Length limits apply to the stored HTML.
- Images inserted into rich text are owned by the record, and deleted with it ([File Uploads](/standards/development/file-uploads)).

## Display

- **Show stored rich text only through the shared display component** (e.g. `RichTextContent`), which sanitises again in the browser (DOMPurify) before rendering. Never pass stored HTML straight to `dangerouslySetInnerHTML`, `v-html` or `innerHTML`.
- Older rows written before rich text are plain text: the display component detects them, escapes them and keeps their line breaks.
- The display component and the editor share one typography (`RICH_TEXT_SX`), so what someone typed looks the same once posted.
- **Anywhere HTML does not belong, use the text.** Previews, notifications, search, CSV exports, AI and MCP tool output, and push messages use the visible text (`richTextToPlainText`), never the HTML and never HTML with the tags stripped crudely (paragraphs would run together).
- **Emails** may show the formatting: run the text through the same sanitiser and make relative image paths absolute against the product's URL.

## Emoji and Reactions

### The emoji picker

One shared picker (e.g. `EmojiPicker`): a popover grid of a short, fixed set in labelled groups (Reactions, Smileys, Gestures, Objects), not the full Unicode catalogue. It is used by the editor's emoji button and by the reaction row.

### Reactions

Anything people converse on (a comment, a reply, a recognition post) has the shared reaction row (e.g. `ReactionBar`, or `CommentReactions` when the API returns one row per person):

- A chip per emoji used, with its count. The viewer's own reactions are filled; clicking a chip adds or takes back theirs.
- While there are no reactions, a **one-click thumbs up** (👍), the most common answer, sits ahead of the "Add reaction" button.
- **Add reaction** opens the shared emoji picker.
- Each chip's accessible name says the emoji, the count, who reacted when known, and what clicking does ("👍 2, from You, Alex. Remove your reaction").
- The change shows at once; the server's answer replaces it, and a failure puts the previous state back.

API rules:

- One endpoint toggles: `POST .../comments/:id/reactions` with `{ "emoji": "👍" }` adds the caller's reaction, or removes it if they already had it, and returns the reactions as they now stand. An existing add/remove pair may stay.
- **Validate the emoji**: one or more pictographs with skin-tone modifiers and joiners only, at most 32 characters; never free text, never digits alone. Share the validator (e.g. `ToggleReactionDto`).
- One row per (record, person, emoji), with a unique key on the three and an `upsert`, so a double click cannot fail.
- The same access rule as reading the record: someone who cannot see a comment (another customer's ticket, a staff-only note) gets a 404, never a 403 that confirms it exists.
- Reactions do not send notifications.

## Shared Implementation in a Monorepo

| Piece | Lives in | Example name |
| ----- | -------- | --------------- |
| Comment editor | UI package | `RichTextEditor` |
| Article editor | UI package | `BlogEditor` |
| Display | UI package | `RichTextContent` |
| Emoji picker | UI package | `EmojiPicker`, `QUICK_REACTIONS` |
| Reaction row | UI package | `ReactionBar`, `CommentReactions` |
| Sanitise, normalise, plain text, email HTML | Backend package | `normaliseRichText`, `sanitiseRichText`, `richTextToPlainText`, `richTextToEmailHtml` |
| DTO decorator | Backend package | `RichTextField({ label })` |
| Reaction validation | Backend package | `ToggleReactionDto`, `REACTION_EMOJI_PATTERN` |

Apps pass configuration (placeholder, minimum height, image upload function, people to mention, submit handler). They never copy a component to change one detail; they add a prop with a default that leaves every other app unchanged.

## Testing

- Server: sanitising removes scripts, event handlers, styles and `javascript:` links; plain text becomes escaped paragraphs; an emptied editor fails "required"; an over-long value fails; the reaction validator accepts emoji (including skin tones) and refuses text and digits.
- Access: a reaction on a comment the caller cannot see is a 404 and writes nothing.
- UI: the reaction row's thumbs up, chip toggle and accessible names; the display component escapes older plain text and strips script.
- Test runners that stub the editor library must stub custom nodes too (`Node.create`, `mergeAttributes`), or importing the UI package fails.

## Checklist

On any change to a field people write in:

- [ ] Is it conversation or a working description? Then it uses the comment editor, with the standard toolbar including emoji.
- [ ] Is it long-form published content? Then the article editor. Short or sent elsewhere? Plain text.
- [ ] Does every write path (REST, MCP, import) sanitise or normalise the HTML?
- [ ] Is it shown only through the shared display component?
- [ ] Do previews, notifications, search, exports and agent tools use the plain text?
- [ ] Do comments have the shared reaction row, with the one-click thumbs up?
- [ ] Is the reaction emoji validated and the endpoint access-checked like the record?
- [ ] UK English in copy; no em dashes.
