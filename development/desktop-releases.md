# Desktop Releases

> Authoritative standard for releasing, distributing and auto-updating desktop apps, whatever framework builds them. Tauri and Electron apps meet the same contract.

Related standards: [Desktop Development (Tauri)](/standards/development/desktop-development), [Electron Development](/standards/development/electron-development), [CI/CD Pipelines](/standards/development/ci-cd-pipelines), [Security Guidelines](/standards/governance/security-guidelines).

## Purpose

Users should not be able to tell which framework built an app from how they get it or how it updates. When
each framework grows its own release path, one of them quietly falls behind: an app drops off the downloads
page, installed copies stop updating, or a signing key lives only in a CI secret and is lost with it. This
standard fixes one contract for every desktop app. The framework standards say how each framework meets it.

## Core Principles

1. **One contract, two implementations.** The feed, file names, publish order, signing and download links are
   the same for every app. Only the tool that writes the files differs.
2. **Change both or neither.** A change to the release contract updates this standard and both frameworks'
   release tooling, templates and documentation together. No framework is "the other one".
3. **Installed copies always update.** Every distributed build checks the feed and updates itself. A release
   that cannot reach installed copies is a bug, not a phase.
4. **Signing material is never single-homed.** Every signing key and password lives in the password manager as
   well as on the release machine. A key that exists only in a CI secret or one laptop will be lost.

## The contract

| Rule | Detail |
| ---- | ------ |
| Tagged versions | Each release is a version tag on `main`; every file that carries the version (`package.json`, and for Tauri `tauri.conf.json` and `Cargo.toml`) matches it |
| Built locally, one entry point | `release:local` in each app builds, signs, notarises, copies and publishes. No release workflow and no GitHub release; CI checks code but never builds installers |
| Archive copy | Every installer is copied to the releases folder (`$AXIOM_RELEASES_DIR`, else Google Drive `My Drive/vault/app releases/<Product Name>/<version>/`) |
| One self-hosted feed | Installers and update manifests publish to `https://updates.axiomstudio.io/<slug>/`, one folder per app, static files only |
| Versionless names | Feed files are named `<Product Name>-<os>-<arch>.<ext>` (`mac-arm64`, `mac-x64`, `mac-universal`, `win-x64`), so each release replaces the same URL and download links never change |
| Manifest last | The publish step checks that the manifest's version matches the app's and that every file it names exists, uploads the installers first and the manifest last |
| Least-privilege upload | Uploads go over SSH to a user that can only write into the feed folder (rrsync, write only, no deletes) |
| No caching surprises | The feed serves everything with `Cache-Control: no-cache` |
| Every Mac arch | A Mac release covers Apple silicon and Intel, as two arch builds or one universal build, and the manifest lists both |
| Signed and notarised | macOS: Developer ID, hardened runtime, notarised and stapled. Windows: signed when a certificate exists; unsigned builds say so |
| Background updates | The updater runs in packaged builds only, checks at launch, downloads in the background, applies on restart and never crashes the app on a feed error |
| Confirm before publishing | An upload reaches every installed copy at its next launch, so a person confirms each publish after the build has been launched and checked |
| Download pages from the feed | Download links (the portfolio downloads page, in-app download buttons) point at the feed's versionless names and read the version from its manifest. Adding an app to the feed adds it to the downloads page in the same change |
| Verify what users get | After publishing, fetch the manifest and check it shows the new version, and that each linked installer returns 200 |

## How each framework meets it

| Contract item | Electron (`@axiomstudio/electron-kit` apps) | Tauri (The Forge's `apps/*-desktop`) |
| ------------- | -------------------------------------------- | ------------------------------------ |
| Entry point | `npm run release:local` | `pnpm release:local` (`scripts/desktop-release-local.mjs`) |
| Feed address | `build.publish` `{ "provider": "generic", "url": ".../<slug>" }` | `plugins.updater.endpoints[0]` = `.../<slug>/latest.json` |
| Mac build | arm64 and x64 in one electron-builder run | One `universal-apple-darwin` build |
| Manifest | `latest-mac.yml`, `latest.yml` (electron-builder) | `latest.json` (Tauri updater format), written by the release script |
| Update payload | `.zip` and `.blockmap` | `.app.tar.gz`, minisign signature inside `latest.json` |
| Update signature | The Developer ID (Squirrel.Mac checks the code signature) | The app's minisign key (`~/.tauri/<slug>.key` and `.key.password`), plus the Developer ID |
| Publish script | `scripts/publish-release.mjs` (`release:publish`) | `scripts/desktop-publish-release.mjs` (`release:publish`) |
| Windows | NSIS built on the Mac, on the feed | MSI and NSIS built on a Windows PC, archive copy only (known gap) |
| Updater | The kit's `updater.ts` | `tauri-plugin-updater`, `check_for_update` at launch |

Both publish scripts share the host, the checks, the order, `--dry-run`, and the skips (`CI`, `AXIOM_UPDATES_SKIP=1`).
A fix to one is made to the other in the same change.

## Signing keys

- An updater key is generated once per app by a person, in their own terminal, with a non-empty password, and
  saved (key and password) in the password manager before the first release. Agents never generate, read or
  print private keys.
- Never replace the key of a released app: installed copies reject every update signed by another key. If a key
  is lost, the next release is installed by hand once, and the loss is recorded in the app's docs.

## Adding a desktop app

- [ ] Created from the framework's template (`axiom-electron-template`, or `pnpm create:desktop`), which already points at the feed
- [ ] Feed slug chosen (lower-case, the app's repo or folder name) and set in the feed address
- [ ] Updater key (Tauri) generated by a person and stored in the password manager
- [ ] First release built, launched, published after confirmation, and checked on the feed
- [ ] Added to the downloads page with its versionless installer names
- [ ] The app's docs say how it releases, linking here

## Known gaps

- Tauri Windows installers are not on the feed: Tauri cannot build them on a Mac, and the Mac run writes the manifest. Closing it means a Windows run that merges its entry into `latest.json` before publishing.
