# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

The official Readwise plugin for Obsidian (`readwise-official`). It syncs highlights from Readwise
(Kindle, Instapaper, Pocket, articles, etc.) into an Obsidian vault as markdown files. Single-plugin
repo — no monorepo, no packages directory.

## Commands

```sh
npm run dev         # rollup --watch, for local development
npm run typecheck   # tsc --noEmit
npm test            # compiles tests then runs them via node's built-in test runner
npm run build       # production build -> main.js (bundled, requires READWISE_SERVER_URL env)
npm run ci          # typecheck + test + build, same as CI
npm run dist        # build + copy styles.css/main.js/manifest.json into dist/
```

Run a single test file directly once compiled:

```sh
npm run typecheck && tsc -p tsconfig.test.json && node --test .test-build/tests/paths.test.js
```

Tests are plain Node `node:test`/`node:assert` files under `tests/*.test.js`. They import compiled
output from `.test-build/src/...` (produced by `tsconfig.test.json`, CommonJS output), not from
`src/` directly — so if you edit `src/*.ts`, tests won't pick it up until the `tsc -p
tsconfig.test.json` step reruns (which `npm test` does automatically).

CI (`.github/workflows/ci.yml`) runs on Node 20 for every PR/push to main/master: typecheck, test,
build. `.github/workflows/release.yml` builds and drafts a GitHub release when a tag is pushed.

## Architecture

- `src/main.ts` — the entire plugin (~1000 lines): `ReadwisePlugin` (extends Obsidian's `Plugin`)
  plus `ReadwiseSettingTab`. Almost all logic lives here; the other `src/` files are small,
  extracted, independently-testable helpers.
- `src/errors.ts` — parses Readwise API error responses into a normalized `ReadwiseSyncError`
  (e.g. detects `account_expired`, and a `401` response into `code: "invalid_token"`, each with a
  friendly message).
- `src/paths.ts` — computes the vault path of the special "Readwise Syncs.md" file, which must be
  detected and excluded from JSON parsing (it's plain markdown, everything else in the export is
  JSON). Handles the vault-root base-folder edge case.
- `src/status.ts` — `StatusBar`, a small message queue for the Obsidian status bar (desktop only;
  mobile shows `Notice` popups instead).
- `src/secretStorage.ts` — `getToken`/`setToken`/`clearToken`/`migrateTokenToKeychain` helpers for
  storing the Readwise token in Obsidian's Keychain (`App.secretStorage`, 1.11.4+) instead of
  plaintext in `data.json`. No dependency on the `obsidian` package, so it's unit tested directly
  against duck-typed fakes rather than mocking Obsidian.

### Sync flow (all in `main.ts`)

1. `syncBookHighlights()` — entry point for every sync (manual command, settings button, scheduled
   interval, or auto-on-load). Decides whether to call `refresh_book_export` (when there are
   `failedBooks` / `booksToRefresh` to retry) before...
2. `queueExport()` — hits `/api/obsidian/init` to kick off an export archive server-side, gets a
   `latest_id` / status ID back.
3. `getExportStatus()` — recursively polls `/api/get_export_status` every second until the archive
   build finishes, downloading any already-ready artifacts along the way via...
4. `downloadArtifact()` — fetches a zip from `/api/v2/download_artifact/:id`, unzips in memory
   (`@zip.js/zip.js`), and for each entry: parses JSON (unless it's the "Readwise Syncs.md" file,
   which is appended raw), writes/creates the file, and tracks `book_id`/`reader_document_id` →
   file path in `settings.booksIDsMap`. Content hashing (`ts-md5`) is used to detect local edits
   before overwriting/appending, so user modifications to synced files aren't silently clobbered.
5. `acknowledgeSyncCompleted()` — POSTs `/api/obsidian/sync_ack` so the server knows this client
   has the data (scoped by `statusID` when available).

Book/document identity: raw Readwise book IDs are used as-is; Reader document IDs are prefixed
(`readerdocument:<id>`) via `encodeReaderDocumentId`/`decodeReaderDocumentId` so both ID spaces can
share the same string-keyed settings maps (`booksIDsMap`, `booksToRefresh`, `failedBooks`).

Settings (`ReadwisePluginSettings`, persisted via Obsidian's `loadData`/`saveData` to `data.json`)
double as sync state — `isSyncing`, `currentSyncStatusID`, `lastSavedStatusID`,
`booksToRefresh`/`failedBooks` queues are all persisted so retries survive an app restart. When
adding fields, add a fallback to `DEFAULT_SETTINGS` at the read site (see `addToFailedBooks`)
since existing users' `data.json` won't have new keys.

Vault event handlers (`rename`, `delete`) registered in `onload()` keep `booksIDsMap` in sync when
the user moves/deletes files outside the plugin; deletions also queue the book for re-sync when
`settings.refreshBooks` is enabled. Note the existing `on("delete")` code comment: Obsidian gives no
ordering guarantee across rapid multi-file deletes, so concurrent map mutations can race.

`baseURL` in `main.ts` is hardcoded to `https://readwise.io`; the comment above it notes to switch
it for local dev against a local server. `pluginVersion` there must be kept manually in sync with
`manifest.json`'s `version` (there's no build-time substitution).

### Token storage (Keychain vs. plaintext)

The Readwise API token can live in two places, tracked by `settings.keychainOnly`:

- **Plaintext** (`settings.token`) — the original behavior, persisted in `data.json`. Still the
  default for existing installs, and the fallback whenever Keychain isn't available.
- **Obsidian Keychain** (`App.secretStorage`, added in Obsidian 1.11.4) — a per-device, OS-backed
  store. Entries do **not** sync between devices, so `main.ts` never auto-migrates an existing
  install; it only auto-enables Keychain-only mode on a genuinely fresh install (`isFreshInstall()`
  in `secretStorage.ts`, detected by `loadData()` returning `null`/`undefined`). Existing users opt
  in via a settings-tab "Move to Obsidian Keychain" action that surfaces the multi-device caveat
  before touching anything.

All reads/writes go through `getToken()`/`setToken()`/`clearToken()` in `secretStorage.ts`, which
feature-detect `app.secretStorage` (`hasSecretStorage()`) and fall back to plaintext transparently —
so no `minAppVersion` bump was needed. `isStranded()` detects the case where a vault was set to
Keychain-only on one device but is opened on another Obsidian build without Keychain support; the
settings tab shows a distinct "update Obsidian" message there instead of the generic "Connect"
prompt, since the token isn't actually gone, just inaccessible on this device.

A `401` from the Readwise API is reported by `errors.ts` as `code: "invalid_token"`; `main.ts`
responds by clearing the stored token (via `clearToken()`) and prompting the user to reconnect,
rather than leaving a dead token in place to keep failing the same way.

Design reference: several of the above patterns (fresh-install detection, the `deleteSecret`
runtime-vs-type-declaration gap, opt-in-only migration, and the stranded-vault case) were validated
against `logancyang/obsidian-copilot` PR #2364, which solved the same migration for a
multi-secret plugin — see the doc comments in `secretStorage.ts` for exact file/line references.

## Releasing

See `PUBLISHING.md` for the full process: version-bump PR (`manifest.json`, `versions.json`,
`package.json`/`package-lock.json` via `npm version`) merged to master first, then tag to trigger
the release workflow, then manually un-draft the GitHub release (Obsidian can't install from a
draft).
