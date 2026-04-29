# App Sync Patch — ADB over USB for HiBy R3 Pro II

**Binary:** `hiby_player` ELF32 MIPS32r2 Little Endian (Ingenic X1600, HiByOS)
**Binary SHA256:** `9c3b5178cf155b75f0b327ebc01d03b8075ba883723d6781d326bddf8344f8ee`
**Binary size:** 5,598,244 bytes (unchanged by patch)

---

## Overview

This patch adds an **App Sync** toggle to the device's system settings. When enabled and the device is connected via USB, ADB starts automatically instead of mass storage. A custom view page (`hiby_app_sync.view`) is displayed during sync, with a stop button to disable ADB and return to normal operation.

The patch does **not** modify the binary size. All injected code lives in two zero-filled code cave regions. All four pre-existing patches (Sorting, DB Manager, Playlist, DB Manager Dialog) are fully preserved.

---

## User-Facing Behavior

**Toggle ON:** creates the marker file `/usr/data/adb_usb_mode`. ADB does not start until USB is connected.

**Toggle OFF:** deletes the marker file.

**USB connect with toggle ON:** mass storage initializes normally, then `FUN_00480f20(1)` is called (the firmware's own ADB activation function, same as the 10-tap About page toggle), which stops mass storage and starts ADB. The `hiby_app_sync.view` page is shown instead of the standard USB page.

**USB connect with toggle OFF:** normal mass storage.

**USB disconnect with toggle ON:** the App Sync page stays visible (teardown is suppressed) (This prevents any action done on the device, e.g. playing music, while the device is in adb mode to prevent unwanted behaviours.

**Stop button on App Sync page:** removes the marker file, clears the runtime flag, calls `FUN_00480f20(0)` to stop ADB and restore mass storage, then closes the page.

**Reboot:** at boot, the early init reads the marker file and sets the runtime flag for the toggle display. ADB is **not** started at boot — it only starts when USB is physically connected.

---

## Patch Architecture

The patch consists of two groups: the **settings toggle** (adds the App Sync entry to the Settings page with a working on/off switch) and the **USB integration** (handles ADB activation on USB connect, the custom view page, and the stop button).

### Code Caves Used

| Region | Address Range | Size | Contents |
|--------|--------------|------|----------|
| Sorting cave extension | `0x41c120` – `0x41c420` | ~768 B | View trampoline, ADB gate, marker helper, teardown gate, stop callback, control table |
| `.rodata` zero region | `0x7f2d4c` – `0x7f2f70` | ~548 B | Click handler, NL trampolines, early init, toggle wrapper, POST_USB redirect, strings |

### Runtime Variables

| Address | Segment | Size | Purpose |
|---------|---------|------|---------|
| `0x9B6438` | BSS | 1 byte | Runtime flag: `1` = App Sync enabled, `0` = disabled |

### Marker File

`/usr/data/adb_usb_mode` — presence indicates App Sync is enabled. Created/deleted by the click handler. Checked at boot (early init) and by the ADB auto-start gate on USB connect.

---

## Part 1: Settings Toggle

### Dispatch Table

A new entry is added at slot 36 (address `0x937434`), with dispatch ID `0x24` and name `app_sync`.

| Address | Size | Description |
|---------|------|-------------|
| `0x937434` | 68 B | Dispatch entry: `{0x24, "app_sync\0", padding}` |

### Loop Limits

Two loop limit constants are incremented by 1 to include the new entry.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x4e0b38` | `addiu $s3, $zero, 0x24` | `addiu $s3, $zero, 0x25` | Dispatch lookup loop: 36 → 37 |
| `0x4e6bec` | `addiu $s3, $zero, 0x18` | `addiu $s3, $zero, 0x19` | Name list loop: 24 → 25 |

### Name List Trampolines

Two hook points add `"app_sync"` to the settings name list. Each replaces two `lui` instructions with a jump to a trampoline that stores the string pointer, then re-executes the replaced instructions and returns.

| Address | Original | Patched | Target |
|---------|----------|---------|--------|
| `0x4e6cf8` + `0x4e6cfc` | `lui $s0, 0x88` + `lui $v0, 0x82` | `j 0x7f2e20` + `nop` | Non-RS2 trampoline |
| `0x4e6b48` + `0x4e6b4c` | `lui $s0, 0x88` + `lui $v0, 0x82` | `j 0x7f2e40` + `nop` | RS2 trampoline |

**Non-RS2 trampoline** at `0x7f2e20` (28 B): stores `"app_sync"` pointer at `$sp+0x4c`, restores `$s0` and `$v0`, jumps to `0x4e6d00`.

**RS2 trampoline** at `0x7f2e40` (28 B): same logic, stores at `$sp+0x44`, jumps to `0x4e6b50`.

### Click Handler

Replaces the dispatch call at `0x4e14c0`.

| Address | Original | Patched |
|---------|----------|---------|
| `0x4e14c0` | `jal 0x4e0ae0` | `jal 0x7f2d4c` |

**Click handler** at `0x7f2d4c` (148 B):

1. Calls `FUN_004e0ae0` to get the dispatch ID for the clicked item
2. If dispatch ID ≠ `0x24`: returns the ID unchanged (other items handled normally)
3. If dispatch ID = `0x24` (App Sync):
   - Toggles the BSS flag at `0x9B6438` (XOR with 1)
   - If turning **ON**: `system("touch /usr/data/adb_usb_mode")`
   - If turning **OFF**: `system("rm -f /usr/data/adb_usb_mode")`
   - Calls `FUN_004af060($s2, "vg_listview_sys_set")` to refresh the list display
   - Returns `-1` (signals the dispatcher that the click was handled)

### Toggle Display Wrapper

Replaces both calls to `FUN_004e0be0` in the display callback, providing toggle state for App Sync without modifying the toggle state table or any struct fields.

| Address | Original | Patched |
|---------|----------|---------|
| `0x4e0eb4` | `jal 0x4e0be0` | `jal 0x7f2ea0` |
| `0x4e0ee4` | `jal 0x4e0be0` | `jal 0x7f2ea0` |

**Toggle wrapper** at `0x7f2ea0` (88 B):

1. Calls original `FUN_004e0be0` with the same argument
2. If result ≥ 0: returns it (existing toggle item)
3. If result = -1 (not a known toggle): calls `FUN_004e0ae0` to get dispatch ID
4. If dispatch ID = `0x24`: returns BSS flag value (0 or 1)
5. Otherwise: returns -1

### Early Init

Hooks the first function call in `FUN_0048b8e0` (app initialization).

| Address | Original | Patched |
|---------|----------|---------|
| `0x48b8f4` | `jal 0x47b380` | `jal 0x7f2e60` |

**Early init** at `0x7f2e60` (56 B):

1. Calls `access("/usr/data/adb_usb_mode", 0)` to check if marker file exists
2. Sets BSS flag at `0x9B6438` accordingly (`1` if file exists, `0` otherwise)
3. Calls original `FUN_0047b380`
4. Does **not** call `adbon` — ADB activation happens only on USB connect

### String Data

| Address | Content |
|---------|---------|
| `0x7f2f00` | `app_sync` |
| `0x7f2f0c` | `/usr/data/adb_usb_mode` |
| `0x7f2f24` | `touch /usr/data/adb_usb_mode` |
| `0x7f2f44` | `rm -f /usr/data/adb_usb_mode` |

---

## Part 2: USB Integration

### ADB Auto-Start on USB Connect

When the USB page constructor (`FUN_0052bfa0`) finishes building the page, control passes through a redirect to the ADB auto-start gate.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x52c32c` | `move $v0, $s1` | `j 0x7f2de0` | Redirect at end of USB page constructor |

**POST_USB redirect** at `0x7f2de0` (16 B): jumps to the ADB auto-start gate at `0x41c180`.

**ADB auto-start gate** at `0x41c180` (~72 B):

1. Checks BSS flag (`0x9B6438`): if 0, skips to return
2. Calls `access("/usr/data/adb_usb_mode", 0)`: if file doesn't exist, skips
3. Both checks pass: calls `FUN_00480f20(1)` to activate ADB (same function as 10-tap About toggle)
4. Restores `$v0 = $s1` (replaced instruction) and jumps to `0x52c334` (epilogue continuation)

### Custom View File (`hiby_app_sync.view`)

When App Sync is active, the USB page loads `hiby_app_sync.view` instead of `hiby_usb.view`.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x52c4f0` + `0x52c4f4` | `lui $a1, 0x94` + `addiu $a1, -0x24b0` | `j 0x41c120` + `nop` | Redirect to view selector |

**View selector trampoline** at `0x41c120` (~48 B):

1. Executes replaced instruction: `addu $s0, $a0, $v0`
2. Default: loads `"hiby_usb.view"` string pointer (original at `0x93db50`)
3. Checks BSS flag: if set, loads `"hiby_app_sync.view"` (at `0x41c160`)
4. Calls `strcat` (`0x8e6160`) with the selected filename
5. Jumps to `0x52c500`

**String** at `0x41c160`: `hiby_app_sync.view`

### Path-Builder Stabilization

Prevents the custom filename from causing a path validation failure inside `FUN_0052c4a0`.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x52c520` | conditional branch | `b 0x52c548` | Force branch to path copy |
| `0x52c524` | (conditional) | `move $s0, $zero` | Force success return value |

### Marker-File Helper

A reusable helper that checks the marker file. Called from the teardown gate and the stop callback.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x480f80` + `0x480f84` | `lui $v0, 0x97` + `lw $v0, ...` | `j 0x41c220` + `nop` | Redirect `FUN_00480f80` |

**Marker-file helper** at `0x41c220` (~32 B):

1. Calls `access("/usr/data/adb_usb_mode", 0)`
2. Returns the result (0 = file exists, non-zero = doesn't exist)

This replaces `FUN_00480f80` which originally read `DAT_00969d04` (cached ADB state). The helper reads the marker file directly, ensuring the USB mode decision always reflects the actual file state.

**Compatibility:** `FUN_00529860` (About page 10-tap ADB path) calls `FUN_00480f80` at `0x52990c`. This call is preserved and now goes through the marker-file helper, which is compatible because the About page flow also creates/removes the marker file.

### Disconnect Teardown Gate

Keeps the App Sync page visible when USB is disconnected (only in App Sync mode).

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x52bda0` + `0x52bda4` | `addiu $sp, $sp, -0x78` + `sw $s3, ...` | `j 0x41c260` + `nop` | Redirect teardown entry |

**Teardown gate** at `0x41c260` (~80 B):

1. Checks BSS flag: if 0, falls through to original teardown at `0x52bda8`
2. Calls marker-file helper (`0x41c220`): if file doesn't exist, falls through
3. Both checks pass (App Sync active + file exists): returns 0 early (skips teardown, page stays)
4. Otherwise: restores context and jumps to `0x52bda8` (original teardown code)

### Stop Button

Adds a clickable stop control to the App Sync page view.

#### Control Table Expansion

`FUN_0052c5a0` was modified to return a pointer to a new 2-entry control table instead of the original single-entry pointer.

| Address | Original | Patched | Description |
|---------|----------|---------|-------------|
| `0x52c5a8` | `lui $v0, 0x94` | `lui $v0, 0x42` | Table pointer high |
| `0x52c5ac` | `addiu $v0, -0x2340` | `addiu $v0, -0x3d00` | Table pointer low → `0x41c300` |
| `0x52c5b8` | `addiu $v0, $zero, 1` | `addiu $v0, $zero, 3` | Entry count: 1 → 3 |

**Control table** at `0x41c300` (2 active entries, each 96 bytes with inline name string + callback pointer at offset `+0x48`):

| Entry | Name (inline) | Callback |
|-------|--------------|----------|
| 0 | `usb_iv_back` | `0x52c480` (original back handler) |
| 1 | `app_sync_stop` | `0x41c2c0` (new stop callback) |

**Stop callback** at `0x41c2c0` (~60 B):

1. `system("rm -f /usr/data/adb_usb_mode")` — removes marker file
2. `jal 0x7f2e60` — re-evaluates marker file and refreshes BSS flag
3. `FUN_00480f20(0)` — stops ADB, restores mass storage
4. `FUN_0048a380(1)` — closes the App Sync page
5. Returns 0

---

## Required Configuration

### `set_functions.json`

Add `app_sync` to the `sys_set` section:

```json
{"app_sync":1}
```

### `sys_set.ini` (UTF-16LE with BOM)

Add the setting and sync page button label:

```
<app_sync>App Sync</app_sync>
<sync_stop>Stop Sync</sync_stop>
```

### View Files

Place `hiby_app_sync.view` in both theme directories:

- `/usr/share/resource/layout/theme1/hiby_app_sync.view`
- `/usr/share/resource/layout/theme2/hiby_app_sync.view`

The view must contain a clickable element named `app_sync_stop` for the stop button. Text for the stop button is read from the view's own text/ini binding (not hardcoded in the binary).

---

## Key Firmware Functions Referenced

| Address | Description |
|---------|-------------|
| `0x480f20` | ADB toggle: `param=1` calls `/usr/bin/adbon`, `param=0` calls `/usr/bin/adboff` |
| `0x480f80` | ADB state query (patched to check marker file instead of cached state) |
| `0x48a380` | Close current page (used by stop button) |
| `0x4af060` | Refresh a listview by name |
| `0x4e0ae0` | Get dispatch ID for a settings list item |
| `0x4e0be0` | Get toggle state for a settings list item |
| `0x8e6160` | `strcat()` |
| `0x8e6240` | `access()` |
| `0x8e7090` | `system()` |

---

## What's NOT Modified

- The original 10-tap ADB toggle on the About page (`FUN_00529860` / `FUN_00480f20`)
- `FUN_0047d760` (USB gadget handler) — completely untouched
- `FUN_004e0be0` (toggle state table reader) — body untouched, only callers redirected
- The toggle state table at `0x819590` — untouched
- The main event loop call at `0x48bbd0` — untouched
- All pre-existing patches: Sorting, DB Manager, Playlist, DB Manager Dialog
- Binary size: 5,598,244 bytes

---

## Other Patches in This Binary

This binary also contains the Sorting, DB Manager, Playlist, and DB Manager Dialog patches. For details see their respective README files.
