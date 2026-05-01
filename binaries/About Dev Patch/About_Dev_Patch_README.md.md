# HiBy R3 Pro II — About Dev GitHub QR Patch

 SHA256: `6937b83454ae0d8363b5b3e3528494f6a0cf60ec6cfa1dcec48d92673512c744` 

---

## Overview

This patch adds a GitHub social button (`about_dev_github`) to the About page and wires it into the existing QR-dialog pipeline used by the other social buttons.

When the user taps `about_dev_github` the firmware opens the `url_qrcode` dialog, selects QR image slot index `3`, and displays `img_path_3` from `url_qrcode.dlg`.

---

## Firmware Architecture — About Social Buttons

The About page uses a static control table at VA `0x0093d62c` with fixed-size `0x60`-byte entries. Each entry contains a widget name string (offset `0x00`) and a callback pointer (offset `0x48`).

**Pre-patch social entries (entry indices 5–8):**

| Index | Name | Callback |
|-------|------|----------|
| 5 | `about_dev_iv_facebook` | `0x00529f80` |
| 6 | `about_dev_iv_wechat` | `0x00529f40` |
| 7 | `about_dev_iv_microblog` | `0x00529ea0` |
| 8 | `about_dev_iv_post_bar` | `0x00529fc0` |

All four callbacks dispatch to `FUN_0052bd20`, which opens the `url_qrcode` dialog and stores a QR image index in `DAT_0096c540` for the dialog loader.

---

## Patch Details

### 1) Control-table entry repurpose

Instead of expanding the table, the existing entry 8 (`about_dev_iv_post_bar`) was repurposed for GitHub. No table size change.

**Entry name patch:**

| Field | Value |
|-------|-------|
| VA | `0x0093d92c` |
| File offset | `0x52d92c` |
| Old | `about_dev_iv_post_bar` |
| New | `about_dev_github` |

**Callback pointer patch:**

| Field | Value |
|-------|-------|
| VA | `0x0093d974` |
| File offset | `0x52d974` |
| Old | `0x00529fc0` |
| New | `0x0041c3c0` |

All other entries in the table remain unchanged.

### 2) Callback stub in code cave

| Field | Value |
|-------|-------|
| VA | `0x0041c3c0` |
| File offset | `0x01c3c0` |
| Size | 56 bytes (14 words) |
| Prior contents | Zero-filled |

The stub validates the event context (same null-check pattern as the existing social callbacks), calls `FUN_0052bd20` with QR index `3`, and returns handled (`v0 = 1`).

**Disassembly:**

```mips
0x41c3c0:  beqz   $a0, 0x41c3f0       # null-check event ptr
0x41c3c4:  lw     $a1, 0x50($a1)       # (branch delay) load context field
0x41c3c8:  beqz   $a1, 0x41c3f0       # null-check context
0x41c3cc:  nop
0x41c3d0:  addiu  $sp, $sp, -0x20     # open stack frame
0x41c3d4:  sw     $ra, 0x1c($sp)      # save return address
0x41c3d8:  jal    0x52bd20            # call FUN_0052bd20
0x41c3dc:  addiu  $a2, $zero, 0x3     # (branch delay) arg: QR index = 3
0x41c3e0:  lw     $ra, 0x1c($sp)      # restore return address
0x41c3e4:  addiu  $v0, $zero, 0x1     # return 1 (handled)
0x41c3e8:  jr     $ra                 # return
0x41c3ec:  addiu  $sp, $sp, 0x20      # (branch delay) close stack frame
0x41c3f0:  jr     $ra                 # early-exit path
0x41c3f4:  addiu  $v0, $zero, 0x1     # (branch delay) return 1
```

**Raw words:**

```
1080000b 8ca50050 10a00009 00000000
27bdffe0 afbf001c 0c14af48 24060003
8fbf001c 24020001 03e00008 27bd0020
03e00008 24020001
```

**Complete slot map:**

| Slot | Path |
|------|------|
| `img_path_0` | `about_dev\facebook_qrcode.png` |
| `img_path_1` | `about_dev\wechat_qrcode.png` |
| `img_path_2` | `about_dev\weibo_qrcode.png` |
| `img_path_3` | `about_dev\github_qrcode.png` *(new)* |

---

## Execution Flow

```
Tap about_dev_github
  → control table lookup (base 0x93d62c + 8×0x60 = entry at 0x93d92c)
  → callback 0x41c3c0
  → FUN_0052bd20(..., index=3)
  → open url_qrcode dialog
  → dialog loader selects img_path_3
  → displays about_dev\github_qrcode.png
```

---

## Binary Patch Summary

Three regions of `hiby_player` are modified:

| File offset | VA | Size | Description |
|-------------|------|------|-------------|
| `0x52d92c` | `0x0093d92c` | 17 B | Entry name → `about_dev_github` |
| `0x52d974` | `0x0093d974` | 4 B | Callback pointer → `0x0041c3c0` |
| `0x01c3c0`–`0x01c3f7` | `0x0041c3c0`–`0x0041c3f7` | 56 B | Injected callback stub (was zero-filled) |

---

## Required Resource Files

These must exist in the active theme resource pack:

| File | Purpose |
|------|---------|
| `about_dev/github.png` | About page icon |
| `about_dev/github_qrcode.png` | QR image loaded by `img_path_3` |

---

## Validation Checklist

1. About page shows the GitHub icon (`about_dev_github`).
2. Tap the GitHub icon.
3. The URL QR dialog appears.
4. The displayed image matches `about_dev\github_qrcode.png`.


This patched binary also contains the follwing patches:

- Sorting Patch
- DB Manager Patch
- Playlist Patch
- DB Manager Dialog Patch
- Sync Patch with increased binary size
