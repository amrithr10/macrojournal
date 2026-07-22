# RighterDeck — a Micro Journal Rev 8 firmware mod

Custom firmware for the [Micro Journal Rev 8](https://github.com/unkyulee/micro-journal)
(the ESP32-S3 writerdeck by Un Kyu Lee) that turns the stock editor into a
small but complete, keyboard-driven writing environment.

**What it adds over the stock firmware**

- **Text selection & clipboard** — select by character, word, line, or the
  whole document; cut / copy / paste.
- **Word & document navigation** — jump by word, to line ends, and across
  the whole document.
- **Undo / redo** — up to 10 coalesced steps.
- **Live markdown styling** — bold, italic (underline), and quote bars
  render as you type; the file stays plain text.
- **Per-file encryption** — real AES-256 with a password; plaintext never
  hits the disk.
- **A redesigned home screen** — two panels, arrow-key navigation, file
  titles, a scrolling file list with quick-launch tools, and a device name
  you can set.
- **A scratchpad** — a always-there note, plus "send selection to
  scratchpad" while writing.
- **Grammana Z** — an on-device LLM assistant (Anthropic, OpenAI, or
  Gemini) with grammar checking and rewriting.
- **Dark mode**, **device rename**, and quality-of-life polish throughout.

Everything runs on stock hardware. Flashing is a one-file, browser-based
step and does **not** erase your journal files.

> **New here? Start with the illustrated [software guide](#software-guide)**
> for a screen-by-screen walkthrough, then use this README as the
> reference.

---

## Contents

- [What's in this repository](#whats-in-this-repository)
- [Requirements](#requirements)
- [Flashing the firmware](#flashing-the-firmware)
- [First-time setup](#first-time-setup)
- [The home menu](#the-home-menu)
- [Working with files](#working-with-files)
- [The editor](#the-editor)
- [Markdown styling](#markdown-styling)
- [Undo / redo](#undo--redo)
- [Scratchpad](#scratchpad)
- [Grammana Z (LLM assistant)](#grammana-z-llm-assistant)
- [Grammar check & rewrite](#grammar-check--rewrite)
- [Per-file passwords (encryption)](#per-file-passwords-encryption)
- [Dark mode](#dark-mode)
- [Device rename](#device-rename)
- [Other menu functions](#other-menu-functions)
- [Configuration files](#configuration-files)
- [Files on the drive](#files-on-the-drive)
- [Troubleshooting & FAQ](#troubleshooting--faq)
- [Security notes](#security-notes)
- [Known limitations](#known-limitations)
- [Rebuilding from source](#rebuilding-from-source)
- [Implementation notes](#implementation-notes)
- [Software guide](#software-guide)
- [Credits & license](#credits--license)

---

## What's in this repository

| File | Purpose |
|---|---|
| `firmware_rev_8.bin` | Ready-to-flash merged image (bootloader + partitions + app). Flash at address **0x0**. |
| `rev8-text-selection-clipboard.patch` | The full source diff against upstream — the source of truth for this mod. |
| `keyboard.json` | A custom key layout with the mod's key names wired in (see [keyboard.json](#keyboardjson)). Copy to the device drive root. |
| `grammana_z.json.example` | Template for the LLM assistant config. Copy to the device as `grammana_z.json` with your API key. |
| `GUIDE.html` | The illustrated software guide (open in any browser). |

`grammana_z.json` (with a real API key) is intentionally **not** in this
repo — it is git-ignored. Never commit it.

## Requirements

- A **Micro Journal Rev 8** (ESP32-S3 writerdeck, 400×300 reflective LCD).
  This firmware targets the `rev_8` build only.
- A computer with **Chrome or Edge** for the web flasher (Web Serial).
- A USB-C cable.
- Optional, for the LLM features: a WiFi network and an API key from
  Anthropic, OpenAI, or Google.

## Flashing the firmware

1. Plug the **upper** USB-C port into your computer and open the web
   flasher: <https://www.espboards.dev/tools/program/> (Chrome/Edge).
2. Click **Connect** and select the serial port.
3. Upload **`firmware_rev_8.bin`** and set the address to **`0x0`**.
4. Click **Program**. It takes about a minute.

Notes:

- Always flash **the merged image at `0x0`**. An app-only `firmware.bin`
  flashed at `0x0` will boot-loop.
- Unlike official releases, this image does **not** wipe the data
  partition — your journal files and `keyboard.json` survive an update.
- If it seems to hang for many minutes, see
  [Troubleshooting](#troubleshooting--faq).

## First-time setup

1. **Flash** the firmware (above) and let the device boot into the editor.
2. Copy `keyboard.json` from this repo onto the drive (see below) if you
   want the author's layout, or make sure your own layout includes the
   mod's key names — otherwise the new shortcuts do nothing.
3. To use the drive: press the menu key to open the menu, then `[U]` for
   **Drive Mode**. The device appears as a USB drive on your computer.
   Copy files to/from its root, then eject.
4. Optional — **WiFi**: menu → `[W]`, add a network (needed for sync and
   Grammana Z).
5. Optional — **Grammana Z**: copy `grammana_z.json` (from the example,
   with your API key) to the drive root. The `[G]` entry appears once the
   file is present.

---

## The home menu

Open the menu with the dedicated menu key. The screen is two panels under
matching headers, split by a divider, with the device title
(**RighterDeck** by default) centered in the toolbar.

**Navigation** — arrow keys move a highlight in the active pane and
**Enter** activates it; the bracketed letter/number hotkeys also work
directly at any time. **ESC** from any sub-screen (WiFi, Language, Sync,
…) returns to the main menu rather than dropping out to your note.

**Left pane — `MENU`** (device commands, ordered by their bracket key):

| Key | Command | What it does |
|---|---|---|
| `[I]` | DARK: ON/OFF | Toggle dark mode |
| `[L]` | LANGUAGE | Choose the keyboard layout |
| `[N]` | DEVICE NAME | Rename the device (the toolbar title) |
| `[S]` | SYNC | Upload files to Google Drive (only shown once configured) |
| `[T]` | BLE KEYS | Use the deck as a Bluetooth keyboard for another device |
| `[U]` | DRIVE | USB Drive Mode (access files from a computer) |
| `[W]` | WIFI | Configure WiFi networks |
| `[B]` | BACK | Return to the editor |

`[F]` (open the file list / start navigation) sits next to the **CHOOSE A
FILE** header rather than in this column.

**Right pane — `CHOOSE A FILE`** (one scrolling list):

- `[S]` **SCRATCHPAD**, `[G]` **GRAMMANA Z**, and `[/]` **LAST NOTE**
  (jumps to the note you edited most recently, with its number in
  brackets) are the first three entries, followed by a separator line.
- Then the ten notes `[0]`–`[9]`, each showing its first line as a title
  (markdown markers stripped) or `-` when empty. Protected notes show
  `LOCKED`. A `>` marks the currently open file/tool.
- Press a bracketed key to open any entry directly (`S` / `G` / `/` /
  `0`–`9`), or press `F` to navigate with the arrows.
- A **down-chevron** at the bottom means more notes are below — scroll to
  them. Titles refresh every time the menu opens.

## Working with files

You have **ten notes** (`0`–`9`) plus three built-in tools (Scratchpad,
Grammana Z, Last Note). Open any of them from the file list.

**File navigation (`F`)** — press `F` and a row highlights in inverse
video; the left pane switches from the command list to the **actions** for
the highlighted row:

| Key | Action |
|---|---|
| Arrows | Move the highlight (wraps at the ends; the list scrolls) |
| Enter | Open the highlighted item |
| `D` | Clear the highlighted **note** (asks to confirm; locked notes verify the password first) |
| `P` | Set / remove a **note's** password |
| `F` / ESC | Exit navigation |

`D` and `P` apply to real notes only — the three tools show just **OPEN**.
The highlight is **sticky**: open something, come back, and it's still
where you left it.

**Titles** — a note's title is its first line, with leading whitespace and
markdown markers (`>`, `#`, `*`) stripped, so a note that starts with
`# Chapter one` shows as `Chapter one`.

## The editor

The editor is the writing surface. The status bar along the bottom shows
the current file (or `SCRATCHPAD` / `GZ` / `GRAMMAR` for the tools), the
word count, the keyboard locale (if not US), and `SAVED` when your work is
on disk. Files save automatically.

**Ctrl and GUI** are the two keys left of the spacebar; they are
**interchangeable** in every shortcut below.

| Action | Keys |
|---|---|
| Select character / line | Shift + arrows |
| Select word | Ctrl+Shift + ←/→ |
| Select line | Ctrl+Shift + ↑/↓ |
| Select to line start / end | Shift + Home / End |
| Select all | Ctrl + A |
| Copy / cut / paste | Ctrl + C / X / V |
| Move by word | Ctrl + ←/→ |
| Line start / end | Home / End (also with Ctrl or Ctrl+Shift) |
| Document start / end | Ctrl + PgUp / PgDn |
| Delete word to the left | Ctrl + Backspace |
| Undo / redo | Ctrl + Z / Y |
| Bold / italic markers | Ctrl + B / I |
| Open / send-to scratchpad | Ctrl+Shift + S |
| Grammana Z assistant | Ctrl+Shift + C |
| Grammar check | Ctrl + G |
| Grammar rewrite | Ctrl+Shift + G |

Selections render in **inverse video**. Typing replaces the selection;
Backspace/DEL deletes it; plain cursor movement drops it. The clipboard
holds **2 KB** and is UTF-8 safe.

The editor works on an **8 KB window** of the file at a time, paging as you
move — so very long files scroll smoothly without needing the whole file
in memory.

## Markdown styling

Files stay plain text; the screen styles markdown live as you type:

- `**bold**` renders **double-struck** (heavier strokes).
- `*italic*` renders **underlined** (the panel can't slant glyphs).
- `> quote` paragraphs get a **bar in the left margin** and keep the quote
  going on each Enter until you press Enter on an empty line.

**Ctrl+B / Ctrl+I** wrap the current selection in the markers, or insert
an empty pair to type into. The markers stay visible; emphasis resets at
each newline.

## Undo / redo

**Ctrl+Z** undoes, **Ctrl+Y** redoes. Bursts of typing collapse into one
step (a ~0.6 s pause marks a boundary), and up to **10 steps** are kept.
History is per-file and per-window — it resets when you switch files or
page across the 8 KB window boundary.

## Scratchpad

**Ctrl+Shift+S** (or the `[S]` SCRATCHPAD row in the file list) opens a
plain-text scratchpad — a quick, always-there note (status bar
`SCRATCHPAD`). ESC saves it and returns you to the file you were in. It
lives in slot 10 (`10.txt` on the drive, included in sync).

**Send selection to scratchpad** — if you have text highlighted in a note,
**Ctrl+Shift+S** instead appends that selection to the scratchpad (on its
own line) *without leaving the note*. A quick way to stash a snippet while
writing: the note and your selection are untouched, and a `SENT TO SCRATCH`
flash by the word count confirms it for 3 seconds. (Sending from an
encrypted note writes that snippet to the scratchpad in plain text, since
the scratchpad itself isn't encrypted.)

## Grammana Z (LLM assistant)

The **GRAMMANA Z** row in the file list (or `G` on the home screen, or
**Ctrl+Shift+C** in the editor) opens the assistant (status bar `GZ`).

- Type a question and press **Shift+Enter** to send. The request runs on
  the second CPU core, so typing stays live while it thinks.
- The answer is appended below a `---` separator. The whole file *is* the
  conversation (with `---` lines separating turns), so follow-up questions
  keep their context automatically.
- **Select-all + delete** starts a fresh conversation; **ESC** returns to
  your writing.

**Provider** — works with **Anthropic, OpenAI, or Google Gemini**, chosen
in [`grammana_z.json`](#grammana_zjson). Requires WiFi and an API key.

## Grammar check & rewrite

- **Ctrl+G** reviews the current file — or only the **highlighted
  selection** — and opens a numbered issue report in a `GRAMMAR` overlay:
  each issue quotes the phrase, says what's wrong, and gives the fix.
- **Ctrl+Shift+G** instead returns the text **rewritten** with corrected
  grammar (voice and wording preserved), as copyable text.

Either way your **original is never modified**; ESC returns to it, and
each run replaces the previous report. For files over 8 KB the loaded
window is reviewed. Uses the same WiFi/API setup as Grammana Z; the prompts
can be overridden with `grammar_system` / `rewrite_system` in the config.

## Per-file passwords (encryption)

Password management lives in **file navigation**: press `F`, highlight a
note, and press **`P`** to set or remove its password — no need to open the
file (locked notes verify the old password first).

Setting a password re-writes the file as **AES-256-CTR ciphertext**, so
plaintext never touches the disk again — Drive Mode, sync, and backups
only ever see unreadable bytes. Opening a protected file prompts for the
password (wrong entries can be retried; ESC backs out). **Leaving a file
to the menu re-locks it** — you'll be asked again next time. Removing a
password decrypts the file in place.

> ⚠️ **A forgotten password means the file is permanently unrecoverable.**
> There is no reset, no recovery, no backdoor. Test the flow on a
> throwaway file first.

**Under the hood:** PBKDF2-HMAC-SHA256 key derivation with a per-file
random salt from the hardware RNG; a 56-byte header carrying a key
verifier (so wrong passwords are rejected without touching content); and a
position-keyed CTR keystream so the editor's windowed reads and in-place
saves work directly on ciphertext. Keys live only in RAM, one file at a
time, and are wiped on lock, file switch, and menu exit.

**Tradeoffs:** word counts for large (>8 KB) protected files only cover the
loaded window; repeated synced snapshots of the same protected file share
a keystream (remove and re-add the password to re-key if that matters);
and clearing a file leaves the usual `*_backup.txt` — for a protected file
that backup is still ciphertext.

## Dark mode

`[I]` on the menu toggles **hardware panel inversion** — white text on
black. Everything (selection, markdown styling, menu highlights) renders
inverted automatically. It persists across reboots.

## Device rename

The toolbar title (default **RighterDeck**) is yours to change: press
`[N] DEVICE NAME` on the menu, type a new name into the header bar (up to
12 characters), and press **Enter** — it saves and drops you straight back
into the editor (ESC cancels). The name shows at the top of the menu from
then on and persists across reboots (stored in `config.json` as `"title"`,
so you can also set it from a computer in Drive Mode).

## Other menu functions

These come from the stock firmware and behave as normal:

- **`[W]` WIFI** — add and edit WiFi networks (stored on the device's
  internal flash, not the drive). Needed for Sync and Grammana Z. The radio
  is only powered during a request and switched off afterward.
- **`[L]` LANGUAGE** — pick the keyboard layout (US, UK, DE, FR, and
  more). Saved to config.
- **`[T]` BLE KEYS** — the deck advertises itself as a **Bluetooth
  keyboard**; pair it from a computer or phone and what you type on the
  deck is sent to that device. (The battery level it reports is a fixed
  default, not a real reading — see [Known limitations](#known-limitations).)
- **`[U]` DRIVE** — **USB Drive Mode**. The device mounts as a USB mass
  storage drive so you can copy files and configs to/from its root. Exiting
  reboots the device to remount the filesystem cleanly.
- **`[S]` SYNC** — upload your files to Google Drive via a Google Apps
  Script endpoint (its URL goes in `config.json` under `sync.url`; there is
  no on-device setup for the URL). The row only appears once a URL is
  configured.

---

## Configuration files

Config files live in the **drive root** (reach it with Drive Mode, `[U]`).

### grammana_z.json

Required only for Grammana Z and grammar check. Start from
`grammana_z.json.example`:

| Field | Meaning |
|---|---|
| `provider` | `anthropic`, `openai`, or `gemini`. Omit to auto-pick from whichever key is filled in. |
| `api_key` | Anthropic API key (`anthropic_api_key` also accepted). |
| `openai_api_key` / `gemini_api_key` | Keys for the other providers. |
| `model` | Optional model override. Defaults: `claude-opus-4-8` / `gpt-4o-mini` / `gemini-2.0-flash`. |
| `max_tokens` | Response budget (default 1024). |
| `system` | Assistant system prompt (the default keeps answers short and plain-text). |
| `grammar_system` / `rewrite_system` | Optional prompt overrides for grammar check / rewrite. |

Errors (no WiFi, bad key, refusal) show in the status bar and clear on the
next keypress.

### wifi.json

The device's WiFi setup. Configure it **on-device** via `[W]` (it's saved
to internal flash for safety, not to the drive). Needed for Sync and the
LLM features.

### keyboard.json

A `/keyboard.json` on the drive **replaces the compiled key layout
entirely**. The one in this repo matches the author's preferences (backtick
next to A, an extra Shift, `"` on the backslash key) **and contains the
mod's key names** (`CTRL`, `GUI`, `SELECT_LEFT`, `WORD_RIGHT`, `DOC_END`,
…). If you use your own layout, you must add those names or the new
features silently go dead. Remove the file to revert to the compiled
layout.

### config.json

The device's own settings file (created automatically). Notable fields
this mod uses: `title` (device name), `darkmode`, `last_note`, and
`sync.url` (Google Drive endpoint). You normally change these through the
menu, but they're editable in Drive Mode too.

## Files on the drive

| File(s) | What it is |
|---|---|
| `0.txt` – `9.txt` | Your ten notes |
| `10.txt` | The scratchpad |
| `11.txt` | The Grammana Z conversation |
| `12.txt` | The last grammar-check / rewrite report |
| `*_backup.txt` | Automatic backup made when a file is cleared |
| `config.json` | Device settings |
| `wifi.json` | WiFi networks (usually on internal flash) |
| `grammana_z.json` | LLM config + your API key |
| `keyboard.json` | Key layout (optional) |

A protected note's `N.txt` is ciphertext on disk; so is its backup.

---

## Troubleshooting & FAQ

**Flashing seems stuck for many minutes.**
The web flasher can stall if the port is busy or the cable is charge-only.
Unplug, use a known-good data cable on the **upper** USB-C port, close any
serial monitor, and retry. A normal program takes ~1 minute.

**The device boot-loops after flashing.**
You likely flashed an app-only `firmware.bin` at `0x0`. Always flash the
**merged** `firmware_rev_8.bin` at `0x0`.

**The new shortcuts (selection, etc.) do nothing.**
Your `keyboard.json` on the drive is overriding the layout and lacks the
mod's key names. Use this repo's `keyboard.json`, or add the new names to
yours. (A `keyboard.json` replaces the compiled layout entirely.)

**Grammana Z says `NO WIFI` / `NO grammana_z.json` / `NO API KEY`.**
- `NO WIFI` — set up a network via `[W]` and make sure it's in range.
- `NO grammana_z.json` — the config file isn't on the drive root (or is
  still named `claude.json` from an older build — rename it).
- `NO API KEY` — the key field for your chosen provider is empty.

**Sync doesn't work / `[S]` isn't on the menu.**
`[S]` only appears when `config.sync.url` is set. There's no on-device way
to set it — add your Google Apps Script URL to `config.json` in Drive Mode.

**I forgot a file's password.**
There is no recovery — the file is permanently unreadable. This is by
design (real encryption). Always test on a throwaway file first.

**A menu label looks cut off / cramped.**
The panel is 400×300 and the font is fixed-width; a few of the longest
labels are deliberately abbreviated to fit. Nothing is wrong.

**Does an update erase my notes?**
No. This image doesn't touch the data partition. (Official releases do —
back up first if you flash one of those.)

## Security notes

- Your `grammana_z.json` holds a real API key — it stays on the device and
  is sent only as an auth header to the configured provider. It is
  **git-ignored here and has never been committed**; keep it that way if
  you fork this repo.
- **All outbound HTTPS** — LLM calls *and* Google Drive sync — verifies the
  server's certificate against a bundled root CA trust store
  (`src/service/Certs/Certs.h`), so a network attacker can't impersonate
  the endpoint to capture your key or text. (The stock firmware skips this
  check; this mod adds it.) A forged/mismatched certificate makes the
  request **fail** rather than leak. Because validation needs the real
  date, the device syncs its clock over NTP the first time WiFi comes up
  each power cycle — a few extra seconds on that first request.
- Grammana Z sends the **whole current file** (or the selection, for
  grammar) to your chosen provider. Don't use it inside files you wouldn't
  share with that provider.
- File encryption protects protected files **at rest** — in Drive Mode,
  sync uploads, and backups — but titles show `LOCKED`, file sizes remain
  visible, and the other, unprotected notes are plain text.
- PBKDF2 runs 6,000 iterations (an ESP32-class budget). A long passphrase
  matters more than the iteration count.

## Known limitations

- **No real battery percentage.** The battery level shown when the deck is
  paired as a Bluetooth keyboard is a fixed default from the BLE keyboard
  library, not a measurement. The rev 8 doesn't route the battery to an ADC
  pin the firmware can read, so a genuine percentage isn't possible without
  a hardware change.
- **Word counts for large (>8 KB) protected files** only cover the loaded
  window.
- **Ten note slots** (`0`–`9`) plus the three tools — the file model is
  fixed.
- **Italic is shown as underline** — the panel can't slant glyphs.

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

The patch also keeps `rev_4_68` and `rev_6` building, so you can sanity-
check that the shared editor changes don't break other revisions.

## Implementation notes

- Editor engine changes are shared by all Micro Journal revisions but
  gated behind `Editor::selectionSupported`, which only the Rev 8 (RLCD)
  display enables — other devices are unaffected.
- New key codes are 2100–2129 in `src/service/Editor/Editor.h`; the
  encryption service is `src/service/Crypt/`, the LLM service
  `src/service/AskClaude/`, and the HTTPS root CA trust store
  `src/service/Certs/`.
- The menu and dialogs render in `profont22` — after hardware testing, the
  only font family that stays readable on this panel (condensed fonts pack
  letters together; thin-stroke fonts render ragged).
- The undo engine and the encryption offset math were verified with
  host-side AddressSanitizer harnesses built from the shipped code
  (windowed reads at odd offsets, fast and splice save paths, 300
  randomized edit cycles).

## Software guide

An illustrated, screen-by-screen walkthrough (with mockups of every
screen) lives in **`GUIDE.html`** — open it in any browser. It's the
friendliest way to learn the device; this README is the reference.

## Credits & license

Built on [Micro Journal](https://github.com/unkyulee/micro-journal) by
**Un Kyu Lee**, licensed **CC BY-NC 4.0** — this mod inherits that license:
share and adapt freely with attribution, no commercial use. The patch
touches only the `micro-journal-rev-4-esp32` firmware tree.
