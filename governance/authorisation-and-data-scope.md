# Authorisation and Data Scope

> Authoritative standard for deciding what each caller may see and change: tenant boundaries, per-user permission settings, and applying them consistently to every way data leaves or enters an application.

## Purpose

Most access-control failures are not a missing login. They are an endpoint that checks permissions on the detail page but not on the list, the search box, the export or the bulk action, so the same record is hidden in one place and handed out in another. This standard defines one model for authorisation and the places it must be applied, so that a permission means the same thing wherever a user (or an integration acting for them) reaches the data.

It applies to every API, background job that runs on a user's behalf, export, report, search index, integration and AI tool surface (see [MCP Servers](/standards/development/mcp-servers)).

## Core Principles

1. **One policy, applied everywhere.** Each permission is evaluated by a single shared policy object. Controllers, services, exports, search and AI tools all call it; none re-implement the rule.
2. **Scope the query, not the response.** Visibility rules are applied to the query before it runs (a `WHERE`), never by filtering results after loading them, and never only in the user interface.
3. **Deny by default.** A missing, unreadable or unknown permission grants nothing. A user whose record cannot be loaded gets no access, not full access.
4. **The tenant boundary is absolute.** No request may read or write another tenant's data, whatever the user's permissions within their own tenant.
5. **Never trust identifiers from the client.** The tenant, the acting user and the permissions come from the authenticated session and the database, never from the request body, query string or a token claim the client can influence.
6. **Hidden means not found.** A record the caller may not see is reported exactly as if it did not exist.
7. **Enforced by tests, not memory.** Architecture tests fail the build when an endpoint or tool skips its gate.

## Two Layers of Scope

| Layer | Question | Enforced by | Failure looks like |
| ----- | -------- | ----------- | ------------------ |
| **Tenant scope** | Which account's data is this request about? | A global query filter (or row-level security) on every tenant-owned table, keyed on the tenant from the session | One customer reading or changing another customer's records |
| **User scope** | Within that account, which records and actions does this user's permission settings allow? | The shared permission policy, applied to each query and each action | A restricted user seeing colleagues' records through a list, search or export |

Both layers apply to every request. User scope never widens tenant scope.

### Tenant scope rules

- Every tenant-owned table has a tenant filter applied automatically by the data layer. New tables get one in the same change that creates them.
- **Bypassing the filter is exceptional.** Any code that disables global filters (for example to reach soft-deleted rows or rows written by another system with a missing tenant id) must re-apply the tenant condition explicitly in the same query, and a comment must say why. An "ignore filters" call without an explicit tenant condition is a defect.
- Writes are checked too. A request bound to one tenant must be unable to save a row belonging to another; fail the save rather than trusting each caller.
- Raw SQL must include the tenant condition as a parameter. Prefer the ORM so the filter applies by itself.
- Background jobs that act for a user run with that user's tenant set explicitly and sealed for the job's lifetime.

### User scope rules

Model each permission as a **level** or a **flag**, and keep the vocabulary small and documented:

| Kind | Typical values | Meaning |
| ---- | -------------- | ------- |
| Level | `Any` / `Own records` / `None` | Which records an action covers: every record in the tenant, only records the user owns or manages, or none |
| Flag | on / off | Whether the user may perform an account-wide action (export, build reports, manage settings, see everyone's tasks) |

- **Ownership matches every identity the user has.** When a user can be represented by more than one id (a legacy system's id and the current one, an SSO subject and a local id, a primary and secondary owner field), the policy compares against all of them. Comparing a single id silently refuses users their own records.
- **Default to the most restrictive value.** If the permission type's zero or default value means "allow everything" (a common enum mistake), the policy object must initialise every setting to "none" explicitly so a forgotten field restricts rather than grants.
- **Administrator semantics are explicit and single-sourced.** Decide whether an administrator holds every permission regardless of stored values, implement that rule once in the policy, and make the user-management screen show the same thing. Do not leave stale stored values to decide what an administrator can do.
- **Account-wide versus record-level.** An action that defines something for the whole account (creating a category, a field, a report) needs the account-wide level (`Any`), not "own records".
- **Bulk actions need the bulk permission.** An action that changes many records at once (bulk reassign, bulk unsubscribe, delete with children) requires the `Any` level of the matching permission, and still changes only the records the caller may see.

## Where the Policy Must Be Applied

Every one of these is a separate way out of (or into) the data, and each needs the policy. Checking only the detail endpoint is the most common failure.

| Surface | Rule |
| ------- | ---- |
| **List and grid endpoints** | Start the query from the policy's scoped set (`policy.Visible(records)`), then apply the caller's filters, sorting and paging on top |
| **Single-record reads** | Load through the scoped set; a hidden record is 404 |
| **Related lists** | Notes, documents, emails, calls, tasks and activity on a record are visible only when the record is; scope them through the parent |
| **Counts, tabs and badges** | Count through the scoped set, or they leak the existence and volume of hidden records |
| **Search and type-ahead** | Scope every source, including any raw-SQL or index-backed matches, which must be filtered through the policy before returning |
| **Exports and downloads** | Require the export permission **and** scope the rows. Settings and metadata exports need the permission too |
| **Reports** | Report grids and report exports follow the report-specific rules (for example "own records only unless allowed to see all"), not just general visibility |
| **Updates and deletes** | Load the record through the scoped set (404 if hidden), then check the action's own level against the record's **current** owners, not the owners sent in the request |
| **Moving data between records** | When a change re-parents something (a note to another record), check the action on both the old and the new parent |
| **Bulk actions over client-supplied ids** | Narrow the ids to those the caller may act on, in the caller's tenant, before acting. Never act on ids exactly as sent |
| **Aggregates** | Decide and document whether totals (dashboards, forecasts) are account-wide or user-scoped. Totals that expose no records may stay account-wide; anything that lists records must be scoped |
| **Audit and activity logs** | If entries cannot be tied to a record id, restrict users without full visibility to their own entries |
| **Audiences** | A marketing audience or automation that reaches "everyone matching a filter" is defined by the account, not the viewer. Keep audience counting unscoped and say so in code, but scope any list of the people in it shown to a user |

## Responses

| Situation | Status | Body |
| --------- | ------ | ---- |
| Not signed in, or the session is invalid | `401 Unauthorized` | Generic |
| Signed in, the record exists but is outside the caller's scope (or tenant) | `404 Not Found` | Identical to a record that does not exist |
| Signed in, may see the record, but may not perform this action | `403 Forbidden` | A readable reason naming the setting, for example "Your permissions do not allow you to download records. Ask an administrator to switch on Download records." |

- Never return `401` for a permission refusal. Clients treat `401` as "signed out" and end the session.
- Return `403` with a reason the user can act on. Do not reveal other users' data or internal identifiers in it.
- Return `404` (not `403`) whenever a `403` would confirm that a hidden record exists.

## Identifiers in Requests

- **Never take the tenant id from input** (body, query string, route or a client-writable claim) to decide whose data a request sees. Derive it from the server-side session.
- **Every id in a request is untrusted**, including ids nested inside payloads (settings JSON, filter definitions, lists of recipients). Resolve each through the scoped set before use.
- **Identifiers used as SQL identifiers** (column or table names from stored configuration) cannot be parameterised. Validate them against a strict allow-list or a strict pattern (letters, digits and underscores only) and refuse anything else; parameterise every value that is not an identifier, including labels and names.

## User Interface

The interface hides what the API refuses, so users are not offered actions that fail. The interface is a convenience, never the control:

- Read permissions from the same values the API uses, with the same administrator rule.
- Treat level permissions as levels: "None" must not be coerced to `true` (a common bug when a string or enum is passed to a boolean conversion).
- Hide, rather than disable, actions the user can never take (export menus, bulk actions, create buttons), and show the API's reason when an action is refused at the record level.

## Testing and Enforcement

- **Policy unit tests** cover every level and flag, the administrator rule, multiple identities per user, and the restrictive default.
- **Enforcement tests** call real endpoints (or their handlers) with a restricted user and assert 404 for hidden records, 403 for refused actions, and that nothing was written.
- **Tenant isolation tests** seed two tenants and assert that every endpoint or tool pointed at the other tenant's ids answers "not found". Generate the cases from the route or tool registry so a new endpoint without a case fails the build.
- **Architecture tests** fail the build when, for example, an export route lacks the export gate, a settings write lacks its permission, or code disables global query filters without a reviewed exception.
- **Run isolation tests against the real database engine** in CI as well as any in-memory provider. In-memory providers often skip foreign keys, collation and query translation, which is exactly where tenant and permission bugs hide.

## Common Gaps to Look For

These are the defects found most often in audits:

1. The detail endpoint checks visibility; the list, search, export and report endpoints do not.
2. Update and delete check the user's role but not the record's owner, or check the owner sent in the request rather than the stored one.
3. Ownership compares one id where the user has two.
4. An "ignore filters" call (to reach soft-deleted rows) also removes the tenant filter.
5. Bulk endpoints act on whatever ids the client sends.
6. Raw-SQL search builds SQL from stored names (a field's label or column) instead of parameters and an identifier allow-list.
7. Exports have no permission check at all.
8. Permission refusals return `401`, logging the user out.
9. The permission enum's default value grants everything.
10. Administrators are restricted by stale stored values that the user-management screen shows as "on".

## Checklist

- [ ] One shared policy object evaluates every permission; nothing re-implements a rule
- [ ] Every tenant-owned table has a global tenant filter; every bypass re-applies the tenant condition
- [ ] Writes for another tenant fail at save time
- [ ] Lists, details, related lists, counts, search, exports, reports and bulk actions all go through the policy
- [ ] Ownership matches every identity the user has
- [ ] Unset permissions default to the most restrictive value
- [ ] The administrator rule is implemented once and matches the user-management screen
- [ ] Hidden records are 404, refused actions 403 with a reason, never 401
- [ ] No tenant id or user id is taken from request input
- [ ] Every nested id is resolved through the scoped set
- [ ] SQL identifiers from configuration are validated; values are parameterised
- [ ] The interface hides what the API refuses, using the same permission values
- [ ] Policy, enforcement, isolation and architecture tests exist, and isolation runs against the real database in CI

## References

- [Security Guidelines](/standards/governance/security-guidelines) (OWASP A01 Broken Access Control)
- [Data Governance & Privacy](/standards/governance/data-governance)
- [MCP Servers](/standards/development/mcp-servers)
- OWASP Authorization Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
