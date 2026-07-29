# MacroJournal — a Micro Journal Rev 8 firmware mod

Custom firmware for the [Micro Journal Rev 8](https://github.com/unkyulee/micro-journal)
(the ESP32-S3 writerdeck by Un Kyu Lee) that turns the stock editor into a
small but complete, keyboard-driven writing environment with a classic
Macintosh-style shell.

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
- **A classic-Mac home** — a pull-down menu bar over a Finder-style
  desktop: quick-launch tool tiles, a scrolling notes window with titles,
  and a keyboard-driven action bar.
- **Note names** — give any note a real name; it shows on the home screen,
  in the Notes menu, and on synced files.
- **A scratchpad** — an always-there note, plus "send selection to
  scratchpad" while writing.
- **Grammana Z** — an on-device LLM assistant (Anthropic, OpenAI, or
  Gemini) with grammar checking and rewriting.
- **Dark mode**, **device rename**, an **About** box, a **Storage** gauge,
  and quality-of-life polish throughout.

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
- [The home screen](#the-home-screen)
- [The menu bar](#the-menu-bar)
- [Working with notes](#working-with-notes)
- [Note names & rename](#note-names--rename)
- [The editor](#the-editor)
- [Markdown styling](#markdown-styling)
- [Undo / redo](#undo--redo)
- [Scratchpad](#scratchpad)
- [Grammana Z (LLM assistant)](#grammana-z-llm-assistant)
- [Grammar check & rewrite](#grammar-check--rewrite)
- [Per-file passwords (encryption)](#per-file-passwords-encryption)
- [Dark mode](#dark-mode)
- [Device name](#device-name)
- [About & Storage](#about--storage)
- [WiFi, language, Bluetooth & drive mode](#wifi-language-bluetooth--drive-mode)
- [Sync to Google Drive](#sync-to-google-drive)
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
| `rev8-text-selection-clipboard.patch` | The source diff of the editor/feature layer against upstream. |
| `keyboard.json` | A custom key layout with the mod's key names wired in (see [keyboard.json](#keyboardjson)). Copy to the device drive root. |
| `grammana_z.json.example` | Template for the LLM assistant config. Copy to the device as `grammana_z.json` with your API key. |
| `sync.js` | The Google Apps Script for [Sync](#sync-to-google-drive), including the note-name titling. Paste into your own Apps Script project. |
| `GUIDE.html` | The illustrated software guide (open in any browser). |

`grammana_z.json` (with a real API key) is intentionally **not** in this
repo — it is git-ignored. Never commit it.

## Requirements

- A **Micro Journal Rev 8** (ESP32-S3 writerdeck, 400×300 reflective LCD).
  This firmware targets the `rev_8` build only.
- A computer with **Chrome or Edge** for the web flasher (Web Serial).
- A USB-C cable.
- Optional, for the LLM and sync features: a WiFi network and an API key
  from Anthropic, OpenAI, or Google.

## Flashing the firmware

1. Plug the **upper** USB-C port into your computer and open the web
   flasher: <https://www.espboards.dev/tools/program/> (Chrome/Edge).
2. Click **Connect** and select the serial port.
3. Upload **`firmware_rev_8.bin`** and set the address to **`0x0`**.
4. Click **Program**. It takes about a minute.

Notes:

- Always flash **the merged image at `0x0`**. It already contains the
  bootloader, partition table, and app at their correct offsets — do **not**
  put it at `0x10000`. An app-only `firmware.bin` at `0x0`, or the merged
  image at the wrong address, boot-loops with `Invalid image block`.
- Unlike official releases, this image does **not** wipe the data
  partition — your journal files and `keyboard.json` survive an update.
- If it seems to hang for many minutes, see
  [Troubleshooting](#troubleshooting--faq).

## First-time setup

1. **Flash** the firmware (above) and let the device boot into the editor.
2. Copy `keyboard.json` from this repo onto the drive (see below) if you
   want the author's layout, or make sure your own layout includes the
   mod's key names — otherwise the new shortcuts do nothing.
3. To reach the drive: from a note press **Esc** (or the menu key) to open
   the menu bar, go to **Device ▸ Drive mode**, and the deck mounts as a
   USB drive on your computer. Copy files to/from its root, then eject.
4. Optional — **WiFi**: menu bar → **Setup ▸ WiFi**, add a network (needed
   for sync and Grammana Z).
5. Optional — **Grammana Z**: copy `grammana_z.json` (from the example,
   with your API key) to the drive root. The Grammana entry works once the
   file is present.

---

## The home screen

The home screen is a classic-Mac desktop. When you leave a note you land
here; the deck also boots straight into your last note, and **Esc** from
that note brings up the [menu bar](#the-menu-bar) over the home screen.

It has three parts:

- **Menu bar** across the top — `File · Notes · Tools · Setup · Device`.
- **Tool tiles** — three quick-launch tiles: **Scratch** (scratchpad),
  **Gramma** (Grammana Z), and **Last** (the note you edited most
  recently).
- **Notes window** — a titled, scrolling window listing your ten notes
  `[0]`–`[9]`. Each row shows the note's **name** (if set) or its first
  line as a title, `-` when empty, or `LOCKED` when encrypted. A scroll bar
  and a down-arrow appear when there are more rows than fit.
- **Action bar** at the bottom — the keys that act on the highlighted note:
  `C CLEAR · P LOCK · R RENAME`.

**Navigation**

- **Arrow keys** move the highlight. Left/Right step across the three
  tiles; **Down** drops from the tiles into the notes list; **Up** from the
  top note returns to the tiles. The notes list scrolls and wraps.
- **Enter** opens whatever is highlighted (a tool or a note).
- **Direct keys** work any time, without highlighting first:

| Key | Does |
|---|---|
| `0`–`9` | Open that note |
| `S` | Scratchpad |
| `G` | Grammana Z |
| `/` | Last note |
| `N` | New note (first empty slot) |
| `P` | Lock / unlock the highlighted note |
| `R` | Rename the highlighted note |
| `C` | Clear the highlighted note |
| `W` `L` `B` `I` `U` | WiFi · Language · BLE keyboard · Dark mode · Drive mode |
| `Esc` / menu key | Open the menu bar |

## The menu bar

The pull-down menu bar is the classic way to reach everything. Open it and
you can drive the whole device from the keyboard.

**Opening it**

- From the **home screen**: press **Esc** or the menu key.
- From **inside a note**: press **Esc**, the menu key, or **double-tap the
  Alt key** (the key next to Ctrl/GUI, tapped twice quickly).

**Using it**

- **←/→** move between the five menus; **↑/↓** move within the open menu;
  **Enter** runs the highlighted item.
- A menu item's **shortcut letter** is shown on its right, and pressing that
  letter runs it from anywhere in the bar.
- The escape ladder from an open menu:
  - **Esc** → leave the bar (to the home screen, or back to the note if you
    opened it from a sub-screen).
  - **Shift+Esc** (in a note) → close the bar and drop the cursor back
    exactly where it was.

**The menus**

| Menu | Items |
|---|---|
| **File** | New note (`N`); for the selected note: Rename (`R`), Lock / Unlock (`P`), Clear (`C`) |
| **Notes** | Every non-empty note, by name or title — jump straight to one |
| **Tools** | Grammana (`G`), Scratchpad (`S`) |
| **Setup** | WiFi (`W`), Language (`L`), BLE keyboard (`B`), Dark on/off (`I`), Sync (when configured) |
| **Device** | About, Device name, Storage, Drive mode (`U`), Restart |

## Working with notes

You have **ten notes** (`0`–`9`) plus three built-in tools (Scratchpad,
Grammana Z, Last note). Open any of them from the home screen — arrow to it
and press **Enter**, or press its direct key.

**Acting on a note** — highlight a note in the notes window, then:

| Key | Action |
|---|---|
| `Enter` | Open it |
| `R` | **Rename** — give it a name (or clear the name) |
| `P` | Set / remove its **password** |
| `C` | **Clear** it (asks to confirm; locked notes verify the password first) |

These are exactly the `File` menu's items and the action bar at the bottom
of the home screen — use whichever you like. Clearing keeps a
`*_backup.txt`.

**New notes** — press **`N`** on the home screen, choose **File ▸ New
note**, or press **Ctrl+N from anywhere** (even mid-sentence in another
note). It opens the first empty slot, saving your current note first. If
all ten slots are in use, nothing happens.

## Note names & rename

Any note can have a real name instead of showing its first line.

- **Rename** with `R` on the home screen, **File ▸ Rename**, or the
  `R RENAME` action bar. A small dialog opens pre-filled with the current
  name: type up to 20 characters, **Enter** saves, **Esc** cancels. Clear
  the field and save to drop the name and go back to the first-line title.
- A named note shows that name on the home screen, in the **Notes** menu,
  and as the title of its [synced](#sync-to-google-drive) Google Drive file.
- Names are stored in `config.json` under a `names` object (keyed by slot),
  so they travel with your settings and never touch the note's text. New
  notes start unnamed.

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
| New note | Ctrl + N |
| Open / send-to scratchpad | Ctrl+Shift + S |
| Grammana Z assistant | Ctrl+Shift + C |
| Grammar check | Ctrl + G |
| Grammar rewrite | Ctrl+Shift + G |
| Open the menu bar | Esc, menu key, or double-tap Alt |

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

**Ctrl+Shift+S**, the **Scratch** tile, **Tools ▸ Scratchpad**, or `S` on
the home screen opens a plain-text scratchpad — a quick, always-there note
(status bar `SCRATCHPAD`). ESC saves it and returns you to the file you were
in. It lives in slot 10 (`10.txt` on the drive, included in sync).

**Send selection to scratchpad** — if you have text highlighted in a note,
**Ctrl+Shift+S** instead appends that selection to the scratchpad (on its
own line) *without leaving the note*. A quick way to stash a snippet while
writing: the note and your selection are untouched, and a `SENT TO SCRATCH`
flash by the word count confirms it for 3 seconds. (Sending from an
encrypted note writes that snippet to the scratchpad in plain text, since
the scratchpad itself isn't encrypted.)

## Grammana Z (LLM assistant)

The **Gramma** tile, **Tools ▸ Grammana**, `G` on the home screen, or
**Ctrl+Shift+C** in the editor opens the assistant (status bar `GZ`).

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

To lock or unlock a note, highlight it on the home screen and press **`P`**
(or use **File ▸ Lock / Unlock**) — locked notes verify the old password
first, so you don't have to open the file.

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

**Setup ▸ Dark**, or `I` on the home screen, toggles **hardware panel
inversion** — white text on black. Everything (selection, markdown styling,
menu highlights) renders inverted automatically. It persists across
reboots.

## Device name

The device name (default **MacroJournal**) shows on the home window's title
bar and the sub-screen headers, and it's yours to change: open **Device ▸
Device name**, type a new name (up to 16 characters) into the dialog, and
press **Enter** — ESC cancels. Clearing the field resets it to
`MacroJournal`. The name persists across reboots (stored in `config.json`
as `"title"`, so you can also set it from a computer in Drive Mode).

## About & Storage

Two informational screens live under the **Device** menu:

- **About** — a full-screen "About this MacroJournal" card with the Happy
  Mac mark, your device name, the firmware version, and the upstream
  credit. **Esc** returns to the home screen.
- **Storage** — a Get-Info-style box with a usage gauge and the real
  **Total / Used / Free** figures for the internal data partition, plus how
  many of your ten note slots are in use. **Esc** returns home. (This is the
  *free-space* view; **Drive mode** is the separate USB one below.)

## WiFi, language, Bluetooth & drive mode

These come from the stock firmware and behave as normal. Reach them from
the menu bar (or their home-screen hotkeys).

- **Setup ▸ WiFi** (`W`) — add and edit WiFi networks (stored on the
  device's internal flash, not the drive). Needed for Sync and Grammana Z.
  The radio is only powered during a request and switched off afterward.
- **Setup ▸ Language** (`L`) — pick the keyboard layout (US, UK, DE, FR,
  and more). Saved to config.
- **Setup ▸ BLE keyboard** (`B`) — the deck advertises itself as a
  **Bluetooth keyboard**; pair it from a computer or phone and what you type
  on the deck is sent to that device. (The battery level it reports is a
  fixed default, not a real reading — see
  [Known limitations](#known-limitations).)
- **Device ▸ Drive mode** (`U`) — **USB Drive Mode**. The device mounts as
  a USB mass storage drive so you can copy files and configs to/from its
  root. Exiting reboots the device to remount the filesystem cleanly.
- **Device ▸ Restart** — reboot the deck.

## Sync to Google Drive

**Setup ▸ Sync** uploads the current note to Google Drive through a Google
Apps Script endpoint. The entry only appears once a URL is configured (in
`config.json` under `sync.url`; there is no on-device setup for the URL).

**Named files** — the deck sends the note's name along with the upload, so
the file in Drive is titled after the note (a named note by its name, an
unnamed note by its first line, the scratchpad and Grammana as
`Scratchpad` / `Grammana`). This needs the bundled script:

1. Open your Google Apps Script project (the one behind your `sync.url`).
2. Replace its contents with **`sync.js`** from this repo and **redeploy**.
3. Set the folder path at the top of the script if you haven't already.

Until the script is updated the deck still syncs exactly as before — the
name it sends is simply ignored, and files keep the old timestamp name.
All sync traffic verifies the server certificate (see
[Security notes](#security-notes)).

---

## Configuration files

Config files live in the **drive root** (reach it with **Device ▸ Drive
mode**).

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

The device's WiFi setup. Configure it **on-device** via **Setup ▸ WiFi**
(it's saved to internal flash for safety, not to the drive). Needed for
Sync and the LLM features.

### keyboard.json

A `/keyboard.json` on the drive **replaces the compiled key layout
entirely**. The one in this repo matches the author's preferences (backtick
next to A, an extra Shift, `"` on the backslash key) **and contains the
mod's key names** (`CTRL`, `GUI`, `SELECT_LEFT`, `WORD_RIGHT`, `DOC_END`,
…). If you use your own layout, you must add those names or the new
features silently go dead. Remove the file to revert to the compiled
layout.

> Note: `Ctrl+N` (new note) and the other Ctrl-combos resolve in the
> compiled firmware, not in `keyboard.json`, so they work regardless of
> your layout file.

### config.json

The device's own settings file (created automatically). Notable fields
this mod uses: `title` (device name), `names` (per-note names, keyed by
slot), `darkmode`, `last_note`, and `sync.url` (Google Drive endpoint).
You normally change these through the menu, but they're editable in Drive
Mode too.

## Files on the drive

| File(s) | What it is |
|---|---|
| `0.txt` – `9.txt` | Your ten notes |
| `10.txt` | The scratchpad |
| `11.txt` | The Grammana Z conversation |
| `12.txt` | The last grammar-check / rewrite report |
| `*_backup.txt` | Automatic backup made when a file is cleared |
| `config.json` | Device settings (title, note names, dark mode, sync URL) |
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

**The device boot-loops after flashing (`Invalid image block`).**
The merged image was flashed at the wrong address, or you flashed an
app-only `firmware.bin`. Flash the **merged** `firmware_rev_8.bin` at
**`0x0`** (not `0x10000`).

**The new shortcuts (selection, etc.) do nothing.**
Your `keyboard.json` on the drive is overriding the layout and lacks the
mod's key names. Use this repo's `keyboard.json`, or add the new names to
yours. (A `keyboard.json` replaces the compiled layout entirely.)

**Grammana Z says `NO WIFI` / `NO grammana_z.json` / `NO API KEY`.**
- `NO WIFI` — set up a network via **Setup ▸ WiFi** and make sure it's in
  range.
- `NO grammana_z.json` — the config file isn't on the drive root (or is
  still named `claude.json` from an older build — rename it).
- `NO API KEY` — the key field for your chosen provider is empty.

**Sync doesn't work / there's no Sync entry.**
Sync only appears when `config.sync.url` is set. There's no on-device way
to set it — add your Google Apps Script URL to `config.json` in Drive Mode.

**My synced files still have timestamp names, not note names.**
The deployed Apps Script hasn't been updated yet. Paste this repo's
`sync.js` into your Apps Script project and redeploy (see
[Sync](#sync-to-google-drive)).

**I forgot a note's password.**
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

The bundled `rev8-text-selection-clipboard.patch` captures the editor and
feature layer against upstream:

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
  --flash_mode dio --flash_size 16MB \
  0x0 bootloader.bin 0x8000 partitions.bin \
  0xe000 <arduino-esp32>/tools/partitions/boot_app0.bin \
  0x10000 firmware.bin
```

> The patch tracks the editor/feature layer. The classic-Mac shell
> (menu bar, home screen, About/Storage, note names) is newer than the
> patch snapshot — check the release notes for the matching source drop if
> you're building the full UI.

## Implementation notes

- Editor engine changes are shared by all Micro Journal revisions but
  gated behind `Editor::selectionSupported`, which only the Rev 8 (RLCD)
  display enables — other devices are unaffected.
- New key codes are 2100–2130 in `src/service/Editor/Editor.h`; the
  encryption service is `src/service/Crypt/`, note names
  `src/service/NoteNames/`, the LLM service `src/service/AskClaude/`, and
  the HTTPS root CA trust store `src/service/Certs/`.
- The classic-Mac UI lives in `src/display/RLCD/`: the shared menu bar in
  `MenuBar/`, the Happy Mac art in `Logo/`, and the home / About / Storage
  screens under `Menu/`.
- The menu and dialogs render in `profont22` — after hardware testing, the
  only font family that stays readable on this panel (condensed fonts pack
  letters together; thin-stroke fonts render ragged). UI text is drawn on a
  fixed, even-aligned cell pitch because the panel stores the horizontal
  axis at half resolution.
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
