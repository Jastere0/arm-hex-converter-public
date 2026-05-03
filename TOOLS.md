# Tool Reference

Every feature in the tool, with what it does and how to use it.

---

## Main converter

The center of the app. Always visible.

### ASM → HEX mode

**What it does:** Translates ARM/ARM64/THUMB assembly into hex bytes.

**Why it's useful:** Writing patches. You know what instructions you want; you need the bytes to write into the binary.

**How to use:**
1. Click `ASM → HEX` at the top
2. Pick an architecture (ARM v7, ARM64, or THUMB)
3. (Optional) Set an offset in the `Offset (hex)` field — this becomes the PC for relative branches
4. Type assembly into the left panel
5. Hex bytes appear on the right, automatically

**Example:**
```
Input (ARM64):
  MOV W0, #1
  RET

Output:
  20 00 80 52  C0 03 5F D6
```

The tool converts as you type. Errors show inline below the offending line.

**Multi-architecture output:** All three arches assemble simultaneously. Switch the active arch to see the right encoding for the binary you're patching.

---

### HEX → ASM mode

**What it does:** Reverses the above — paste hex bytes, get disassembly back.

**Why it's useful:** You're looking at unknown bytes (from a hex dump, a memory inspector, a forum post) and want to know what instructions they represent.

**How to use:**
1. Click `HEX → ASM`
2. Pick an architecture
3. Paste hex bytes into the left panel — formatting doesn't matter:
   - `20 00 80 52` ✓
   - `0x20 0x00 0x80 0x52` ✓
   - `200080 52` ✓
   - `20,00,80,52` ✓
4. Disassembly appears on the right

**Example:**
```
Input:
  20 00 80 52 C0 03 5F D6

Output (ARM64):
  mov   w0, #1
  ret
```

---

### Quick Funcs mode

**What it does:** Pre-baked one-click patches for "make this function return X".

**Why it's useful:** Most patching needs are "make this boolean return true" or "make this function return 0". Saves the few seconds of typing the same patches you've written 50 times.

**How to use:**
1. Click `⚡ Quick Funcs`
2. The right panel shows cards for each return type:
   - **INT** — 32-bit integer return (e.g., `int GetCoins()` → returns 999999)
   - **LONG / INT64** — 64-bit integer return
   - **FLOAT** — single-precision float return (e.g., `float GetMultiplier()` → returns 1.0)
   - **DOUBLE** — double-precision float return
   - **BOOL** — boolean (1 = true, 0 = false)
   - **VOID** — empty function (just RET / BX LR)
3. Type a value if needed (e.g., `1` for true)
4. Each card shows the bytes for ARM, ARM64, and THUMB
5. Click `Copy` on whichever you need

**Example:**
```
Type "BOOL", value=true → 

ARM64:  20 00 80 52  C0 03 5F D6   (MOV W0, #1; RET)
ARM:    01 00 A0 E3  1E FF 2F E1   (MOV R0, #1; BX LR)
THUMB:  01 20 70 47                (MOVS R0, #1; BX LR)
```

---

## Architecture selector

**What it does:** Switches between ARM (v7), ARM64 (v8), and THUMB.

**Why it matters:** Different game binaries target different architectures. Picking the wrong one produces nonsense bytes.

**How to know which:**
- Look at your `.so` file's APK folder. `lib/arm64-v8a/libil2cpp.so` → ARM64. `lib/armeabi-v7a/libil2cpp.so` → ARM (or THUMB depending on the build).
- The IL2CPP browser auto-detects the arch from the ELF header when you load a `.so`.
- ARM64 is the dominant target for modern games (post-2019). ARM v7 still exists for older devices.

---

## Offset (hex)

**What it does:** Sets the program counter (PC) for the conversion.

**Why it matters:** PC-relative branches (`B target`, `BL target`, `ADR Rd, label`) need a real address to compute the encoded displacement. With offset = `0x562030`, a `B 0x562040` becomes a `+16` byte displacement. Without an offset set, branches are computed relative to address 0.

**Example:**
```
Without offset:        With offset = 0x1000:
  B 0x1010                B 0x1010
  encoded as: F2 03 00 EA  encoded as: 02 00 00 EA
  (different relative displacement)
```

When you `Send to patcher` from IL2CPP browser or Compare, the offset is auto-set to the method's offset.

---

## Diff mode

**What it does:** Splits the layout into "original" (top) and "modified" (bottom) panels for side-by-side comparison.

**Why it's useful:** Before applying a patch, you want to confirm it's the same length as the original method (otherwise you'll overflow into the next function). Diff mode shows both side by side with byte-level differences highlighted.

**How to use:**
1. Toggle `Diff mode` checkbox
2. Two panels appear:
   - Top: paste/edit the **original** bytes
   - Bottom: paste/edit your **patched** bytes
3. Differences are highlighted automatically

---

## Address gutter

**What it does:** Shows an address column on the left of input/output.

**Why it's useful:** When working with longer routines, you want to know "this instruction is at offset +0x14, that one's at +0x18". Without a gutter, you have to count by hand.

**How it works:** When the offset field has a value (e.g., `0x562030`), each line's address is shown: `0x562030`, `0x562034`, etc.

---

## Smart paste

**What it does:** Auto-detects whether what you pasted is hex or assembly, and switches mode accordingly.

**Why it's useful:** Saves clicking. Paste hex from a hex editor → mode flips to HEX→ASM automatically. Paste asm from a forum post → flips to ASM→HEX.

**How it decides:**
- All-hex characters with whitespace/separators → HEX
- Contains mnemonics (MOV, LDR, B, RET, etc.) → ASM
- Ambiguous → leaves mode alone

---

## 🧬 IL2CPP Browser

**What it does:** Loads `dump.cs` + `libil2cpp.so` together. Lets you browse the game's classes and methods, see disassembly, send any method to the converter for patching.

**Why it's useful:** This is the entry point for IL2CPP modding. `dump.cs` (produced by Il2CppDumper) has class definitions and method offsets. The `.so` has the actual bytes. Combined, you get a navigable map of the game's logic.

**How to use:**
1. Click `🧬 IL2CPP` in the titlebar
2. Browse to your `dump.cs` and `libil2cpp.so` (these come from Il2CppDumper)
3. Click `Load`
4. Pick architecture (ARM64 / ARM / THUMB)
5. The left panel shows classes in tree form. Click to expand.
6. Pattern tagging runs in the background, marking methods as `bool-true`, `bool-false`, `empty`, `return-const`, `getter`, `setter`. Filter by tag to find common patch targets quickly.
7. Click any method → right panel shows its disassembly
8. Click `Send to patcher` → main window switches to HEX→ASM with the bytes loaded, offset set, ready to patch

**Example workflow:**
```
1. Open IL2CPP browser
2. Load dump.cs + libil2cpp.so
3. Search "IsPaid" → find method GameLogic.IsPaid()
4. Click → see "MOV W0, #0; RET" (returns false always)
5. Click "Send to patcher"
6. Switch to ASM→HEX, type "MOV W0, #1; RET"
7. Hex output: 20 00 80 52 C0 03 5F D6
8. Save to history (auto-logs to Journal)
```

**Search behavior:** Searching narrows the visible classes. Class headers stay collapsed; click to expand and see only matching methods. This keeps the UI fast even with 50K methods.

**Pattern filter buttons:** `All`, `bool-true`, `bool-false`, `empty`, `return-const`, `getter`, `setter`. Click any to show only methods matching that pattern.

---

## 🔀 Compare versions

**What it does:** Loads two pairs of `(dump.cs, .so)` — typically v1.0 and v1.1 of the same game. Diffs them. Tells you which methods are unchanged, changed, added, removed.

**Why it's useful:** When a game updates, you want to know what got modified. Compare gives you that list, directly.

**How to use:**
1. Click `🔀 Compare`
2. Pick OLD `dump.cs`, OLD `.so`, NEW `dump.cs`, NEW `.so`
3. Click `Run Compare`
4. Results categorized:
   - **🔄 Changed** — same method exists in both, but bytes differ
   - **❓ Uncertain** — possible match below confidence threshold; review manually
   - **➕ Added** — exists only in NEW
   - **➖ Removed** — exists only in OLD
   - **Unchanged** — same offset, same bytes (usually hidden by default)
5. Click any entry → right panel shows OLD bytes and NEW bytes side by side
6. Click `Send OLD to patcher` or `Send NEW to patcher` to load it into the converter

**Filter buttons:** `All`, `🔄 Changed`, `❓ Uncertain`, `➕ Added`, `➖ Removed`, `Unchanged`. Click any to narrow.

**How matching works:**
1. **Exact name match** — methods with same `Class.Signature` in both versions, in the same class, get matched immediately.
2. **Byte-similarity match** — for unmatched methods, the tool tries byte-similarity matching within a position window in the same class. Catches obfuscated methods whose names changed but whose code is mostly the same.
3. Each match has a confidence score. Below 0.7 → flagged as Uncertain.

---

## 📓 Journal

**What it does:** Auto-logs every patch you make. Each entry stores class, method, offset, original bytes, patched bytes, asm source, fingerprint bytes for migration.

**Why it's useful:** A running record of your work. Reload any past patch, copy hex, export as a `.patches.txt`, or migrate offsets when the game updates.

**How to use:**
1. Click `📓 Journal`
2. List of all patches you've ever made (persistent across sessions, stored in `%APPDATA%/ARMHexConverter/journal.json`)
3. Per-entry buttons:
   - `Load` — opens the patch in the converter (mode, arch, offset, asm all set)
   - `Copy hex` — puts the patched bytes on clipboard
   - `Delete` — removes from journal
4. Top buttons:
   - `Migrate to new version…` — see below
   - `Export…` — save as `.txt` with comments and metadata
   - `Clear journal` — wipe (with confirmation)

**What gets logged:** Every time you click `Save to history` in ASM→HEX mode with a non-empty offset and non-empty bytes. The class+signature comes from the patch context (set when you Sent from IL2CPP/Compare).

**Example entry:**
```
2024-09-15 14:32:18 — GameLogic.IsPaid()
  arch: ARM64,  offset: 0x562030,  size: 8 bytes
  
  Original: E0 03 1F 2A C0 03 5F D6
  Patched:  20 00 80 52 C0 03 5F D6
  
  Source asm:
    MOV W0, #1
    RET
```

---

## 📓 Journal → Migrate to new version

**What it does:** Finds where your existing patches moved when the game updates. Uses byte-fingerprint matching against a new `(dump.cs, .so)` pair.

**Why it's useful:** Without this, every game update means re-finding every method by hand. With it, you click a button and 30 seconds later your offsets are updated.

**How it works:**
1. Each Journal entry stores 64 bytes BEFORE and 64 bytes AFTER the patch site (read from the source `.so` at patch time).
2. Migration scans every method in the new `dump.cs` and compares its surrounding bytes to your stored fingerprints.
3. Highest-scoring candidate wins. Confidence is classified:
   - **✓ exact (100%)** — bytes are identical (method unchanged)
   - **~ fuzzy (70-99%)** — surrounding code changed slightly, still likely the same method
   - **? uncertain (55-69%)** — match exists but weak; review before applying
   - **✗ not found (<55%)** — method was removed, inlined, or heavily rewritten
   - **⊘ no fingerprint** — entry was logged before this feature existed; can't migrate

**How to use:**
1. Open `📓 Journal`
2. Click `Migrate to new version…`
3. Pick the new `dump.cs` and new `.so`
4. Click `Scan`
5. Review results
6. Click `Apply migrations` to update Journal entries' offsets in place. Old offsets are saved in each entry's migration history for audit.

After applying, your Journal entries point at the new version's offsets and have new fingerprints recorded for the next migration.

---

## 📋 Templates

**What it does:** Gallery of pre-made patch patterns plus user-saved custom templates.

**Why it's useful:** "Make this return true" is a five-byte instruction sequence you've written hundreds of times. One click instead.

**Built-in templates:**
- **Force return TRUE** — `MOV W0, #1; RET` (or arch equivalent)
- **Force return FALSE** — `MOV W0, #0; RET`
- **Skip integrity check** — Same as return false but tagged for that intent
- (more entries as defined in code)

**Adding your own:** Type assembly in the input → click `Save as template…` (in the input card's action row) → give it a name and category → it appears in Templates dialog.

**How to use:**
1. Click `📋 Templates`
2. Browse by category (Bypass, Constants, Bytecode, etc.)
3. Click any template card → loads its assembly into the input

**Pinning:** Click the star on a template to pin it. Pinned templates appear at the top of the gallery.

---

## ✓ Verify

**What it does:** Side-by-side validator. Paste original bytes and patched bytes; tells you whether the patch is byte-equivalent (drop-in safe), smaller (need to NOP-pad), or larger (will overflow into the next method — DANGEROUS).

**Why it's useful:** Before flashing a patch onto a real binary, you want to sanity check it.

**How to use:**
1. Click `✓ Verify`
2. Paste original bytes in the top panel
3. Paste patched bytes in the bottom panel
4. Status shows:
   - ✓ green if same length
   - ⚠ yellow if shorter (suggests NOP padding)
   - ✗ red if longer (overflow warning)

---

## 🔬 Bytes Diff

**What it does:** Compares two arbitrary hex strings byte-by-byte. Highlights differences with offsets.

**Why it's useful:** Generic byte-comparison tool. When you're comparing the same method across two builds you have on disk, or comparing your patch against what's actually in the file after applying it.

**How to use:**
1. Click `🔬 Bytes Diff`
2. Paste hex into the two input fields
3. Differences are highlighted with their byte offset and the differing values

---

## 🕒 History

**What it does:** Sliding side panel showing your last 200 conversions. Persistent across sessions.

**Why it's useful:** Quick access to recent work without polluting Journal (which is patches-only).

**Difference from Journal:**
- History: every conversion (any mode, any arch). Last 200.
- Journal: only patches (ASM→HEX with offset). Last 500. Has fingerprints, class context, migration support.

**How to use:**
1. Click `🕒 History`
2. Side panel slides out
3. Click any entry → restores it (input, mode, arch, offset)

---

## 📖 Manual

**What it does:** Searchable reference of every assembly instruction the tool supports, with a working example for each.

**Why it's useful:** Lookup. "What's the syntax for pre-indexed LDR in ARM64? What condition codes does B support?"

**Organization:** Grouped by architecture (ARM, ARM64, THUMB) and category (data movement, arithmetic, branches, comparison, system).

**Searching:** Type into the search field at top. Filters entries live.

**How to use:**
1. Click `📖 Manual`
2. Browse or search
3. Each entry shows:
   - Mnemonic
   - One-line description
   - Working code example you can copy

---

## ❓ Guide

**What it does:** Beginner-oriented tutorial covering common modding workflows.

**Why it's useful:** First-time users want to understand what the tool is FOR before learning every feature.

**Sections cover:**
- **Main converter** — the basic workflow
- **Workflow tools** — IL2CPP, Compare, Journal, Templates, Verify
- **Analysis dialogs** — Bytes Diff, Manual, Shortcuts
- **Reference** — terminology and ELF/IL2CPP background

---

## 🎨 Theme

**What it does:** Picker for color theme.

**Available themes (all dark):**
- Neon Green (default)
- Midnight Blue
- GitHub Dark
- Tokyo Night
- Catppuccin Mocha
- Cyberpunk
- Nord
- Monokai
- Solarized Dark
- Amber Terminal
- High Contrast

**How to use:**
1. Click `🎨 Theme`
2. Pick one
3. Applied immediately

The choice is saved in `settings.json` and restored on next launch.

---

## ⌨ Shortcuts

**What it does:** Lists all keyboard shortcuts.

**Notable shortcuts:**
- `Ctrl+M` — toggle ASM↔HEX modes
- `Ctrl+Enter` — save to history
- `Ctrl+L` — clear input
- `Esc` — close current dialog
- `Ctrl+F` — focus search box (in dialogs that have one)

---

## ⋯ More menu

When the window is too narrow to fit all titlebar buttons, the rightmost button becomes `⋯ More`. It opens a dropdown containing the buttons that didn't fit.

**Priority order** (which buttons hide first when narrow):
1. **Always visible:** Journal, Compare, IL2CPP — primary workflow tools
2. **Hide second:** Manual, Guide, Shortcuts — reference
3. **Hide first:** Templates, Verify, Bytes Diff, Theme, History — secondary

So even on a 1280-wide screen, you'll always have your primary buttons visible.

---

## Patch context strip

When you `Send to patcher` from IL2CPP or Compare, a band appears above the input panel showing what you're patching:

```
PATCHING
GameLogic  ·  public bool IsPaid()                          [Clear ✕]
8 bytes at 0x562030  ·  ARM64 (v8)
```

**Why it's useful:**
- Reminds you what method you're working on
- Shows the original size (so a size warning fires if your patch is bigger)
- Auto-tags the resulting Journal entry with class/signature/original-bytes

**Clearing it:** Click `Clear ✕`, or click `Clear` on the input card, or switch to Quick Funcs mode. The strip disappears and patches you make are no longer tagged with the previous method's context.

---

## Architecture detection (Compare dialog)

When Compare loads `.so` files, it auto-detects each side's architecture by sampling 50 methods and scoring them as ARM/ARM64/THUMB. The arch row shows the result:

```
OLD: ARM64 (v8)    NEW: ARM64 (v8)
```

If they differ, the row turns red with a warning — comparing different arches usually means you picked the wrong files.

---

## Window state persistence

Every dialog (IL2CPP browser, Compare, Journal, Migrate, Templates, Manual, Guide, Bytes Diff, Verify, Shortcuts, Theme picker) remembers:
- Window size
- Window position
- Last selected entry (where applicable)
- Search query
- Filter state
- Page size / pagination position

Stored in `settings.json`. Cleared if you delete the file.

---

## Quick conversion troubleshooting

**Wrong bytes for the architecture you expect:**
- Check the architecture selector. ARM and ARM64 use very different encodings.
- ARM64 uses `W0`/`X0` registers; ARM uses `R0`. Mixing them produces errors.

**Branches don't resolve correctly:**
- Set the offset. Without it, branches are relative to address 0.

**Patch is the wrong size:**
- Use Diff mode or Verify to compare against the original.
- Pad with NOPs (`NOP` for ARM, `NOP` for ARM64, `NOP` for THUMB-2 — Keystone resolves to the right encoding).

**Hex paste isn't working:**
- Hex must contain only `0-9 A-F a-f` and separator characters. Non-hex characters cause parse errors with line numbers.
