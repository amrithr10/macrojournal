# Micro Journal Rev 8 — Text Selection, Clipboard & Navigation Mod

Custom firmware for the Micro Journal Rev 8 (ESP32-S3 writerdeck by unkyulee)
adding text selection with inverse-video highlight, copy/cut/paste, and
word/line/document navigation to the built-in editor.

## Files

| File | Purpose |
|---|---|
| `firmware_rev_8.bin` | Merged flash image (bootloader + partitions + app). Flash at address **0x0**. |
| `rev8-text-selection-clipboard.patch` | Full source diff — the source of truth for the mod. |
| `keyboard.json` | Custom key layout (backtick next to A, extra Shift, `"` on backslash key) with the new selection/modifier keys wired in. Copy to the device drive root via Drive Mode. Only valid with this firmware — remove it if reverting to stock. |

## Shortcuts

Ctrl and GUI are the two keys left of the spacebar; both work identically.

**Navigation** — Ctrl+←/→: word · Home/End: line start/end (also with Ctrl,
or Ctrl+Shift) · Ctrl+PgUp/PgDn: start/end of the whole document (crosses
the 8 KB window, saves first).

**Selection** — Shift+arrows: character/line · Ctrl+Shift+←/→: word ·
Ctrl+Shift+↑/↓: line · Shift+Home/End: to line start/end · Ctrl+A: all.

**Clipboard** — Ctrl+C / X / V: copy / cut / paste (2 KB clipboard,
UTF-8 safe). Ctrl+Backspace deletes the word left of the cursor.

**Formatting (markdown-styled)** — files stay plain text; the Rev 8 screen
styles markdown as you type: `**bold**` renders bold (double-strike),
`*italic*` renders underlined, and `> quote` paragraphs get a bar in the
left margin. Ctrl+B / Ctrl+I wrap the current selection in markers (or
insert an empty pair to type into). Markers stay visible; emphasis resets
at each newline.

**Undo / redo** — Ctrl+Z undo, Ctrl+Y redo. Bursts of typing collapse into
one step (a ~0.6 s pause marks the boundary); up to 10 steps are kept. The
history is per-file and per-window — it resets when you switch files or
page across the 8 KB window boundary.

**File titles** — the home screen's file list shows each file's first
line as its title (markdown markers stripped, standard font, replaces the
word count), with `>` marking the currently open file. Titles refresh
every time the menu opens; protected files show `LOCKED`.

**File traverse mode** — `[F]` on the home screen highlights a file in
the list; arrows move the highlight (wrapping), Enter opens it, `D` jumps
to its clear-file confirmation, `P` to its password flow (a protected
file asks for its password first). ESC or `F` exits; digits still open
directly.

**Per-file passwords (encryption)** — `[P] PASSWORD` on the main menu
sets a password on the current file: the file is re-written as AES-256-CTR
ciphertext (PBKDF2-derived key, per-file salt) — plaintext never touches
the disk, so Drive Mode and sync only ever see unreadable bytes. Opening a
protected file (menu, Fn+number, or boot) prompts for the password; wrong
entries can be retried, ESC backs out to the menu. `[P]` on a protected,
open file offers to remove the password (decrypts in place). The key for
the open file stays cached until you switch files or power off. **A
forgotten password means the file is permanently unrecoverable** — there
is no reset. Known tradeoffs: word counts for large (>8 KB) protected
files only cover the loaded window, and synced snapshots of the same
protected file share a keystream (remove + re-add the password to re-key
if that matters to you).

**Dark mode** — `[I] DARK MODE` on the main menu toggles hardware panel
inversion (white text on black). Persisted in config, survives reboots;
everything (bold, underline, selection, quote bars) renders inverted
automatically.

**To-do scratchpad** — Ctrl+Shift+N from the editor or the menu opens a
plain-text scratchpad (status bar shows `TODO`); ESC saves it and returns
to the file you were in. The scratchpad is hidden file slot 10 — it lives
on the drive as `10.txt` (visible in Drive Mode, included in sync), and
Fn+0–9 can't reach it. Switching files with Fn+number while in the
scratchpad leaves it like any other file.

**Grammana Z (LLM assistant)** — `[G] GRAMMANA Z` on the home menu (or
Ctrl+Shift+C in the editor) opens the assistant screen (hidden file slot
11, status bar `GZ`). Type a question, press **Shift+Enter** to send;
the answer is appended below a `---` separator (typing stays live during
the request — it runs on the second core). The whole file is sent as
context, with `---` lines separating user/assistant turns, so follow-ups
just work; select-all + delete starts fresh. ESC returns to your writing.

The **provider is configurable** in `claude.json` — set `provider` to
`anthropic`, `openai`, or `gemini`, or just fill in one of `api_key`
(Anthropic), `openai_api_key`, or `gemini_api_key` and it auto-picks
whichever is present. `model` overrides the per-provider default
(`claude-opus-4-8` / `gpt-4o-mini` / `gemini-2.0-flash`). The same key
powers grammar check and rewrite. Auth and request shape are handled per
provider; a bad key or blocked request shows in the status bar.

**Grammar check & rewrite** — Ctrl+G reviews the current file (or just
the highlighted selection, if there is one) and shows a numbered issue
report in a `GRAMMAR` overlay (hidden slot 12): each issue quotes the
phrase, says what's wrong, and gives the fix. Ctrl+Shift+G instead returns
the text rewritten with corrected grammar — voice and wording preserved —
as copyable text in the same overlay. Either way your original is never
modified; ESC returns to it, and each run replaces the previous result.
Prompts can be overridden with `grammar_system` / `rewrite_system` in
`claude.json`. Uses the same WiFi/API setup as Ask Claude; for files
larger than 8 KB the loaded window is reviewed.

Requires: `wifi.json` (the device's normal WiFi setup) and `claude.json`
in the drive root (see `claude.json.example` for the multi-provider
template). `claude.json` holds a real API key
and is deliberately **not** in this repo (see `.gitignore`). Errors (no
WiFi, bad key, refusal) show in the status bar and clear on the next
keypress.

Typing replaces the selection; Backspace/DEL delete it; plain movement
drops it.

## Rebuilding

```sh
git clone https://github.com/unkyulee/micro-journal
cd micro-journal
git checkout 7b17cccef7a71af76b0165ece92a2f6c0c62ae26   # patch base (2026-07-12)
git apply /path/to/rev8-text-selection-clipboard.patch
cd micro-journal-rev-4-esp32                             # Rev 8 shares this tree
pio run -e rev_8
```

Then merge the flashable image (app-only `firmware.bin` at 0x0 boot-loops —
always merge):

```sh
cd .pio/build/rev_8
esptool.py --chip esp32s3 merge_bin -o firmware_rev_8.bin \
  --flash_mode dio --flash_freq 80m --flash_size 16MB \
  0x0 bootloader.bin 0x8000 partitions.bin \
  0xe000 <arduino-esp32>/tools/partitions/boot_app0.bin \
  0x10000 firmware.bin
```

## Flashing

1. Upper USB-C port → PC, https://www.espboards.dev/tools/program/
2. Connect → upload `firmware_rev_8.bin` → address **0x0** → Program (~1 min).
3. Unlike official releases this image does not wipe the data partition;
   journal files and `keyboard.json` survive.

## Implementation notes

- Editor engine changes are shared by all Micro Journal revisions but gated
  behind `Editor::selectionSupported`, which only the Rev 8 (RLCD) display
  enables — other devices are unaffected (rev_4_68 and rev_6 build-verified).
- New key codes are 2100–2119 in `src/service/Editor/Editor.h`.
- A `/keyboard.json` on the device replaces the compiled key layers entirely;
  it must use the new names (`CTRL`, `GUI`, `SELECT_LEFT`, `WORD_RIGHT`,
  `DOC_END`, …) or the features silently go dead.
