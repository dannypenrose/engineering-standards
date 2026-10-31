# File Uploads

> Authoritative standards for storing, serving and deleting files users upload: profile pictures, attachments, images in rich text and any other user-supplied file.

## Purpose

Uploaded files outlive their code paths easily. A row is deleted, a cascade
fires, an image is swapped, and the file stays on disk: still publicly
reachable by its address, still taking space, and invisible to everyone. This
standard makes every upload follow one storage path and one lifecycle, so a
file exists exactly as long as the thing that owns it.

## Core Principles

1. **One owner.** Every uploaded file belongs to exactly one record (a card, a ticket, a comment, a user, an article).
2. **Deleted with its owner.** When the owner is deleted, or stops pointing at the file (a replaced picture, an image removed from an article), the file is deleted too. No exceptions outside a media library (see below).
3. **One storage path.** Every app stores files through one shared helper into one configured root, so the storage backend can change in one place.
4. **Store paths, not locations.** The database keeps the public path (`/uploads/<area>/<file>`), never a disk path, a bucket key or a full URL with a domain.
5. **Trust content, not names.** File types are checked from the file's bytes; names are generated, never taken from the user.

## Storage

| Rule | Detail |
| ---- | ------ |
| One root per app | A single setting (for example `UPLOADS_DIR`) names the root. Never a per-feature setting |
| One folder per area | `avatars`, `cards`, `tickets`, `kb`, `files`: an area is a kind of file, named in lowercase |
| Through the shared helper | Save and delete only through the shared storage functions (`saveUpload`, `deleteUpload`, `deleteUploads` in `@scope/backend-core`). No `fs.writeFile` or SDK calls in feature code |
| Generated names | A random name keeping only a plain extension (`newUploadFilename`). The original name may be stored separately for display |
| Public by design | Everything under the root is served publicly at `/uploads/<area>/`. Private files (documents only the backend reads) live outside it, with their own setting |
| Kept by deploys | The root is outside anything a deploy replaces (for example, the backend's `data/` folder, which every deploy excludes) |

Moving to blob storage (Azure, S3) changes the shared helper and where
`/uploads/` points. Because stored paths carry no location, no data changes.

## Lifecycle: delete with the owner

### The rule

Code that deletes or changes an owner deletes the files it no longer needs,
**in the same operation**:

| Event | What happens to the files |
| ----- | ------------------------- |
| Owner deleted | Every file it owns, including those of children removed by a database cascade (comments, checklist items, every card on a deleted board) |
| File replaced (a new picture, logo or attachment) | The old file |
| File removed from rich text (an article edited) | Files the content no longer links to, unless another record still links to them |
| Account deleted | The account's profile picture and any other files only it owned |

### How

1. **Read** the stored names of every file the operation will orphan, before the delete (cascades remove the rows that name them).
2. **Delete** the database rows.
3. **Delete** the files, best effort, only after the database change succeeded. A missing file is not an error; a failed database change leaves the files in place.

```typescript
// Collect first: the cascade removes the attachment rows
const files = await cardAttachmentFiles(prisma, { column: { boardId: id } });
await prisma.board.delete({ where: { id } });
await deleteUploads('cards', files);
```

Never delete files first: a failed database delete would leave rows pointing
at missing files.

### Rich text

Files embedded in HTML (images in an article or a description) are not
recorded in a table, so they are found by their paths in the content
(`uploadFilenamesInHtml`). Before deleting one, check no other record still
links to it.

### The backstop: sweeping orphans

Some deletes cannot see every file they orphan, for example a shared module
deleting a user whose rows cascade into app tables. Every app therefore has an
upload sweep that compares each area's folder with the names its database
still uses:

- **Report by default**, delete only with `--apply`.
- **Leave recent files alone** (one day by default): an upload can land before the form that uses it is saved.
- **Run it** after bulk deletes and account deletions, and on a schedule.

The sweep is a safety net, not the mechanism. A delete path that relies on it
is a bug.

## Exception: media libraries

A media library, where users pick an existing file to reuse in several places,
is the only exception: there a file's owner is the library entry, not the
records that use it. Such a feature must:

- Keep files in its own area, with a table row per file.
- Delete a file only when its library entry is deleted, and refuse (or warn) while records still use it.
- Be the only way files in its area are reused.

## Validation

| Check | Rule |
| ----- | ---- |
| Type | Read from the file's first bytes (magic numbers), never from the name or the browser's content type |
| SVG | Refused for anything shown inline: it can carry script and is served from the app's own origin |
| Size | Limited per area, and enforced before the file is written |
| Images for display | Shrunk in the browser before upload where possible (e.g. profile pictures cropped to a square) and still validated on the server |
| Serving | `X-Content-Type-Options: nosniff` on the uploads location |

## Testing

Every delete or replace path that touches an owner of files has a test that:

1. Writes real files to a temporary uploads root.
2. Runs the delete or update.
3. Asserts the owned files are gone and unrelated files remain.

## Checklist

- [ ] Files are saved and deleted only through the shared storage helper
- [ ] Every delete of an owner (including cascades) deletes its files after the database change
- [ ] Replacing or removing a file deletes the old one
- [ ] Rich text images are cleaned up on edit and delete
- [ ] The app's upload sweep covers every area it writes to
- [ ] Private files live outside the public uploads root
- [ ] Tests prove owned files are deleted and others are kept

## Related

- [Data Governance](/standards/governance/data-governance)
- [Security Guidelines](/standards/governance/security-guidelines)
- [Deployment Standards](/standards/development/deployment-standards)
