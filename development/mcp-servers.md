# MCP Servers

> Authoritative standard for building Model Context Protocol (MCP) servers that let AI assistants (Claude, ChatGPT, Cursor, VS Code, command-line agents and others) work with a product's data on a user's behalf.

## Purpose

An MCP server gives an AI application a set of tools that read and change real customer data. In a commercial product that is a new way into every customer's account, driven by a model that can be manipulated by the content it reads. This standard sets the rules that make such a server safe to offer: who it acts as, what it may reach, what the customer agreed to, what it must never do, and how that is proven.

It covers remote MCP servers reached over HTTP with OAuth sign-in. For using MCP servers in development tooling, see [Claude Code Standards](/standards/ai/claude-code-standards).

## Core Principles

1. **The AI acts as one user, in one account, never more.** Every call runs with the permissions of the person who connected it, inside the single account they chose, re-checked on every request.
2. **Explicit, informed consent.** No connection exists until the user has seen what is being granted and has agreed to the data notice. Consent is recorded and revocable.
3. **Least privilege by design.** Scopes are narrow and separately consented; the tool registry offers only what is needed; destructive and outward-facing actions are absent, not just discouraged.
4. **Everything the model sends is untrusted.** Tool arguments are validated strictly, every id is checked against the caller's scope, and nothing from the model can choose a recipient, a tenant or a user.
5. **Everything the model reads is data, not instructions.** Content from records, emails and forms may contain prompt injection; the server never lets it widen what a tool can do.
6. **Draft, never send.** Anything that reaches people outside the account (emails, messages, published content, automations) is created as a draft that a person reviews and releases in the product.
7. **Visible and accountable.** Every call is audited, every change is attributed to "the user via the app", and users and administrators can see and revoke connections.

## Architecture Overview

| Component | Responsibility |
| --------- | -------------- |
| **Protected resource** (the MCP endpoint) | Accepts JSON-RPC over HTTP, requires a bearer token, returns `401` with a `WWW-Authenticate` header pointing at its resource metadata |
| **Authorisation server** | OAuth 2.1 with PKCE: client registration, consent, tokens, refresh, revocation |
| **Grant** | The server-side record of one connection: user, account, app, scopes, resource, created, last used, revoked |
| **Caller context** | Built per request from the grant and the user row read fresh from the database; never from token claims alone |
| **Tool registry** | The complete list of tools. A tool that is not registered does not exist; tests are generated from this list |
| **Record guard** | The single path every id passes through: tenant filter, then the user's permission policy |
| **Audit log** | One row per call: tool, outcome, record ids touched, duration, row count; never arguments or results |

## Authentication (OAuth 2.1)

Follow the MCP authorisation specification and OAuth 2.1:

- **Public clients with PKCE only.** AI apps cannot keep a secret. Advertise only `none` for client authentication in the server metadata (`token_endpoint_auth_methods_supported`, `revocation_endpoint_auth_methods_supported`). Advertising secret-based methods you then refuse makes some clients fail registration.
- **Client registration.** Support Client ID Metadata Documents (the client id is an https URL to its metadata) and, where needed, Dynamic Client Registration. Validate redirect URIs strictly: https, or http to loopback for local tools. Refuse custom schemes you do not intend to support, and log every refused registration with its reason, because clients report these failures vaguely.
- **Protected resource metadata** (RFC 9728) at `/.well-known/oauth-protected-resource/<path>`. Its `resource` value **must equal the exact address users are told to paste**. Clients reject a mismatch and stop before signing in, which looks like a broken server. Generate both the address shown in your product and the metadata from the same configured public base URL; never derive the address from a frontend's API host.
- **Authorisation server metadata** (RFC 8414) with an `issuer` that is the exact public https address. Endpoints must be absolute https URLs even behind a proxy or tunnel; many clients refuse http.
- **Resource indicators** (RFC 8707). Each token is bound to one resource (one MCP server). A token for one server must be useless on another.
- **Short-lived access tokens, rotating refresh tokens.** Rate-limit the token and registration endpoints.
- **Cross-origin access.** Allow the browser-based clients you support to fetch metadata (CORS on the well-known endpoints), without opening the authenticated API to arbitrary origins.

## The Consent Screen

The consent screen is where a customer's user turns an AI app into an actor inside their account. In a commercial setting it is also the record that the customer was told what would happen to their data. Treat it as a legal and security control, not a formality.

### What it must show

| Element | Requirement |
| ------- | ----------- |
| **Who is asking** | The connecting app's name and the host it will return to, so a user can spot an impostor. Show the client's own domain, not just a name it supplied |
| **Who is granting** | The signed-in user's name and email |
| **Which account** | When a user belongs to more than one account, they choose exactly one. The connection is locked to it; connecting another account means another connection |
| **What is granted** | Each scope in plain language ("Read your records", "Update records and add notes", "Draft campaigns for you to review"), with what each allows and, where useful, what it does not |
| **Their permissions still apply** | A clear statement that the app can do no more than the user can, and stops when their access changes |
| **The data notice** | What data the app will receive and where it goes: that records and their content are sent to the AI provider the user has chosen and processed under that provider's terms; which categories are withheld (for example contact details, unless an administrator allows them); that the app never sends emails or messages itself; how to disconnect |
| **The commitments** | What the server will never do (send, schedule, delete, change subscriptions), stated as promises that the code enforces |
| **Approve and Deny** | Two equal, clearly labelled actions. Deny must return the user to the app with an `access_denied` result |

### Agreement

- **Agreement is an explicit act.** The user must tick a checkbox stating they have read the data notice and agree (for example "I understand that information from this account will be shared with the app I am connecting and its AI provider") before **Approve** is enabled. Pre-ticked boxes, implied consent or agreement buried in a link are not acceptable.
- **The server enforces it, not just the button.** The approval request carries the agreement, and the authorisation server refuses an approval without it, so a crafted request cannot skip the notice.
- **The user chooses the scopes.** Each scope can be unticked; the grant holds only what was left ticked.
- **Re-authenticate before approving.** Ask for the user's password (or a fresh sign-in through the identity provider) on the consent screen, so a borrowed or left-open session cannot connect an AI app to the account.
- **Record the agreement**: the user, account, app, scopes, the version of the notice shown, and the time. Keep it with the grant so it can be produced for an audit or a data-protection request. When the notice text changes materially, increase its version and ask again at the next connection.
- **Administrators decide first.** Connecting AI apps is a feature an account administrator switches on for the account and for each user; the consent screen is only reachable when both are on. Show a clear message, not an error, when it is not.
- **Commercial terms.** Where the product's customer agreement or data processing agreement does not already cover AI integrations, the account-level switch should require an administrator to accept the integration terms on the customer's behalf first, and that acceptance is recorded in the same way.
- **Accessibility and clarity.** The screen meets the accessibility standard, works on a phone, and uses plain language: no legal text in place of the summary, with a link to the full terms.

### After consent

- The grant records the scopes, account, resource and notice version.
- Every connection is listed for the user (and for administrators across the account) with the app, account, scopes, created and last used, and a **Disconnect** (or **Revoke**) action that takes effect on the next request.
- Connections are revoked automatically when the user's password is reset, the user is suspended or removed, their access to the account ends, or the account switches the feature off.

## User Scope and Permission Scope

Apply the [Authorisation and Data Scope](/standards/governance/authorisation-and-data-scope) standard in full. MCP-specific rules:

- **Build the caller per request** from the server-side grant and a fresh read of the user and their permissions. Never trust permission or tenant claims carried in the token; a token outlives a permission change.
- **Seal the tenant.** Once the request is authenticated, the account is fixed for its lifetime. Any attempt to change it, or to save a row for another account, throws.
- **The same policy as the product.** The MCP server uses the product's permission policy, not a copy. A permission must mean the same thing in the web app and through an AI app.
- **Offer tools by permission.** A tool the user can never use (because a permission is off) is not listed, so the model is not tempted to try. Each tool still checks at run time.
- **Scopes narrow, never widen.** OAuth scopes (for example read, write, drafts) cut down what a connection may do; they never grant more than the user's own permissions.
- **Feature switches per account and per user.** The account administrator controls whether AI connections are allowed at all, and which sensitive categories (such as contact details) may be shared.

## Tool Design

- **Closed schemas.** Every tool has a JSON Schema with `additionalProperties: false`, typed and bounded arguments (lengths, ranges, list sizes), and required fields marked. Reject unknown arguments rather than ignoring them, so a misspelt field is not silently dropped.
- **No tenant, user or permission arguments.** A tool never takes an account id, user id or role from the model. Users the model may name (an assignee, an account manager) are resolved and checked against the account.
- **Every id goes through the record guard**, including ids nested in payloads. Out-of-scope ids are "not found", never "forbidden".
- **Build structured settings yourself.** When a tool writes configuration (an automation step, a filter, a report), build it from the tool's typed arguments; never accept raw JSON from the model, so no extra field can be smuggled in.
- **Outbound recipients are never model-supplied.** Notifications go only to users of the account or the record's owners, selected by id, never to an email address or URL the model provides.
- **Create and edit; do not delete.** Deletes, bulk changes, exports, imports, subscription changes and anything irreversible are absent from the registry unless a product decision documents otherwise, with an extra confirmation step outside the AI.
- **Drafts for anything outward-facing.** Campaigns, messages, publications and automations are created switched off or unscheduled. The flags that make something live are never set by a tool, and a test asserts it.
- **Small, allow-listed results.** Return only the fields a tool needs, in a stable shape, with paging (a modest maximum such as 50 rows) and a size cap. Mark free text written by other people as data.
- **Readable errors.** Errors are categorised (invalid, not found, refused, conflict) with a message the model can act on, and never include stack traces, SQL or internal identifiers.
- **Reuse the product's write services.** Tools call the same application services as the web controllers, so a business rule fixed in one place is fixed for both.

## Data Protection and Prompt Injection

- **Minimise.** Send the model the least data that serves the request. Contact details and message bodies are withheld unless an administrator has allowed them, and a redaction step removes them from free text as a backstop.
- **Treat read content as hostile.** Notes, emails, form answers and web content may contain instructions. They can influence what the model asks for, but never what a tool permits: scope, recipients and outward actions are enforced server-side regardless of what the model requests.
- **No secrets in results.** Never return tokens, API keys, password hashes or internal configuration.
- **Respect the product's privacy promises.** Any notice shown to customers ("the AI never sees contact details", "the app never sends emails") must be enforced in code, and the wording changes only with the code that keeps it true.

## Rate Limits and Quotas

- Per-connection, per-user and per-account limits on calls per minute and per day, configurable per account by administrators.
- A limit returns a readable error naming the limit and when it resets.
- Count blocked attempts in the audit log as well as successful ones.

## Audit and Attribution

- **Audit every call**: grant, user, account, app, tool, outcome, duration, row count and the record ids touched. Never log arguments or results, which may hold personal data.
- **Attribute every change** to "the user via the app" in the record's own author fields and in the activity log, so an AI-made change is never mistaken for the user's own work.
- **Mark drafts made by an AI** so the product can show "Drafted by an AI app: review before sending".
- Alert on anomalies: a spike in refused calls, any tenant-mismatch assertion, repeated registration failures.

## Connection Instructions

Users connect AI apps by pasting an address; make that reliable:

- Show the exact server address (the one the resource metadata names) with a copy button, generated by the API from its configured public address.
- Give step-by-step instructions per client (web assistants, desktop apps, IDEs, command-line agents), including where the client stores the setting **for every project** (user or global scope) and **for one project or directory** (project or workspace scope), and which scope shares the setting with a team.
- State plainly that the product signs in with OAuth and does not issue API keys.
- Keep the in-product instructions and the published guide in step.

## Testing

- **Tenant isolation suite generated from the tool registry.** Seed two accounts, lock the caller to one, point every tool at the other's ids, and assert "not found" (or, for lists, that only own rows return). A registered tool without a case fails the build. Run it against the real database engine in CI, not only an in-memory provider.
- **Architecture tests** over the MCP code: no raw SQL, no disabling of global query filters, no access to the "last used account" of the user (the connection's account is the only one), no reference to send or publish paths, every tool has a unique name, a known scope and a closed schema with no tenant or user argument.
- **Draft safety tests** assert that no tool can set the flags that send, schedule or enable.
- **Policy tests** for each permission level, the administrator rule and restricted defaults.
- **OAuth tests** for metadata (auth methods, issuer, resource), registration validation and handler ordering in the authorisation library you use.

## Operations

- Log refused registrations, refused tokens and consent decisions with reasons.
- Monitor the metadata endpoints from outside, including that the advertised `resource` matches the published address.
- Document the connection lifecycle (connect, use, revoke, automatic revocation) for support staff.
- Version the consent notice and keep previous versions for audit.

## Checklist

### Authentication
- [ ] OAuth 2.1 with PKCE; only `none` advertised for client authentication
- [ ] Client ID Metadata Documents and/or Dynamic Client Registration, with strict redirect validation and logged refusals
- [ ] Protected resource metadata `resource` equals the address users paste; both come from one configured public URL
- [ ] Issuer and endpoints are exact public https URLs
- [ ] Tokens bound to one resource; short-lived access tokens; rotating refresh tokens

### Consent
- [ ] Shows the app and its return host, the signing-in user, the one chosen account and each scope in plain language
- [ ] Shows the data notice: what is shared, with whom, what is withheld, what the app will never do
- [ ] Requires an explicit, unticked-by-default agreement before Approve is enabled, and the server refuses approval without it; Deny returns `access_denied`
- [ ] Each scope can be unticked; the user re-authenticates before approving
- [ ] Records user, account, app, scopes, notice version and time with the grant
- [ ] Re-asks when the notice changes materially
- [ ] Reachable only when the account and the user have the feature switched on

### Scope
- [ ] Caller built per request from the grant and a fresh user row
- [ ] Tenant sealed for the request; cross-tenant saves throw
- [ ] The product's own permission policy applies to every tool
- [ ] Tools listed only when the user's permissions allow them
- [ ] Scopes narrow; they never exceed the user's permissions

### Tools
- [ ] Closed schemas; unknown arguments refused
- [ ] No tenant, user or role arguments
- [ ] Every id, including nested ones, through the record guard
- [ ] Structured settings built by the server; no raw configuration from the model
- [ ] No model-supplied recipients
- [ ] No deletes, sends, exports or subscription changes; outward-facing items are drafts
- [ ] Allow-listed, paged, size-capped results

### Accountability
- [ ] Every call audited without arguments or results
- [ ] Changes attributed to "user via app"; AI drafts marked
- [ ] Connections listed and revocable by users and administrators; revoked automatically on access changes
- [ ] Rate limits and quotas per connection, user and account

### Testing
- [ ] Isolation suite generated from the registry, run against the real database in CI
- [ ] Architecture, draft-safety, policy and OAuth tests

## References

- [Authorisation and Data Scope](/standards/governance/authorisation-and-data-scope)
- [Security Guidelines](/standards/governance/security-guidelines)
- [Data Governance & Privacy](/standards/governance/data-governance)
- [Claude Code Standards](/standards/ai/claude-code-standards)
- Model Context Protocol specification (authorisation): https://modelcontextprotocol.io/specification
- OAuth 2.1: https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- RFC 9728 OAuth 2.0 Protected Resource Metadata; RFC 8414 Authorization Server Metadata; RFC 8707 Resource Indicators; RFC 7591 Dynamic Client Registration
