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

**Undo / redo** — Ctrl+Z undo, Ctrl+Y redo. Bursts of typing collapse into
one step (a ~0.6 s pause marks the boundary); up to 10 steps are kept. The
history is per-file and per-window — it resets when you switch files or
page across the 8 KB window boundary.

**To-do scratchpad** — Ctrl+Shift+N from the editor or the menu opens a
plain-text scratchpad (status bar shows `TODO`); ESC saves it and returns
to the file you were in. The scratchpad is hidden file slot 10 — it lives
on the drive as `10.txt` (visible in Drive Mode, included in sync), and
Fn+0–9 can't reach it. Switching files with Fn+number while in the
scratchpad leaves it like any other file.

**Ask Claude** — Ctrl+Shift+C opens the ask screen (hidden file slot 11,
status bar shows `ASK`). Type a question, press **Shift+Enter** to send it
to the Anthropic API over WiFi; the status bar shows `ASKING...` and the
answer is appended below a `---` separator (typing stays live during the
request — it runs on the second core). Everything in the file is sent as
conversation context, with `---` lines separating user/assistant turns —
so follow-ups just work; select-all + delete starts a fresh conversation.
ESC returns to your writing.

Requires: `wifi.json` (the device's normal WiFi setup) and `claude.json`
in the drive root — `{"api_key": "sk-ant-...", "model": "claude-opus-4-8",
"max_tokens": 1024, "system": "..."}`. `claude.json` holds a real API key
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
