# File > Reload (Ctrl+R) — Rubber-Duck Walkthrough

This is a pre-implementation explanation of the feature, written out loud to catch
inconsistencies before touching `src/main.cpp`. It corresponds to ToDo item #11
(not committed; tracked in the developer's local `ToDo.txt`).

**Revision note:** this document went through one round of code review after the
first draft below was written. The review caught several real bugs in the
original design (not just wording issues). Rather than silently rewrite the
original reasoning, the corrections are appended in a dedicated section near the
end (`## Corrections after code review`), so the audit trail of what was wrong
and why stays visible. Read the original walkthrough first, then the
corrections section, which supersedes it wherever they disagree.

## What the feature does, in one sentence

Re-reads the file currently backing the open document from disk, discarding any
unsaved in-memory edits, while trying to put the caret back where it was — even if
the content shifted underneath it.

## Why this needs more than "call LoadTextFile again"

`LoadTextFile()` already does the read/decode/set-text work (it's what `FileOpen`
uses). But it has three behaviors that are wrong for a *reload*, unless corrected
by the caller:

1. It unconditionally resets the caret to position 0 (`EM_SETSEL 0,0`) — a reload
   should put the caret back where the user was, not at the top of the file.
2. It unconditionally sets `g_bModified = FALSE` — correct for reload (we *are*
   discarding changes), so no correction needed here, just confirming it's fine.
3. When `bClearResumeState=TRUE`, it adds the reloaded path to MRU and clears any
   active resume-file state. For a plain reload of the *same* file this would be a
   harmless MRU bump — but for a resumed file it would wrongly delete the resume
   temp file mid-workflow. Passing `FALSE` sidesteps this entirely.

So the wrapper function (`FileReload`) has to: capture position before, call
`LoadTextFile` with `bClearResumeState=FALSE`, then fix the caret after.

## Why "capture position" isn't just "remember the character offset"

If the file changed on disk (the whole point of this feature — e.g. an external
process rewrote it), a raw character offset can now point at completely different
text. Fortunately this exact problem was already solved for bookmarks:
`RefreshBookmarkLineIndices` re-locates a bookmark whose surrounding text no
longer matches at its old offset, using a three-step fallback:

1. `BookmarkContextMatchesAt(pos, context)` — is the same short text still right
   there? (fast path — true for the overwhelming majority of reloads, since most
   edits happen elsewhere in the document)
2. `FindContextNearPos(pos, context, 8192)` — search a window around the old
   position (handles "a few lines were added/removed above me")
3. `FindContextInDocument(context)` — search the whole document (handles large
   shifts)
4. If all three fail (the line was deleted, or the document is empty), fall back
   to the old numeric offset, clamped to the new document length.

`GetLineContextFromCharPos` is the function that captures the "short text"
context in the first place (used by `ToggleBookmark` today). Reusing all four of
these functions means the reload's position-restoring logic is a small, thin
wrapper around already-tested code, not a new parallel implementation.

One thing to double check: these are `static` functions defined together around
lines 7028–7291, appearing *before* `FileOpen`/`FileSave` (~8808) in the file.
Since I'm placing `FileReload()` near `FileOpen`/`FileSave`, it's *after* these
statics are already defined — no forward declaration needed for them. This does
mean `FileReload()` cannot be placed earlier in the file (e.g. near the top)
without adding forward declarations; that's a minor ordering constraint, not a
blocker.

## Why the resumed-file case needs special handling at all

A "resumed" document (`g_bIsResumedFile == TRUE`) is unusual: what's currently in
the editor did **not** come from `g_szFileName` on disk. It came from a separate
temp file (`g_szResumeFilePath`), and `g_szFileName` (the thing shown in the
title bar) is either:
- the path the file *would* be saved back to (if it was previously a saved file
  when the crash/shutdown happened), or
- empty (if it was an untitled document when the crash/shutdown happened).

If `FileReload` naively used `g_szFileName` as the reload target here, it would
either silently fetch a completely different file's content (the pristine,
pre-crash version at the original location) or — for the untitled case — have
nothing to reload from at all despite there clearly being recovered content on
screen.

The developer confirmed the intent: reload should target **the resume temp file
itself** (`g_szResumeFilePath`), not the original location. This makes Reload
mean "revert to the recovered/crash-time snapshot, discarding whatever I've typed
*during this session since recovery*" — which is actually a coherent, useful
action, and doesn't require the original file to be touched at all (consistent
with the existing design principle: "Original files remain untouched until you
explicitly save").

Getting the pristine pre-crash original (at `g_szOriginalFilePath`) instead is
still possible — just use plain `File > Open` on that path. No new command is
needed for that; the developer confirmed this explicitly.

## The tricky part: `LoadTextFile` clobbers globals I need to keep

`LoadTextFile(path, ...)` sets `g_szFileName` and `g_szFileTitle` to whatever
path is passed in, and always sets `g_bModified = FALSE`. If I pass
`g_szResumeFilePath` (e.g. `...\RichEditor\notes_resume.txt`) directly, after the
call:
- the title bar would show `notes_resume.txt` instead of the correct original
  name (or "Untitled"),
- `g_bModified` would be `FALSE`, incorrectly implying the recovered content now
  matches something safely saved somewhere, when it's still only a temp-file
  snapshot,
- `g_bIsResumedFile` itself isn't touched by `LoadTextFile`, so it would remain
  `TRUE` — that part's fine and needs no correction.

So for the resumed branch, I save `g_szFileName`, `g_szFileTitle`,
`g_szResumeFilePath`, and `g_szOriginalFilePath` into locals *before* calling
`LoadTextFile`, and restore them (plus re-force `g_bModified = TRUE`)
*afterward*. This exact save/restore-around-a-load-or-save-call pattern already
exists in the codebase (`WriteResumeFileContent` does it around `SaveTextFile`
for the reverse direction), so it's not a new pattern, just applied to
`LoadTextFile` instead of `SaveTextFile`.

Buffer sizes matter here and I initially got them wrong in my head: `g_szFileName`
is declared `WCHAR[EXTENDED_PATH_MAX]` (32767), **not** `WCHAR[MAX_PATH]` — I
have to size my local save buffers to match, or risk silent truncation on a long
path. `g_szFileTitle` genuinely is `WCHAR[MAX_PATH]` (it only ever holds a bare
filename, not a full path), so that one local can stay smaller. Interestingly,
some *existing* code (`FinalizeSuccessfulSave`) calls `wcscpy_s(g_szFileName,
MAX_PATH, ...)` — using `MAX_PATH` as the bound for a buffer that's actually
`EXTENDED_PATH_MAX`-sized. That looks like a pre-existing inconsistency (it just
means paths longer than 260 chars would be truncated there, not a buffer
overflow, since `MAX_PATH` is smaller than the true capacity — so it's a
functionality limitation, not a memory-safety bug). It's out of scope for this
feature and I won't touch it, but my new code will consistently use
`EXTENDED_PATH_MAX` for anything mirroring `g_szFileName`,
`g_szResumeFilePath`, or `g_szOriginalFilePath`.

## Does the "enable/disable" rule correctly cover every case?

Rule: enabled when `g_szFileName[0] != '\0'` **or** `g_bIsResumedFile`.

Walking through every reachable state:
- Untitled, never resumed (`g_szFileName` empty, `g_bIsResumedFile` FALSE) →
  disabled. Correct — nothing on disk anywhere to reload from.
- Saved file, no crash history → `g_szFileName` non-empty → enabled. Correct.
- Resumed, was previously a saved file (`g_szFileName` = original path,
  `g_bIsResumedFile` TRUE) → enabled via either half of the OR. Correct, and the
  *target* logic (resume file, not original) takes over from here.
- Resumed, was previously untitled (`g_szFileName` empty, `g_bIsResumedFile`
  TRUE) → enabled via the second half of the OR. Correct — there's still a
  resume temp file backing it even though there's no "original" location.

I don't think there's a reachable state this rule gets wrong. One assumption
this rule leans on: `g_bIsResumedFile == TRUE` always implies
`g_szResumeFilePath` is non-empty. Checking where `g_bIsResumedFile` is ever set
to `TRUE`: the startup-recovery code and the "Open Resume File" submenu handler
both set `g_szResumeFilePath` in the same breath as the flag, so this holds in
every current code path. I'll still defensively check
`g_szResumeFilePath[0] != '\0'` inside `FileReload` itself before using it as a
path, rather than trusting the invariant blindly — cheap insurance against some
future code path setting the flag without the path.

## Does the confirmation prompt correctly avoid nagging when there's nothing to lose?

Only shown when `g_bModified` is true. For a resumed document, `g_bModified` is
forced `TRUE` at recovery time and only goes back to `FALSE` via an explicit
save — which also clears `g_bIsResumedFile` in the process (see
`FinalizeSuccessfulSave`). So by the time a resumed document could have
`g_bModified == FALSE`, it's already stopped being "resumed" — there's no state
where `g_bIsResumedFile` is true and `g_bModified` is false. Good: the
resumed-specific confirmation wording and the "only prompt if modified" rule
never conflict.

## Bookmarks: why save them before reloading

`LoadTextFile` calls `LoadBookmarksForCurrentFile()`, which unconditionally does
`ClearBookmarks()` then re-reads the `[Bookmarks.XXXXXXXX]` INI section for the
current file. If the user toggled a bookmark earlier in this session and it was
never flushed to the INI (that only happens today on app exit, or right after an
explicit successful save), a reload would silently revert to the last-persisted
bookmark set, discarding the toggle. Calling `SaveBookmarksForCurrentFile()`
right before the reload avoids this — it writes the in-memory bookmark set to
the INI first, so the subsequent `LoadBookmarksForCurrentFile()` (inside
`LoadTextFile`) reads back exactly what was there a moment ago.

Note: this same gap exists in `FileNew()` and `FileOpen()` today (neither saves
bookmarks before switching away) — that's a pre-existing latent inconsistency,
not something introduced by this feature. Not fixing it here; flagged separately
in case it's worth a follow-up.

For an untitled document (resumed or not), `g_szFileName` is empty, so
`SaveBookmarksForCurrentFile()` (and `LoadBookmarksForCurrentFile()`) both
no-op immediately — bookmarks simply aren't persisted for untitled documents at
all. That's existing, unrelated behavior; nothing new to handle here.

## File extension / template context after a resumed reload

`LoadTextFile` calls `UpdateFileExtension(pszFileName)` using whichever path was
passed in. For the resumed branch, that's `g_szResumeFilePath`
(e.g. `notes_resume.md`) — and since `GenerateResumeFileName` already preserves
the original extension when building the resume filename, the extension comes
out correct by construction. Still, to avoid depending on that naming
convention holding forever, after restoring the saved globals I'll call
`UpdateFileExtension(g_szFileName)` once more explicitly, so the extension is
always derived from the authoritative original name rather than incidentally
matching. Cheap (it's just a string split plus a template-menu rebuild, which
already happens once inside `LoadTextFile` anyway).

## Status bar flash — why extract a helper instead of copy-pasting

The existing `[Autosaved]` flash (`DoAutosave`) does three things inline: save
current status text, set new text, arm a 1-second timer that restores the saved
text. The new `[Reloaded]` flash needs the exact same three steps. Rather than
duplicating that block, pulling it into `FlashStatusBarMessage(text, durationMs)`
means both call sites become one line each, and — since I'm touching this code
anyway — I can fix the fact that `[Autosaved]` is currently a hardcoded English
literal (`L"[Autosaved]"`) with no Czech translation, by loading both flash
texts from string resources instead. The shared `IDT_AUTOSAVE_FLASH` timer and
its `WM_TIMER` restore-handler don't need to change at all — they already just
restore whatever text was saved, regardless of which flash triggered it.

Small risk noted, accepted as negligible: if a reload happens to fire within the
same 1-second window as an in-flight autosave flash, whichever one starts second
will overwrite `g_szAutosaveFlashPrevStatus` with the *first* flash's text
(not the true original status), so the final restore could show slightly stale
text for one cycle. This is a pre-existing limitation of the single shared
buffer, purely cosmetic, and self-corrects on the next real status update. Not
worth a redesign for a one-second cosmetic edge case.

## What happens on a failed reload (file deleted, permission denied, etc.)

`LoadTextFile` returns `FALSE` before it ever calls `SetWindowText` — every
failure path (`CreateFile`, `GetFileSize`, `ReadFile`, memory allocation, UTF-8
conversion) returns early, having shown its own error via `ShowError`, without
touching the RichEdit control's content at all. So if reload fails, the
in-memory document is completely untouched — no data loss, no partial state.
`FileReload` just needs to skip the caret-restore/flash steps when
`LoadTextFile` returns `FALSE`, which the `if (LoadTextFile(...))` guard already
handles by construction.

## Testing limitations

This is Windows GUI code cross-compiled with `x86_64-w64-mingw32-g++` from
Linux. I can only verify a clean, warning-free `make` build in this
environment — there is no way to actually launch the `.exe` and click through
the menu, type Ctrl+R, or watch the caret land in the right place. The logic
above is as far as static reasoning can go; real verification of caret
placement and the resumed-file flow would need a human on Windows.

## Final sanity check on IDs before writing code

- `ID_FILE_RELOAD = 1010` — next free sequential ID after `ID_FILE_OPENRESUME`
  (1009); confirmed no other `1010` definition exists.
- String IDs `2198`–`2201` — confirmed free (last used ID in that block is
  `2197`, from the previous session's resume-file work).
- `BUILTIN_COUNT` goes from 25 to 26 in `BuildAcceleratorTable` — must count the
  actual accelerator lines after editing to make sure this stays accurate,
  since a mismatch here doesn't cause a compile error, just a wrong-sized
  `malloc` (either wasted space if too high, or an out-of-bounds write into the
  accelerator array if too low — this one is worth double-checking after the
  edit, not just trusting arithmetic).

## Corrections after code review

A code-review pass against the actual source (not just this document) found
three real bugs in the design above, plus a handful of smaller corrections.
Each is explained the same way as the rest of this document: what was wrong,
why, and the fix. These supersede the corresponding sections above.

### Bug 1: caret would snap to line start, not the original column

**What was wrong:** the original plan captured context and matched/relocated
using the *exact caret position* (`crBefore.cpMin`). But
`GetLineContextFromCharPos(charPos, ...)` doesn't extract text starting at
`charPos` — it first resolves `charPos` to its containing line via
`EM_EXLINEFROMCHAR` + `EM_LINEINDEX`, then extracts text starting at that
**line's start**. `ToggleBookmark` only ever stores a line-start position for
exactly this reason — the whole context-matching mechanism is built around
line-start anchors, not arbitrary columns.

Using the exact caret column as both the capture point and the match/restore
point means: unless the caret happened to already be sitting at column 0, the
"fast path" exact match (`BookmarkContextMatchesAt`) would almost always miss
(the text starting exactly at the caret's mid-line column won't equal a
line-start-anchored context snippet), forcing the slower search paths — and
even when a match is found, the returned position is a line start, so the
caret would jump to the beginning of the matched line, silently discarding the
original column every single time. This would have made Reload feel broken in
exactly the "keep reading, come back to my exact spot" scenario that motivated
the whole feature.

**Fix:** capture the line start and the column offset within that line
separately, relocate using the line start, then re-add the (clamped) column
offset:

```cpp
CHARRANGE crBefore   = RE_GetSel(g_hWndEdit);
LONG oldLineIndex     = (LONG)SendMessage(g_hWndEdit, EM_EXLINEFROMCHAR, 0, crBefore.cpMin);
LONG oldLineStart     = (LONG)SendMessage(g_hWndEdit, EM_LINEINDEX, oldLineIndex, 0);
LONG oldColumn        = crBefore.cpMin - oldLineStart;
WCHAR szContext[BOOKMARK_CONTEXT_LEN];
GetLineContextFromCharPos(oldLineStart, szContext, BOOKMARK_CONTEXT_LEN);

// ... after a successful LoadTextFile ...

LONG newLineStart = oldLineStart;
if (!BookmarkContextMatchesAt(oldLineStart, szContext)) {
    LONG found = FindContextNearPos(oldLineStart, szContext, 8192);
    if (found < 0) found = FindContextInDocument(szContext);
    if (found >= 0) {
        newLineStart = found;
    } else {
        int newLen = GetWindowTextLength(g_hWndEdit);
        if (newLineStart > newLen) newLineStart = newLen;
    }
}
LONG newLineLength = (LONG)SendMessage(g_hWndEdit, EM_LINELENGTH, newLineStart, 0);
LONG finalPos = newLineStart + min(oldColumn, newLineLength);
RE_SetSel(g_hWndEdit, finalPos, finalPos);
SendMessage(g_hWndEdit, EM_SCROLLCARET, 0, 0);
SetFocus(g_hWndEdit);
```

The `min(oldColumn, newLineLength)` clamp handles the case where the matched
line is now shorter than the original column (e.g. the line was edited to be
shorter elsewhere) — falls back to end-of-line rather than overshooting into
the next line.

### Bug 2: resumed-file reload silently clears the original file's bookmarks

**What was wrong:** in the resumed branch, `LoadTextFile(g_szResumeFilePath, FALSE)`
sets `g_szFileName` to the *temp* path internally, **before** it calls
`LoadBookmarksForCurrentFile()`. That function unconditionally clears the
in-memory bookmark array and then looks up the INI section keyed by whatever
`g_szFileName` currently is — at that moment, the temp resume path, which
essentially never has a matching `[Bookmarks.XXXXXXXX]` section (bookmarks are
saved keyed by the *original* path). Result: the bookmark array gets wiped and
nothing meaningful loads back. Restoring `g_szFileName` to the original path
afterward does not undo this — it doesn't retroactively re-trigger a bookmark
load.

The earlier `SaveBookmarksForCurrentFile()` call (done to avoid losing
in-session bookmark toggles) actually still works correctly — it runs *before*
`LoadTextFile`, while `g_szFileName` still holds the original path, so the
INI ends up with the right data. The bug is that nothing re-reads it back
afterward.

**Fix:** after restoring `g_szFileName`/`g_szFileTitle`/etc. in the resumed
branch, explicitly call `LoadBookmarksForCurrentFile()` again, now that
`g_szFileName` is correct:

```cpp
if (bWasResumed) {
    wcscpy_s(g_szFileName, EXTENDED_PATH_MAX, szSavedFileName);
    wcscpy_s(g_szFileTitle, MAX_PATH, szSavedFileTitle);
    wcscpy_s(g_szResumeFilePath, EXTENDED_PATH_MAX, szSavedResumeFilePath);
    wcscpy_s(g_szOriginalFilePath, EXTENDED_PATH_MAX, szSavedOriginalFilePath);
    g_bIsResumedFile = TRUE;
    g_bModified = TRUE;
    LoadBookmarksForCurrentFile();   // re-read now that g_szFileName is correct
    UpdateTitle();
    UpdateStatusBar();
}
```

### Bug 3: "resumed implies modified" is false, so the confirmation could be silently skipped

**What was wrong:** the plan assumed `g_bIsResumedFile == TRUE` always implies
`g_bModified == TRUE`, and used that to justify "only prompt when modified"
as sufficient for both cases. Checked against `DoAutosave` (`main.cpp:12653-12662`):
autosave is allowed to run on a resumed document that has a real filename
(`g_szFileName` non-empty, `g_bModified` true at that point) and, on success,
sets `g_bModified = FALSE` while **deliberately leaving `g_bIsResumedFile`
untouched** (it passes `bClearResumeState=FALSE` specifically to preserve
resume state). So a resumed document can absolutely end up with
`g_bModified == FALSE` while still showing `[Resumed]` in the title.

This matters because autosave writes to `g_szFileName` (the original
location), not to `g_szResumeFilePath` (the temp file) — so after such an
autosave, the temp file can be *stale* relative to what's already safely on
disk. If Reload silently skipped the confirmation in this state (because
"not modified"), it would revert the document to older content with zero
warning.

Separately, also confirmed: a cancelled Windows shutdown
(`WM_ENDSESSION` with `wParam == FALSE`, `main.cpp:4315-4322`) deletes the temp
file and clears `g_szResumeFilePath` but **never clears `g_bIsResumedFile`**.
This is a pre-existing latent bug in the shutdown-handling code, unrelated to
Reload, and out of scope to fix here — but it means `g_bIsResumedFile` can be
`TRUE` with an *empty* `g_szResumeFilePath` in a real (if rare) reachable
state. Reload must not assume the invariant "resumed implies a valid path"
either.

**Fix (two parts):**
1. Prompt whenever `g_bModified || g_bIsResumedFile` — not just `g_bModified`.
   For the ordinary case this is unchanged (identical to before). For a
   resumed document it now always warns, regardless of whether autosave
   already cleared the modified flag.
2. Before using `g_szResumeFilePath` as the reload target, check
   `g_szResumeFilePath[0] != L'\0'`. If empty despite `g_bIsResumedFile` being
   true (the cancelled-shutdown scenario above), show a brief informational
   message instead of attempting to load an empty path, e.g. "No recovery file
   is available to reload; use File > Open to load the file directly." No
   attempt is made to fix the underlying shutdown-handling gap as part of this
   feature.

### Smaller corrections

- **`UpdateFileExtension(g_szFileName)` after restore must be conditional.**
  The "Open Resume File" submenu handler intentionally leaves `g_szFileName`
  empty after loading an arbitrary resume file (treating it like an untitled
  document). If my restore step in the resumed branch calls
  `UpdateFileExtension(g_szFileName)` unconditionally, and `g_szFileName` is
  `""` in that case, it regresses the extension to `"txt"` — overwriting the
  correct extension that `LoadTextFile` had already derived from the resume
  file's own name (e.g. `"md"` from `notes_resume.md`) moments earlier. Fix:
  only call the extra `UpdateFileExtension` when the restored `g_szFileName`
  is non-empty; otherwise leave whatever `LoadTextFile` already set alone.

- **`wcscpy_s` does not "truncate" on overflow — it clears the destination.**
  The original document described `FinalizeSuccessfulSave`'s
  `wcscpy_s(g_szFileName, MAX_PATH, pszFileName)` as a truncation risk for
  paths longer than 260 characters. Checked: `wcscpy_s` with a source that
  doesn't fit the given size does **not** copy a truncated prefix — by secure
  CRT semantics it sets the destination to an empty string and invokes the
  invalid-parameter handler. This is actually worse than truncation (total
  loss of the field, not a shortened-but-present value), though still not a
  buffer overflow. This existing behavior in `FinalizeSuccessfulSave` and
  `WriteResumeFileContent` (which uses unbounded `wcscpy` from an
  `EXTENDED_PATH_MAX` source into `MAX_PATH`-sized locals — a genuine overflow
  risk, not just truncation) is out of scope to fix here, but confirms: my new
  code must always size local copies to `EXTENDED_PATH_MAX` to match the real
  declared capacity of `g_szFileName`/`g_szResumeFilePath`/
  `g_szOriginalFilePath`, never assume a smaller bound is "just a truncation."

- **Flash-buffer helper should not stomp its own saved text on a second flash.**
  As designed, `FlashStatusBarMessage` always calls `SB_GETTEXT` to capture
  "previous" text before showing the new one. If a second flash fires while
  the first flash's 1-second timer is still pending, `SB_GETTEXT` would
  capture the *currently flashed* text (e.g. `[Autosaved]`) rather than the
  true original status, so the eventual restore shows the wrong thing until
  the next real status update. Fix: track whether a flash is already active
  (a small `BOOL g_bStatusFlashActive` flag) and only capture "previous" text
  when not already mid-flash; otherwise just replace the visible text and
  reset the timer, keeping the originally-saved text intact.

- **`PromptSaveChanges()` is not reused for Reload — confirmed intentional.**
  A review pass raised this as a concern; checked against the actual
  pseudocode in this document, which always uses a dedicated `MessageBox`, not
  `PromptSaveChanges()`. Reusing it would have been wrong (its `IDNO` path
  deletes the resume file and clears resume state as a side effect, which is
  not appropriate for a Reload that isn't closing the document) — but that
  was never the plan. Noted here explicitly so it isn't second-guessed again
  later.

- **Ctrl+R routing through `IsDialogMessage`/`TranslateAccelerator` — confirmed
  no special handling needed.** `IsDialogMessage(g_hDlgFind, &msg)` is only
  consulted for the Find dialog specifically; the Output Pane is a plain child
  window of the main frame, not a separate dialog, so accelerators reach it
  normally via `TranslateAccelerator`. Ctrl+R therefore behaves exactly like
  every other File-menu accelerator (Ctrl+N/O/S/L): works from anywhere in the
  main window including the Output Pane, doesn't fire while the Find dialog
  has focus. This is consistent with existing behavior, not a special case to
  design around.

- **Pre-existing, out-of-scope issues surfaced by the review, not addressed by
  this feature:** `LoadTextFile` accepts a short `ReadFile` (doesn't verify
  `dwBytesRead == dwFileSize`) and doesn't check `SetWindowText`'s return
  value before committing state — both affect every caller (Open, resume-load,
  and now Reload equally), not something introduced here. Flagged for a
  possible separate follow-up, not fixed as part of this change.

