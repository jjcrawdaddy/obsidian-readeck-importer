# Flat Storage with Frontmatter ID Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Store synced bookmark notes flat in `{folder}/{title}.md` with a `readeck_id` frontmatter field for sync tracking, and route images to a separate configurable folder.

**Architecture:** Add an `imagesFolder` setting; write notes flat in `{folder}/`; embed `readeck_id` in frontmatter; scan frontmatter on each sync to locate existing notes instead of checking folder paths; store images at `{imagesFolder}/{bookmark-id}/{filename}`.

**Tech Stack:** TypeScript, Obsidian Plugin API (`app.vault`, `app.metadataCache`), esbuild

---

### Task 1: Add `imagesFolder` to settings interface

**Files:**
- Modify: `src/interfaces.ts`

**Step 1: Add the field to `ReadeckPluginSettings`**

In `src/interfaces.ts`, add `imagesFolder` after `folder`:

```typescript
export interface ReadeckPluginSettings {
	apiUrl: string;
	apiToken: string;
	username: string,
	folder: string;
	imagesFolder: string;   // <-- add this
	lastSyncAt: string;
	autoSyncOnStartup: boolean;
	overwrite: boolean;
	delete: boolean;
	mode: string;
}
```

**Step 2: Add default value in `DEFAULT_SETTINGS`**

```typescript
export const DEFAULT_SETTINGS: ReadeckPluginSettings = {
	apiUrl: "",
	apiToken: "",
	username: "",
	folder: "Readeck",
	imagesFolder: "",       // <-- add this (empty = use folder value)
	lastSyncAt: "",
	autoSyncOnStartup: true,
	overwrite: false,
	delete: false,
	mode: "text",
}
```

**Step 3: Build to verify no type errors**

Run from repo root: `npm run build`
Expected: exits with no errors

**Step 4: Commit**

```bash
git add src/interfaces.ts
git commit -m "feat: add imagesFolder to plugin settings"
```

---

### Task 2: Add `imagesFolder` input to settings UI

**Files:**
- Modify: `src/settings.ts`

**Step 1: Add the setting after the Folder setting**

In `src/settings.ts`, after the existing `Folder` Setting block (ends at line ~136), add:

```typescript
new Setting(containerEl)
    .setName('Images folder')
    .setDesc('Folder where images are saved. Defaults to the notes folder if left empty.')
    .addText(text => text
        .setPlaceholder('e.g. References/Images')
        .setValue(this.plugin.settings.imagesFolder)
        .onChange(async (value) => {
            this.plugin.settings.imagesFolder = value;
            await this.plugin.saveSettings();
        }));
```

**Step 2: Build to verify**

Run: `npm run build`
Expected: exits with no errors

**Step 3: Commit**

```bash
git add src/settings.ts
git commit -m "feat: add images folder setting to UI"
```

---

### Task 3: Add `readeck_id` to frontmatter generation

**Files:**
- Modify: `src/bookmarks.ts`

**Step 1: Update `generateBookmarkHeader` to accept and emit `readeck_id`**

Change the method signature and add `readeck_id` as the first frontmatter field:

```typescript
private generateBookmarkHeader(id: string, bookmark: Bookmark): string {
    let header = `---\n`;
    header += `readeck_id: "${id}"\n`;
    if (bookmark.title) {
        header += `title: "${bookmark.title.replace(/"/g, '\\"')}"\n`;
    }
    // ... rest of the fields unchanged ...
    header += `---\n`;
    return header;
}
```

**Step 2: Update the call site in `getReadeckData`**

Find this line (around line 118):
```typescript
const bookmarkHeader = this.generateBookmarkHeader(bookmark.json);
```

Change to:
```typescript
const bookmarkHeader = this.generateBookmarkHeader(id, bookmark.json);
```

**Step 3: Build to verify**

Run: `npm run build`
Expected: exits with no errors

**Step 4: Commit**

```bash
git add src/bookmarks.ts
git commit -m "feat: add readeck_id field to note frontmatter"
```

---

### Task 4: Add frontmatter scan method to `BookmarksService`

**Files:**
- Modify: `src/bookmarks.ts`

**Step 1: Add import for `MetadataCache` (already available via `app.metadataCache`, no new import needed)**

**Step 2: Add the `buildReadeckIdMap` private method**

Add this method to `BookmarksService` (after the constructor, before `getReadeckData`):

```typescript
private async buildReadeckIdMap(): Promise<Map<string, TFile>> {
    const map = new Map<string, TFile>();
    const folder = this.app.vault.getAbstractFileByPath(this.settings.folder);
    if (!folder || !(folder instanceof TFolder)) return map;

    for (const child of folder.children) {
        if (!(child instanceof TFile) || child.extension !== 'md') continue;
        const cache = this.app.metadataCache.getFileCache(child);
        const readeckId = cache?.frontmatter?.readeck_id;
        if (readeckId) {
            map.set(String(readeckId), child);
        }
    }
    return map;
}
```

**Step 3: Build to verify**

Run: `npm run build`
Expected: exits with no errors

**Step 4: Commit**

```bash
git add src/bookmarks.ts
git commit -m "feat: add frontmatter scan to build readeck_id lookup map"
```

---

### Task 5: Change markdown file path and update write logic

**Files:**
- Modify: `src/bookmarks.ts`

**Step 1: Add a helper to get the effective images folder**

Add this private getter to `BookmarksService`:

```typescript
private get effectiveImagesFolder(): string {
    return this.settings.imagesFolder?.trim() || this.settings.folder;
}
```

**Step 2: Rewrite `addBookmarkMD` for flat storage**

Replace the existing `addBookmarkMD` method with:

```typescript
private async addBookmarkMD(
    bookmarkId: string,
    bookmarkTitle: string,
    bookmarkContent: string | null,
    bookmarkAnnotations: Annotation[],
    existingFile: TFile | undefined,
) {
    const safeTitle = Utils.sanitizeFileName(bookmarkTitle);
    const filePath = `${this.settings.folder}/${safeTitle}.md`;
    let noteContent = bookmarkContent || '';
    if (bookmarkAnnotations.length > 0) {
        const annotations = this.buildAnnotations(bookmarkId, bookmarkAnnotations);
        noteContent += `\n${annotations}`;
    }

    if (existingFile) {
        // Rename if title changed
        if (existingFile.path !== filePath) {
            await this.app.fileManager.renameFile(existingFile, filePath);
        }
        if (this.settings.overwrite) {
            await this.app.vault.modify(existingFile, noteContent);
            new Notice(`Readeck importer: Overwriting note for ${bookmarkTitle}`);
        } else {
            new Notice(`Readeck importer: Note for ${bookmarkTitle} already exists`);
        }
    } else {
        await this.app.vault.create(filePath, noteContent);
        new Notice(`Readeck importer: Creating note for ${bookmarkTitle}`);
    }
}
```

**Step 3: Build a `readeckIdMap` at the start of `getReadeckData` and thread it through**

In `getReadeckData`, after ensuring the bookmarks folder exists, add:

```typescript
const readeckIdMap = await this.buildReadeckIdMap();
```

**Step 4: Update the bookmark processing loop to use flat paths**

Replace the existing per-bookmark block (lines ~112–131) with:

```typescript
for (const [id, bookmark] of bookmarksData.entries()) {
    if (bookmark.text || bookmark.annotations.length > 0) {
        const existingFile = readeckIdMap.get(id);
        const bookmarkHeader = this.generateBookmarkHeader(id, bookmark.json);
        const bookmarkContent = bookmarkHeader + (bookmark.text || '');
        await this.addBookmarkMD(id, bookmark.json.title, bookmarkContent, bookmark.annotations, existingFile);
    }

    if (bookmark.images.length > 0 && bookmark.json) {
        const imgsFolderPath = `${this.effectiveImagesFolder}/${id}`;
        await this.createFolderIfNotExists(id, imgsFolderPath);
        for (const image of bookmark.images) {
            const filePath = `${imgsFolderPath}/${image.filename}`;
            await this.createFile(bookmark.json.title, filePath, image.content, false);
        }
    }
}
```

**Step 5: Remove the old `createFile` and `createFolderIfNotExists` calls that are now duplicated (the loop uses `addBookmarkMD` directly)**

**Step 6: Build to verify**

Run: `npm run build`
Expected: exits with no errors

**Step 7: Commit**

```bash
git add src/bookmarks.ts
git commit -m "feat: store notes flat in folder using frontmatter ID for lookup"
```

---

### Task 6: Update image path references in note content

**Files:**
- Modify: `src/bookmarks.ts`

**Context:** Readeck returns markdown with image paths like `./imgs/filename.jpg`. With the new structure, images live at `{imagesFolder}/{id}/filename.jpg`. We use `Utils.updateImagePaths` to rewrite these references before writing the note.

**Step 1: Update image path rewriting in the bookmark processing loop**

After building `bookmarkContent` but before passing it to `addBookmarkMD`, add path rewriting when images are present:

```typescript
let content = bookmark.text || '';
if (bookmark.images.length > 0) {
    const newImgPath = `${this.effectiveImagesFolder}/${id}/`;
    content = Utils.updateImagePaths(content, './imgs/', newImgPath);
}
const bookmarkHeader = this.generateBookmarkHeader(id, bookmark.json);
const bookmarkContent = bookmarkHeader + content;
await this.addBookmarkMD(id, bookmark.json.title, bookmarkContent, bookmark.annotations, existingFile);
```

**Step 2: Build to verify**

Run: `npm run build`
Expected: exits with no errors

**Step 3: Commit**

```bash
git add src/bookmarks.ts
git commit -m "fix: rewrite image paths for new flat storage layout"
```

---

### Task 7: Update delete logic

**Files:**
- Modify: `src/bookmarks.ts`

**Step 1: Replace folder-based delete with file + images-folder delete**

Replace the delete block in `getReadeckData` (currently using `deleteFolder`):

```typescript
if (this.settings.delete) {
    const toDeleteIds = bookmarksStatus.filter(b => b.type === 'delete').map(b => b.id);
    for (const id of toDeleteIds) {
        // Delete the markdown note (find by readeck_id)
        const noteFile = readeckIdMap.get(id);
        if (noteFile) {
            await this.app.vault.delete(noteFile);
            new Notice(`Readeck importer: Deleted note for bookmark ${id}`);
        } else {
            new Notice(`Readeck importer: Could not find note for bookmark ${id}`);
        }
        // Delete the images folder
        const imgsFolderPath = `${this.effectiveImagesFolder}/${id}`;
        const imgsFolder = this.app.vault.getAbstractFileByPath(imgsFolderPath);
        if (imgsFolder && imgsFolder instanceof TFolder) {
            await this.app.vault.delete(imgsFolder, true);
        }
    }
}
```

**Step 2: Build to verify**

Run: `npm run build`
Expected: exits with no errors

**Step 3: Commit**

```bash
git add src/bookmarks.ts
git commit -m "feat: update delete to remove note by readeck_id and images folder"
```

---

### Task 8: Remove unused `deleteFolder` method and clean up

**Files:**
- Modify: `src/bookmarks.ts`

**Step 1: Remove `deleteFolder` private method** (no longer called anywhere)

Delete the `deleteFolder` method:
```typescript
private async deleteFolder(id: string, path:string, showNotice: boolean = false) { ... }
```

**Step 2: Build final clean build**

Run: `npm run build`
Expected: exits with no errors

**Step 3: Commit**

```bash
git add src/bookmarks.ts
git commit -m "refactor: remove unused deleteFolder method"
```

---

### Task 9: Manual verification checklist

Test these scenarios in Obsidian with the plugin loaded from the worktree build:

- [ ] Fresh sync: notes appear as `{folder}/{title}.md` (no ID subfolder)
- [ ] Fresh sync with images: images appear at `{imagesFolder}/{id}/{filename}`, image links in note resolve correctly
- [ ] Re-sync (overwrite on): existing note is updated, not duplicated
- [ ] Re-sync (overwrite off): existing note is skipped with notice
- [ ] Title changed in Readeck: note is renamed on next sync
- [ ] Delete in Readeck (delete setting on): note deleted, images folder deleted
- [ ] Settings UI: Images folder input is visible and saves correctly
- [ ] Empty images folder setting: images fall back to notes folder
