# RighterDeck — a Micro Journal Rev 8 firmware mod

Custom firmware for the [Micro Journal Rev 8](https://github.com/unkyulee/micro-journal)
(the ESP32-S3 writerdeck by Un Kyu Lee) that turns the stock editor into a
small but complete writing environment: text selection and clipboard, word
and document navigation, undo/redo, live markdown styling, per-file
AES-256 encryption, file titles and quick file navigation, a to-do
scratchpad, dark mode, and an on-device LLM assistant ("Grammana Z") with
grammar checking and rewriting — all driven from the keyboard, all
rendered for the 400x300 reflective LCD.

Everything runs on the stock hardware. Flashing is a one-file,
browser-based step and does not erase your journal files.

## What's in this repository

| File | Purpose |
|---|---|
| `firmware_rev_8.bin` | Ready-to-flash merged image (bootloader + partitions + app). Flash at address **0x0**. |
| `rev8-text-selection-clipboard.patch` | The full source diff against upstream — the source of truth for this mod. |
| `keyboard.json` | A custom key layout with the mod's key names wired in (see [keyboard.json](#keyboardjson)). |
| `grammana_z.json.example` | Template for the LLM assistant configuration. Copy to the device as `grammana_z.json` with your API key. |

## Quick start

1. Connect the **upper** USB-C port to a computer and open
   <https://www.espboards.dev/tools/program/> in Chrome or Edge.
2. Connect → choose `firmware_rev_8.bin` → set address **0x0** → Program
   (about a minute). Unlike official releases, this image does **not**
   wipe the data partition — your files survive.
3. Optional, for the LLM features: enter Drive Mode (`[U]` on the menu),
   copy `grammana_z.json` (from `grammana_z.json.example`, with a real API key)
   into the drive root, and set up WiFi on the device (`[W]`).

## The menu

The menu (toolbar title: **RighterDeck** — rename it with `[N]`) is two
panels under matching inverted headers, split by a divider. Arrow keys
navigate the highlighted pane and Enter activates the highlight;
letter/number hotkeys still work directly at any time. ESC from any
sub-screen (WiFi, Language, Sync, …) returns to this main menu.

- **MENU** — the commands, alphabetical: BLE KEYS, DARK, DEVICE NAME,
  DRIVE, FILES, LANGUAGE, SYNC (shown once sync is configured), WIFI,
  and BACK. One is always highlighted; arrows move it and Enter runs it,
  or just press the bracketed key.
- **CHOOSE A FILE** — one scrolling list: `[S]` SCRATCHPAD, `[G]`
  GRAMMANA Z, and `[/]` LAST NOTE (jumps to the note you edited most
  recently) as the first three entries, a separator line, then the ten
  notes `[0]`–`[9]`. Press the bracketed key to open any of them
  directly (S / G / / / 0–9), or navigate with `F`. A down-chevron
  shows when more sits below; `>` marks the open file/tool; protected
  notes show `LOCKED`. Titles (each note's first line, markers
  stripped) refresh every time the menu opens.

**File navigation (`F`)** — press `F` and a row highlights in inverse
video; the left pane switches from the command list to the actions for
that row. Arrows move the highlight through all thirteen entries
(Scratchpad, Grammana Z, Last Note, a separator, then notes 0–9),
wrapping at the ends, and the list scrolls to keep the highlight in
view. **Enter** opens the highlighted item. For notes only, **D** clears it (password-verified for locked notes, straight into the
usual confirmation, then back to the menu) and **P** opens its password
flow — the three tool entries show just OPEN. The command keys are inactive while
traversing; `F` or ESC exits. The highlight is sticky: open something
and come back, and it's still where you left it.

**Rename the device (`N`)** — the toolbar title (default **RighterDeck**)
is yours to change: press `[N] DEVICE NAME` on the menu, type a new name into the
header bar (up to 12 characters), and press Enter — it saves and drops
you straight back into the editor (ESC cancels). The name shows at the
top of the menu from then on and persists across reboots (stored in
`config.json` as `"title"`, so you can also set it from a computer in
Drive Mode).

## Editor shortcuts

Ctrl and GUI are the two keys left of the spacebar; they are
interchangeable in every shortcut below.

| Action | Keys |
|---|---|
| Select character / line | Shift + arrows |
| Select word | Ctrl+Shift + ←/→ |
| Select line | Ctrl+Shift + ↑/↓ |
| Select to line start/end | Shift + Home/End |
| Select all | Ctrl + A |
| Copy / cut / paste | Ctrl + C / X / V |
| Move by word | Ctrl + ←/→ |
| Line start / end | Home / End (with or without Ctrl/Shift) |
| Document start / end | Ctrl + PgUp / PgDn |
| Delete word left | Ctrl + Backspace |
| Undo / redo | Ctrl + Z / Y |
| Bold / italic markers | Ctrl + B / I |
| Scratchpad | Ctrl+Shift + S |
| Grammana Z assistant | Ctrl+Shift + C |
| Grammar check | Ctrl + G |
| Grammar rewrite | Ctrl+Shift + G |

Selections render in inverse video. Typing replaces the selection;
Backspace/DEL deletes it; plain movement drops it. The clipboard holds
2 KB and is UTF-8 safe.

**Undo/redo** keeps up to 10 steps; bursts of typing collapse into one
step (a ~0.6 s pause marks the boundary). History is per-file and
per-window — it resets when you switch files or page across the 8 KB
window boundary.

**Markdown styling** — files stay plain text; the screen styles as you
type: `**bold**` renders double-struck, `*italic*` renders underlined,
and `> quote` paragraphs get a bar in the left margin and continue on
Enter until an empty line ends them. Ctrl+B / Ctrl+I wrap the current
selection in markers (or insert an empty pair to type into).

## Features in detail

### Per-file passwords (real encryption)

Password management lives in file navigation: press `F`, highlight a
note, and `P` sets or removes its password (locked notes verify the old
password first) without opening it. The file is re-written as
**AES-256-CTR ciphertext** — plaintext never touches the disk again, so
Drive Mode and sync only ever see unreadable bytes. Opening a protected
file prompts for the password; wrong entries can be retried; ESC backs
out. **Leaving a file to the menu re-locks it.** `P` on a protected note
in file navigation asks for the password and removes the protection
(decrypts in place).

> **A forgotten password means the file is permanently unrecoverable.**
> There is no reset, no recovery, no backdoor. Test the flow on a
> throwaway file first.

Design details for the curious: PBKDF2-HMAC-SHA256 key derivation
(per-file random salt from the hardware RNG), a 56-byte header with a
key verifier (so wrong passwords are rejected without touching
content), and a position-keyed CTR keystream so the editor's windowed
reads and in-place saves work directly on ciphertext. Keys live only in
RAM, one file at a time, and are wiped on lock, file switch, and menu
exit.

Known tradeoffs: word counts for large (>8 KB) protected files only
cover the loaded window; repeated synced snapshots of the same
protected file share a keystream (remove and re-add the password to
re-key if that matters to you); and clearing a file leaves the usual
`*_backup.txt` on the drive — for a protected file that backup is still
ciphertext.

### Grammana Z (LLM assistant)

The GRAMMANA Z row in the file list (or `G` on the home screen, or
Ctrl+Shift+C in the editor) opens the assistant
(status bar `GZ`). Type a question, press **Shift+Enter** to send — the
request runs on the second core, so typing stays live. The answer is
appended under a `---` separator; the whole file is the conversation
(`---` separates turns), so follow-ups just work. Select-all + delete
starts fresh; ESC returns to your writing.

Works with **Anthropic, OpenAI, or Google Gemini** — see
[grammana_z.json](#grammana_zjson).

### Grammar check and rewrite

**Ctrl+G** reviews the current file — or only the highlighted selection
— and opens a numbered issue report (each issue quotes the phrase, says
what's wrong, gives the fix). **Ctrl+Shift+G** instead returns the text
rewritten with corrected grammar, voice preserved, as copyable text.
Either way the original is never modified; ESC returns to it, and each
run replaces the previous report. For files over 8 KB the loaded window
is reviewed.

### Scratchpad

Ctrl+Shift+S from anywhere (or the SCRATCHPAD row in the file list)
opens a plain-text scratchpad (status bar `SCRATCHPAD`); ESC saves and
returns to the file you were in. It lives in slot 10 (`10.txt` on the
drive, included in sync).

### Dark mode

`[I]` toggles hardware panel inversion — white text on black,
everything (selection, markdown styling, menu) included. Persists
across reboots.

## Configuration files

All three live in the drive root (Drive Mode: `[U]` on the menu).

### grammana_z.json

Required only for Grammana Z and grammar check. Start from
`grammana_z.json.example`:

| Field | Meaning |
|---|---|
| `provider` | `anthropic`, `openai`, or `gemini`. Omit to auto-pick from whichever key is filled in. |
| `api_key` | Anthropic API key (`anthropic_api_key` also accepted). |
| `openai_api_key` / `gemini_api_key` | Keys for the other providers. |
| `model` | Optional override. Defaults: `claude-opus-4-8` / `gpt-4o-mini` / `gemini-2.0-flash`. |
| `max_tokens` | Response budget (default 1024). |
| `system` | Assistant system prompt (the default keeps answers short and plain-text). |
| `grammar_system` / `rewrite_system` | Optional prompt overrides for check/rewrite. |

Errors (no WiFi, bad key, refusal) show in the status bar and clear on
the next keypress.

### wifi.json

The device's normal WiFi setup — configure it on-device via `[W]`.
Needed for sync and the LLM features. The radio is only powered during
a request and switched off after.

### keyboard.json

A `/keyboard.json` on the drive replaces the compiled key layout
entirely. The one in this repo matches the author's preferences
(backtick next to A, extra Shift, `"` on the backslash key) **and
contains the mod's key names** (`CTRL`, `GUI`, `SELECT_LEFT`,
`WORD_RIGHT`, `DOC_END`, …). If you use your own layout, add those
names or the new features silently go dead. Remove the file if
reverting to stock firmware.

## Security notes (read before sharing files or keys)

- Your `grammana_z.json` holds a real API key — it stays on the device and
  is sent only as an auth header to the configured provider. It is
  **git-ignored here and has never been committed**; keep it that way
  if you fork this repo.
- All outbound HTTPS — LLM calls **and** Google Drive sync — verifies
  the server's certificate against a bundled root CA trust store
  (`src/service/Certs/Certs.h`), so a network attacker can't
  impersonate the endpoint to capture the key or your text. (The stock
  firmware skips this check; this mod adds it.) A wrong or man-in-the
  middle certificate makes the request fail rather than leak. Because
  certificate validation needs the real date, the device syncs its
  clock over NTP the first time WiFi comes up each power cycle — a few
  extra seconds on that first request.
- Grammana Z sends the **whole current file** (or selection, for
  grammar) to your chosen provider. Don't use it inside files you
  wouldn't share with that provider.
- File encryption protects the content of protected files at rest —
  including in Drive Mode, sync uploads, and backups — but file
  **titles show `LOCKED`**, sizes remain visible, and the other nine
  files are plain text.
- PBKDF2 runs 6,000 iterations (an ESP32-class budget). A long
  passphrase matters more than the iteration count.

## Rebuilding from source

```sh
git clone https://github.com/unkyulee/micro-journal
cd micro-journal
git checkout 7b17cccef7a71af76b0165ece92a2f6c0c62ae26   # patch base (2026-07-12)
git apply /path/to/rev8-text-selection-clipboard.patch
cd micro-journal-rev-4-esp32                             # Rev 8 shares this tree
pio run -e rev_8
```

Then merge the flashable image (an app-only `firmware.bin` at 0x0
boot-loops — always merge):

```sh
cd .pio/build/rev_8
esptool.py --chip esp32s3 merge_bin -o firmware_rev_8.bin \
  --flash_mode dio --flash_freq 80m --flash_size 16MB \
  0x0 bootloader.bin 0x8000 partitions.bin \
  0xe000 <arduino-esp32>/tools/partitions/boot_app0.bin \
  0x10000 firmware.bin
```

## Implementation notes

- Editor engine changes are shared by all Micro Journal revisions but
  gated behind `Editor::selectionSupported`, which only the Rev 8
  (RLCD) display enables — other devices are unaffected (`rev_4_68`
  and `rev_6` stay build-clean).
- New key codes are 2100–2129 in `src/service/Editor/Editor.h`; the
  encryption service is `src/service/Crypt/`, the LLM service
  `src/service/AskClaude/`, and the HTTPS root CA trust store
  `src/service/Certs/`.
- The menu and dialogs render in `profont22` — after hardware testing,
  the only font family that stays readable on this panel (condensed
  fonts pack letters together; thin-stroke fonts render ragged).
- The undo engine and the encryption offset math were verified with
  host-side AddressSanitizer harnesses built from the shipped code
  (windowed reads at odd offsets, fast and splice save paths, 300
  randomized edit cycles).

## Credits and license

Built on [Micro Journal](https://github.com/unkyulee/micro-journal) by
**Un Kyu Lee**, licensed **CC BY-NC 4.0** — this mod inherits that
license: share and adapt freely with attribution, no commercial use.
The patch touches only the `micro-journal-rev-4-esp32` firmware tree.
