# Flat Storage with Frontmatter ID Design

## Overview

Replace per-bookmark subfolders with flat markdown file storage, using a `readeck_id` frontmatter field for sync tracking. Add a configurable images folder setting.

## Motivation

Currently each bookmark creates a subfolder `{folder}/{bookmark-id}/` containing the markdown note and an `imgs/` subdirectory. Users want markdown files stored flat in `{folder}/` without the intermediate ID-named subfolders.

## Design

### New Setting: `imagesFolder`

Add `imagesFolder: string` to `ReadeckPluginSettings` (default: `""`). When empty, falls back to `folder`. Exposed in the settings UI as a text input alongside the existing folder input.

### Flat Markdown Storage

Markdown files are written to `{folder}/{title}.md` instead of `{folder}/{bookmark-id}/{title}.md`.

Every note includes a `readeck_id` field in its YAML frontmatter:

```yaml
---
readeck_id: "abc123"
title: "My Article"
url: "https://example.com"
...
---
```

### Sync Tracking via Frontmatter Scan

On each sync, before processing bookmarks, the plugin scans all `.md` files in `{folder}` and builds a `Map<string, TFile>` keyed by `readeck_id`. This replaces the folder-existence check.

- If a match is found and `overwrite` is enabled, the file is updated in place.
- If the bookmark title has changed, the existing file is renamed to match the new title.
- If no match is found, a new file is created.

### Image Storage

Images are stored at `{imagesFolder}/{bookmark-id}/{filename}`. The markdown content's image references are rewritten to point to this path (same logic as today via `Utils`, different root).

### Delete Behavior

When a bookmark is marked for deletion:

1. Scan `{folder}` for a file with matching `readeck_id` and delete it.
2. Delete `{imagesFolder}/{bookmark-id}/` if it exists.

Both steps are best-effort — missing files/folders emit a notice and do not abort the sync.

## Files Affected

- `src/interfaces.ts` — add `imagesFolder` to `ReadeckPluginSettings` and `DEFAULT_SETTINGS`
- `src/bookmarks.ts` — replace folder-based logic with frontmatter scan; update file/image paths; update delete logic
- `src/settings.ts` — add `imagesFolder` text input to settings UI
