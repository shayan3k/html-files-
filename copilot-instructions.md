# Copilot Instructions — Day Planner (`task_planner.html`)

> Scope: this file covers **`task_planner.html` only**. `markdown_viewer.html` lives in the same
> folder but is a separate tool. Do not change it when working on the planner.

## Purpose
A personal, offline **daily task planner** that the owner uses every day to plan, review and adjust
work. Everything runs from one local HTML file opened in Edge/Chrome. There is no server, build step
or install.

### Features
- **Day / Week / Month views.** The app always opens on **today**. Work is separated per day
  (`tasks.date = 'YYYY-MM-DD'`).
- **Tasks with subtasks** (one level). Each has a priority (0–3), notes and done/todo status.
  Parent completion syncs with its subtasks.
- Tasks can be dragged to another day. Unfinished tasks can be rolled to the next day or today,
  and the roll counter is tracked in `rollover`.
- **Progress card and rings** (`progCard`, `ringSvg`): a gradient arc; at 100% it turns green and
  shows a check badge. The ring has a 7-unit stroke and a tinted track. An inset disc (`.rv`) holds
  the value as `N<small>%</small>` plus a "DONE" caption (`.cap`, rings ≥70px only). The card shows
  `done / total`, a segmented `.pc-bar` (≤12 tasks; a single filled bar above that) and a status line.
- **Attachments.** Choose or drag & drop any number of files, of any type or size, onto a task
  card, week card or month item. Files are stored on disk and listed in the task's detail panel.
  PDFs, images, text, audio and video open in a new tab; other types download.
- **Ask Claude drawer** (`#bClaude`, `#aiPanel`, `AI` object, shortcut `A`): a chat UI only. Its history
  is kept in `localStorage['tp.ai.chat']`, capped at 60 messages. Sending builds a payload
  `{message, view, range, tasks[], history[]}` and calls the pluggable
  `AI.transport(payload, {signal, onText})`. When this is `null`, the drawer shows a "not connected" notice.
  The backend is not built yet; wire it by assigning `AI.transport`.
- **Themes:** light, dark, or system (the default, following `prefers-color-scheme`).
- **Preferences** (theme, view, showDone, weekMon, sidebar, expanded tasks) are kept in
  `localStorage['tp.settings']`.
- **Font:** Plus Jakarta Sans Variable (OFL), embedded as a base64 woff2 `@font-face`.

## Storage layout (next to the HTML file)
```
task_planner.html
task_planner.db          ← SQLite database (created automatically if missing)
files/
  task-<parentTaskId>/   ← attachments for that task and its subtasks
    report.pdf
    report (2).pdf       ← name clashes get " (n)" suffix
.db-backups/             ← timestamped copies written by the DB skill before each change
```
- Files are read and written with the **File System Access API**. On first run the user picks the
  folder containing the HTML with `showDirectoryPicker`.
- The directory handle is stored in IndexedDB (`day-planner` / `kv` / key `dir`). It is re-verified
  on each load; re-granting permission needs a user gesture, which is why a "Reconnect" modal
  appears.
- Saves are debounced (350 ms after `changed()`) and write the whole DB with
  `db.export()` → `createWritable()`. An IndexedDB backup copy is also kept.
- If the folder is not linked (or the browser is not Chromium), data is kept in browser storage and
  the `tp.unsynced` flag is set. Attachments **require** a linked folder (`ensureLinked`).

## Database schema (sql.js, SQLite in WASM, inlined)
- `tasks(id, parent_id → tasks.id, date, title, notes, status 'todo'|'done', priority 0-3, position REAL, rollover, created_at, updated_at, completed_at)`
- `attachments(id, task_id, name, path /* relative, e.g. files/task-9/a.pdf */, size, type, added_at)`
- `meta(key, value)`: `schema_version` = `'2'`, `created_at`.
- Schema creation and migration run in `openDatabase()` using `CREATE … IF NOT EXISTS` and a
  version bump. Always keep migrations additive and idempotent.
- ⚠️ `db.export()` resets pragmas, so **foreign-key cascades are not relied on**. Deletes cascade
  manually in `Tasks.remove`, which also collects attachment rows so that `Tasks.restore` can undo
  them.

## Data rules and invariants (source of truth: the `Tasks` model)
Anything that writes to the DB, whether in the app or an external script, must preserve these:

| Field | Rule |
|-------|------|
| `id` | INTEGER autoincrement. Never reuse or renumber ids. |
| `parent_id` | `NULL` for a top-level task, otherwise the id of a **top-level** task. Subtasks are only one level deep. |
| `date` | `'YYYY-MM-DD'` in local time. A subtask always has the same date as its parent. |
| `title` | Non-empty text. `notes` defaults to `''`. |
| `status` | `'todo'` or `'done'` (CHECK constraint). |
| `priority` | `0` none, `1` low, `2` medium, `3` high (CHECK constraint). |
| `position` | REAL sort key. New rows get `MAX(position)+1` within the same date with `parent_id IS NULL`, or within the same parent for subtasks. Drag-reorder may write fractional values. |
| `rollover` | Increment by 1 only when an **unfinished** task moves to a **later** date. |
| `created_at`, `updated_at`, `completed_at` | UTC ISO strings with milliseconds, e.g. `new Date().toISOString()` → `2026-10-02T01:05:57.452Z`. Set `updated_at` on every change. Set `completed_at` when a task is marked done and set it to `NULL` when it goes back to todo. |

Status sync:
- `setDone(parent, done)` also marks its subtasks done, keeping any existing `completed_at`.
- Reopening a parent does **not** reopen its subtasks.
- `syncParent` runs after any subtask change or new subtask:
  - When every child is done, the parent becomes done.
  - When any child is todo and the parent is done, the parent becomes todo.
- `move(id, date)` moves the task and its subtasks together, and sets `position` to `nextPos` on the
  new date.

## External DB edits and concurrency
- The page holds the DB **in memory** and rewrites the whole file on save, so writes made by
  another process are only safe while the tab is idle.
- `checkExternal` runs every 5 s and on window focus. It reloads silently when the file's
  `lastModified` is newer than `store.lastMod`, but **only if** there are no unsaved edits, no save is
  in progress, and the tab is visible. Otherwise the next save from the tab overwrites the external
  changes.
- Manual reload: **More (…) > Reload from disk** (`data-act="reload"` → `reloadFromDisk`).
- To change data without the UI, use the **`updating-task-planner-db` skill**:
  - Location: `%USERPROFILE%\.copilot\skills\updating-task-planner-db\`
    (`SKILL.md` and `tp_db.py`, Python stdlib only).
  - Allowed operations: list, show, search, add (with subtasks or from JSON), update
    title/notes/priority, status todo/done, move, rollover, plus `--dry-run`.
  - It **never deletes**. A SQLite authorizer blocks DELETE and DROP.
  - Before every write it saves a backup to `.db-backups\task_planner.<timestamp>.db`.
  - Do not run ad-hoc SQL against `task_planner.db`.

## Code map (inside the single `<script>` blocks)
| Area | Key symbols |
|------|-------------|
| Utils | `esc`, `icon`, `debounce`, `LS`, date helpers `keyOf/dateOf/addDays/todayKey/relDay` |
| DB | `openDatabase`, `q(sql, params)`, `exec`, `lastId` |
| Model | `Tasks` (byDates, add, update, setDone, move, remove → rows, restore(rows) …) |
| Attachments | `FILES_DIR`, `safeName`, `uniqueIn`, `pathHandle`, `Att` (list, counts, add, file, detach, restoreRows, purgeLater), `ensureLinked` |
| Storage | `store`, `linked`, `changed`, `saveSoon/saveNow`, `attachDir`, `chooseFolder`, `reconnect`, `reloadFromDisk`, `checkExternal`, `downloadDb` |
| UI | `render()` → day/week/month renderers, `ringSvg`, `progCard`, `attHtml`, toasts/modals, a single delegated click handler on `data-act` / `data-att` attributes |
| Drag & drop | Internal task drag uses `dragId`. External file drops use `FILE_TARGETS` and `isFileDrag` (requires `dragId == null` and the `Files` type), plus `#dragPill` |

Deleting an attachment or a task shows an **undo toast**. Files on disk are purged by
`Att.purgeLater` about 6.8 s later, only if the rows were not restored.

**Tooltips.** `[data-tip]` elements use a single floating `#tip` element (`position:fixed`,
`showTip`/`hideTip`). It is not a CSS `::after`, because `.task{overflow:hidden}` (needed for the
priority stripe) clipped it. A `MutationObserver` copies `data-tip` into `aria-label`.
**Menu.** `placeMenu()` keeps the `#bMore` dropdown 8px inside the viewport (horizontal clamp plus
`maxHeight`). It re-runs on resize.
**Badges.** `taskStatus(t, kids)` returns todo, doing, done or overdue. `statusBadge` and
`prioBadge` (signal bars `pBars`) are used on day cards, compact week cards (`.wbadges`) and
month items (`.mt.od`).
Test API note: `Tasks.add({title, date, parent, priority})` uses `parent`, not `parent_id`, and
returns no id.

## Editing guidance
- The file is about 1 MB because it inlines **sql-wasm.js, the WASM binary (base64) and the font
  (base64)** on a few huge lines. Never `view` it whole. Use `grep` plus `view_range`, and never
  reformat or rewrite those lines.
- Keep it a **single self-contained file**: no CDN or external requests, and it must keep working
  offline from `file://` or `http://localhost`.
- Design tokens are CSS custom properties on `:root` / `[data-theme=dark]` (`--bg`, `--panel`,
  `--text`, `--accent`, `--font-ui` …). Reuse them rather than hard-coding colours. The visual style
  matches `markdown_viewer.html` (glassy panels and a gradient accent).
- Every data mutation must call `changed()` and then `render()`.
- Escape all user text with `esc()` before injecting HTML. Never open `.html` or `.svg` attachments
  inline.

## Testing recipe
1. `python -m http.server 8765` in this folder, then open `http://localhost:8765/task_planner.html`
   with Playwright (Chromium).
2. Dismiss the onboarding modal (`#modal .btn`), then mock the folder with OPFS:
   ```js
   const root = await navigator.storage.getDirectory();
   const dir = await root.getDirectoryHandle('tp-test', { create: true });
   dir.queryPermission = dir.requestPermission = async () => 'granted';
   await attachDir(dir, { interactive: true });
   ```
3. Exercise the features with `Tasks.add(...)`, `attachFiles(id, [new File(['x'], 'a.pdf')])`, or a
   simulated `drop` `DragEvent` carrying a `DataTransfer`. Then check `dir/files/task-<id>/` and
   `q('select * from attachments')`.
4. Check that the console has no errors, and take light and dark screenshots.
5. Clean up: remove the OPFS test entry, delete the `day-planner` IndexedDB, clear localStorage, and
   delete `.playwright-mcp/`. **Never touch the real `task_planner.db` or `files/`** in this folder.
6. DB skill tests: copy the DB to `$env:TEMP` and always pass `--db <copy>`. Delete the copy and its
   `.db-backups` folder afterwards.

## Constraints
- Edge/Chrome desktop only, because the File System Access API is required for on-disk saving.
- Windows + PowerShell is the owner's environment.
