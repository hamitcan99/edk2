# CLAUDE.md — edk2 + LvglPkg Project Context

## Project Goal
Replace EDK2's native HII Form Browser (text-based UI) with an LVGL-based graphical renderer.
The approach is **Display Engine Replacement** — implement `EFI_DISPLAY_ENGINE_PROTOCOL` so that
`SetupBrowserDxe` continues to do all IFR parsing, conditional evaluation, and config routing,
while our code handles only the rendering layer via LVGL.

## Repository Structure
```
~/workspace/edk2/          ← edk2 fork (main repo, branch: dev)
├── LvglPkg/               ← LvglPkg submodule (forked from YangGangUEFI/LvglPkg)
│   ├── Library/LvglLib/   ← LVGL UEFI port (GOP flush, input handling)
│   ├── Application/       ← Demo applications
│   └── lvgl/              ← LVGL source (submodule)
├── OvmfPkg/               ← Modified: UsbMouseAbsolutePointerDxe added
└── Build/                 ← Build outputs (gitignored)
```

## Build Commands

### Environment Setup (run every new terminal session)
```bash
cd ~/workspace/edk2
source edksetup.sh
export PACKAGES_PATH=$HOME/workspace/edk2:$HOME/workspace/edk2/LvglPkg
```

### Build OVMF (UEFI firmware for QEMU)
```bash
build -a X64 -t GCC -b DEBUG -p OvmfPkg/OvmfPkgX64.dsc
```

### Build LvglPkg (demo applications)
```bash
build -a X64 -t GCC -b DEBUG -p LvglPkg/LvglPkg.dsc
```

### Aliases
```bash
alias build_edk2='build -a X64 -t GCC --buildtarget DEBUG -p OvmfPkg/OvmfPkgX64.dsc'
```

## QEMU Run Script
```bash
bash ~/workspace/edk2/qemu_lvgl.sh
```

### qemu_lvgl.sh content
```bash
#!/bin/bash
mkdir -p /tmp/efi_files
cp Build/Lvgl/DEBUG_GCC/X64/*.efi /tmp/efi_files/
cp Build/OvmfX64/DEBUG_GCC/FV/OVMF_VARS.fd /tmp/OVMF_VARS.fd

qemu-system-x86_64 \
  -machine q35 \
  -m 512M \
  -smp 2 \
  -drive if=pflash,format=raw,unit=0,readonly=on,file=Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd \
  -drive if=pflash,format=raw,unit=1,file=/tmp/OVMF_VARS.fd \
  -drive format=raw,file=fat:rw:/tmp/efi_files \
  -device qemu-xhci,id=xhci \
  -device usb-kbd,bus=xhci.0 \
  -device usb-mouse,bus=xhci.0 \
  -display gtk \
  -serial stdio 2>&1 | tee /tmp/qemu_log.txt
```

### Running in UEFI Shell
```
fs0:
LvglDemos.efi        # LVGL demo with keypad encoder
UefiDashboard.efi    # Dashboard demo
```

### Exit QEMU
`Ctrl+A` then `X`

## Key Architecture Decisions

### Current Stack
```
QEMU (x86_64, q35) → OVMF (EDK2) → UEFI Shell → LvglPkg .efi application
```

### Target Stack (end goal)
```
QEMU → OVMF → DXE phase → LvglDisplayEngineDxe (replaces DisplayEngineDxe)
                         → SetupBrowserDxe walks IFR, evaluates conditionals
                         → LvglDisplayEngineDxe renders via LVGL
```

### HII Build-Time Pipeline
```
UNI files
  │  (StrGather)
  ▼
STRING_TOKEN integers (.h) + string binary data
  │
VFR files  ←─ C preprocessor pulls in STRING_TOKEN integers here
  │  (VfrCompile)
  ▼
IFR byte array (.c file)
  │
  │  compiled + linked into driver .efi
  ▼
Driver EntryPoint calls HiiAddPackages()
  │
  ▼
EFI_HII_DATABASE_PROTOCOL  ← Forms package + Strings package stored here
```

Key point: by the time the C compiler sees anything, VFR and UNI are fully
reduced to a `.h` of `#define` integers and a `.c` of a raw byte array.
The C compiler has no idea they came from anything special.

### HII Runtime Flow
```
BDS (user presses F2/DEL)
  │  calls SendForm()
  ▼
EFI_FORM_BROWSER2_PROTOCOL  ← entry point, called by BDS
(SetupBrowserDxe)             - walks IFR opcodes linearly
                              - evaluates suppressif/grayoutif expressions
                              - manages navigation state, save/discard/reset
                              - calls driver for current values + saves
  │
  │  needs current values / user saves
  ▼
EFI_HII_CONFIG_ACCESS_PROTOCOL   ← your driver implements this
  ExtractConfig()   ← browser asks: what are the current values?
  RouteConfig()     ← browser says: user saved, write these values
  Callback()        ← browser says: user changed question X interactively
  │
  │  needs to draw — THIS IS THE SEAM WE REPLACE
  ▼
EFI_DISPLAY_ENGINE_PROTOCOL      ← WE REPLACE THIS
(DisplayEngineDxe → LvglDisplayEngineDxe)
  FormDisplay()     ← receives FORM_DISPLAY_ENGINE_FORM, already parsed
  ExitDisplay()     ← browser is closing
  ConfirmDataChange() ← show save/discard popup
  │
  ▼
LVGL
  │  flush callback
  ▼
EFI_GRAPHICS_OUTPUT_PROTOCOL     ← pixels on screen
```

### Why We Replace DisplayEngineDxe — Not SetupBrowserDxe

`EFI_DISPLAY_ENGINE_PROTOCOL` is the exact seam the EDK2 architects provided
for this purpose. By the time `FormDisplay()` is called, SetupBrowserDxe has
already done all the hard work:

- IFR opcode walking
- Scope stack management
- Expression bytecode evaluation (suppressif / grayoutif / disableif)
- Config string extraction via EFI_HII_CONFIG_ROUTING_PROTOCOL
- String ID → text resolution via HiiGetString()
- Navigation state

`FormDisplay()` receives a `FORM_DISPLAY_ENGINE_FORM` containing a clean linked
list of `FORM_DISPLAY_ENGINE_STATEMENT` structs — one per visible question, with
current value, prompt string, help string, options list, and grayed/locked flags
already resolved.

We do NOT reimplement IFR parsing. SetupBrowserDxe handles it all.

### Statement → LVGL Widget Mapping

This is the core of our `FormDisplay()` implementation:

| FORM_DISPLAY_ENGINE_STATEMENT OpCode | LVGL Widget            |
|--------------------------------------|------------------------|
| EFI_IFR_SUBTITLE                     | lv_label_create()      |
| EFI_IFR_CHECKBOX                     | lv_checkbox_create()   |
| EFI_IFR_NUMERIC                      | lv_spinbox_create()    |
| EFI_IFR_ONE_OF                       | lv_dropdown_create()   |
| EFI_IFR_ORDERED_LIST                 | lv_list_create()       |
| EFI_IFR_STRING                       | lv_textarea_create()   |
| EFI_IFR_PASSWORD                     | lv_textarea_create() + password mode |
| EFI_IFR_REF (goto)                   | lv_btn_create()        |
| EFI_IFR_ACTION                       | lv_btn_create()        |

### LVGL on UEFI — Three Requirements

```c
// 1. Display flush — copy LVGL framebuffer to GOP
void lvgl_flush_cb(lv_disp_drv_t *drv, const lv_area_t *area, lv_color_t *buf)
{
    // gop->Blt() to copy buf to framebuffer at area coordinates
    lv_disp_flush_ready(drv);
}

// 2. Input — translate EFI input protocols to LVGL input device
void lvgl_keyboard_cb(lv_indev_drv_t *drv, lv_indev_data_t *data)
{
    // read EFI_SIMPLE_TEXT_INPUT_EX_PROTOCOL
    // translate EFI key codes → LVGL key codes
}

// 3. Tick — LVGL needs a millisecond tick source
// call lv_tick_inc(ms_elapsed) via EFI_TIMER_EVENT or gBS->Stall()
```

### Platform DSC Change (the only platform file to touch)

```ini
# OvmfPkg/OvmfPkgX64.dsc — remove this:
MdeModulePkg/Universal/DisplayEngineDxe/DisplayEngineDxe.inf

# Add this:
LvglPkg/LvglDisplayEngineDxe/LvglDisplayEngineDxe.inf
```

Everything above that line — UNI, VFR, IFR, HII database, SetupBrowserDxe,
driver EFI_HII_CONFIG_ACCESS_PROTOCOL — is completely untouched.

## Mouse/Input Status
- **AbsolutePointer + Mouse Wheel**: ✅ Working with QEMU `usb-mouse` via
  `UsbMouseAbsolutePointerDxe`. A single custom `mouse_read` callback owns the one
  `GetState()` call per frame, handling X/Y rescaling, left-button state, and Z-axis
  wheel accumulation together. This is race-free: `GetState()` is single-consumer (the
  USB driver clears `StateChanged` on each read), so a separate wheel poller would steal
  cursor events. PR#17 lazy-binding via `RegisterProtocolNotify` preserved.
  Note: synthesized absolute — cursor moves proportionally but does not mirror the host
  pointer 1:1. Wheel ratchet: 8 raw Z counts per scroll step, 40 px per detent.
- **SimplePointer**: Removed — no longer used. The QEMU setup only produces
  `EFI_ABSOLUTE_POINTER_PROTOCOL` (UsbMouseDxe intentionally excluded).
  The old SimplePointer code path in `GetXYZ`/`EfiMouseInit` was dead code and has been
  removed.
- **Keyboard**: ✅ Working via custom `keypad_read` callback in `lv_port_indev.c`.
  Uses `EFI_SIMPLE_TEXT_INPUT_EX_PROTOCOL` directly: returns `PRESSED` while the EFI
  buffer holds a key, `RELEASED` when empty. This is the correct model for UEFI's
  press-only (no key-up) protocol and lets LVGL's `long_press_repeat` throttle control
  navigation rate. The built-in `lv_uefi_simple_text_input_indev` was tried but caused
  runaway navigation (PRESSED+RELEASED per keystroke in the same tick bypasses
  LVGL's rate limiting).

### How AbsolutePointer is wired
- `OvmfPkgX64.fdf` and `OvmfPkg/Include/Dsc/UsbComponents.dsc.inc` include
  `UsbMouseAbsolutePointerDxe` only — `UsbMouseDxe` is intentionally NOT included.
- Reason: EDK2's `CoreConnectSingleController` sorts driver bindings by `Version`
  field highest→lowest (`MdeModulePkg/Core/Dxe/Hand/DriverSupport.c:607-618`).
  `UsbMouseDxe` has `Version=0xa`, `UsbMouseAbsolutePointerDxe` has `Version=0x1`,
  so when both are present `UsbMouseDxe` binds first and locks UsbIo BY_DRIVER,
  blocking AbsolutePointer. Removing `UsbMouseDxe` from the firmware lets the
  AbsolutePointer driver bind unopposed.
- `qemu_lvgl.sh` must use `-device usb-mouse` (Boot/Mouse, Subclass=1/Protocol=2)
  — NOT `usb-tablet` (Subclass=0/Protocol=0), which neither edk2 mouse driver binds.

### True 1:1 absolute tracking (not yet implemented)
For real host-cursor mirroring with `usb-tablet`, a custom HID-class AbsolutePointer
driver in `LvglPkg/` would be needed (parses usb-tablet's 16-bit absolute X/Y
report descriptor, range 0..32767). Out of scope for now — synthesized absolute is
sufficient for current LVGL development.

### Mouse Input Design (lv_port_indev.c)
Uses `ConsoleInHandle` to reach the ConSplitter aggregate for both mouse and wheel.
ConSplitter installs `EFI_ABSOLUTE_POINTER_PROTOCOL` on its VirtualHandle
(= `gST->ConsoleInHandle`) at driver entry, then aggregates all physical devices as
they bind. `GetState()` iterates the internal device list and rescales coordinates to a
virtual range. This is the correct UEFI pattern — no `LocateHandleBuffer` needed.

**Single-consumer constraint**: `GetState()` clears the physical device's `StateChanged`
flag on each read (see `GetMouseAbsolutePointerState`, `UsbMouseAbsolutePointer.c:911`).
Therefore mouse X/Y AND wheel Z must be read in the same `GetState()` call. The custom
`mouse_read` callback does this. The built-in `lv_uefi_absolute_pointer_indev` was
reverted because it discards `CurrentZ` and there is no race-free way to add a separate
Z-poller. The built-in **display** backend is still used (`lv_uefi_display_create`).

## Key Source Files
```
# LVGL UEFI port
LvglPkg/Library/LvglLib/LvglLib.c           ← Init/deinit, main loop
LvglPkg/Library/LvglLib/lv_port_indev.c     ← Mouse (custom: X/Y/buttons/wheel in one GetState) + keyboard (custom)
LvglPkg/Library/LvglLib/lv_conf.h           ← LV_USE_UEFI=1, LV_USE_UEFI_INCLUDE

# LVGL built-in UEFI driver (vendored, read-only)
LvglPkg/Library/LvglLib/lvgl/src/drivers/uefi/lv_uefi_display.c      ← GOP flush (replaces old lv_uefi_display.c)
LvglPkg/Library/LvglLib/lvgl/src/drivers/uefi/lv_uefi_indev_pointer.c ← absolute pointer indev
LvglPkg/Library/LvglLib/lvgl/src/drivers/uefi/lv_uefi_indev_keyboard.c ← NOT compiled (removed from LvglLib.inf; keyboard uses custom keypad_read)
LvglPkg/Library/LvglLib/lvgl/src/drivers/uefi/lv_uefi_indev_pointer.c  ← NOT compiled (removed; pointer uses custom mouse_read)
LvglPkg/Library/LvglLib/lvgl/src/drivers/uefi/lv_uefi_indev_touch.c    ← NOT compiled (removed; discards Z axis, replaced by custom mouse_read)
LvglPkg/Library/LvglLib/lvgl/src/drivers/uefi/lv_uefi_edk2.h         ← EDK2 framework binding

# Display Engine — the seam we implement
MdeModulePkg/Include/Protocol/DisplayProtocol.h         ← EFI_DISPLAY_ENGINE_PROTOCOL
                                                          FORM_DISPLAY_ENGINE_FORM
                                                          FORM_DISPLAY_ENGINE_STATEMENT
MdeModulePkg/Universal/DisplayEngineDxe/FormDisplay.c  ← reference implementation to replace
MdeModulePkg/Universal/DisplayEngineDxe/FormDisplay.h  ← internal helpers reference

# SetupBrowserDxe — do NOT modify, just let it call us
MdeModulePkg/Universal/SetupBrowserDxe/           ← IFR walker, expression evaluator

# HII protocols — read-only reference
MdePkg/Include/Uefi/UefiInternalFormRepresentation.h  ← All IFR opcodes + structures
MdePkg/Include/Protocol/HiiDatabase.h                 ← EFI_HII_DATABASE_PROTOCOL
MdePkg/Include/Protocol/FormBrowser2.h                ← EFI_FORM_BROWSER2_PROTOCOL
                                                         (what BDS calls — not our concern)
MdePkg/Include/Protocol/HiiConfigAccess.h             ← EFI_HII_CONFIG_ACCESS_PROTOCOL
                                                         (driver-side, not our concern)

# Reference VFR/HII driver
MdeModulePkg/Universal/DriverSampleDxe/          ← Best VFR/HII example

# Theme / chrome
LvglPkg/Include/LvglTheme.h                      ← User-customizable color palette and fonts
LvglPkg/LvglDisplayEngineDxe/LvglAptioChrome.c  ← Aptio-style frame: title bar, help pane,
                                                   footer hotkey bar (BuildFooter walks HotKeyListHead,
                                                   renders HelpString chips via AddHotKeyChip)
LvglPkg/LvglDisplayEngineDxe/AptioWallpaper.c   ← Background wallpaper
```

## LvglDisplayEngineDxe Module (created)
```
LvglPkg/LvglDisplayEngineDxe/
  LvglDisplayEngineDxe.c     ← produces/installs EFI_DISPLAY_ENGINE_PROTOCOL
  LvglDisplayEngineDxe.inf   ← module INF
  LvglFormRenderer.c         ← FormDisplay(): FORM_DISPLAY_ENGINE_FORM → LVGL widgets
  LvglFormRenderer.h         ← Renderer types and API
  LvglAptioChrome.c          ← Aptio-style chrome: title bar, help pane, footer hotkey bar
  LvglAptioChrome.h          ← Chrome API
  AptioWallpaper.c           ← Background wallpaper rendering

LvglPkg/Include/
  LvglTheme.h                ← User-customizable color palette and font table
```

## Toolchain Info
- OS: Ubuntu 24.04, Linux 6.17
- GCC: 13.3.0
- Toolchain tag: `GCC` (GCC5 was removed from EDK2 in late 2024)
- QEMU: qemu-system-x86_64
- Architecture: X64

## Git Workflow
```
edk2 fork:
  branch: dev  ← active development
  branch: master ← tracks upstream tianocore/edk2

LvglPkg fork (hamitcan99/LvglPkg):
  branch: master         ← active development
  branch: pr/gcc-fixes   ← PR #14: GCC build fixes + README
  branch: pr/mouse-wheel ← PR #15: mouse wheel support
  branch: pr/display-engine ← PR #16: LvglDisplayEngineDxe
  branch: pr/mouse-notify ← PR #17: lazy mouse indev via protocol notification
  upstream: YangGangUEFI/LvglPkg

# Sync upstream edk2
git checkout master
git fetch upstream
git merge upstream/master
git checkout dev
git rebase master
```

## Upstream PRs to YangGangUEFI/LvglPkg
- **PR #14** (pr/gcc-fixes): GCC build fixes + README update — `EFIAPI` fixes, unused variable, GCC build docs
- **PR #15** (pr/mouse-wheel): Mouse wheel support — Z-axis tracking via ConsoleInHandle, ratchet threshold
- **PR #16** (pr/display-engine): LvglDisplayEngineDxe — LVGL-based HII form renderer + screenshots
- **PR #17** (pr/mouse-notify): Lazy mouse indev creation via `RegisterProtocolNotify` — fixes unusable mouse when LvglLib is consumed by a DXE_DRIVER (USB not connected at constructor time)
- **PR #13** (closed): Original combined PR, split into #14/#15/#16 per maintainer request

## hamitcan99/LvglPkg PRs (internal feature branches)
- **PR #2** (feat/lvgl-builtin-display): Switch display backend to LVGL's built-in UEFI driver — `LV_USE_UEFI=1`, delete project's `lv_uefi_display.c`, use `lv_uefi_display_create()`
- **PR #3** (feat/lvgl-builtin-input): Switch pointer to built-in `lv_uefi_absolute_pointer_indev`; restore custom `keypad_read` for keyboard (built-in keyboard caused runaway navigation)
- **PR #4** (feat/lvgl-builtin-cleanup): Re-port mouse-wheel scrolling — revert pointer to custom `mouse_read` for race-free single-`GetState()` X/Y+wheel; remove dead built-in indev sources from LvglLib.inf; update docs

## LVGL Built-in UEFI Driver Migration Plan

LVGL v9.5 ships a built-in UEFI backend at `lvgl/src/drivers/uefi/`.  The goal is to
replace the project's ~750-line hand-written glue (custom GOP flush + custom pointer/
keyboard polling) with the upstream driver, keeping only the pieces the built-in does
not provide.

### Branch 1 — `feat/lvgl-builtin-display` ✅ DONE (hamitcan99/LvglPkg PR #2)
**Goal**: swap the GOP display backend to the built-in driver.

Changes:
- `lv_conf.h`: `LV_USE_UEFI 0` → `1`, `LV_USE_UEFI_INCLUDE` → `"lv_uefi_edk2.h"`
- `LvglLib.c`: `lv_uefi_init(gImageHandle, gST)` before `lv_init()`; replace
  `lv_uefi_disp_create()` with `lv_uefi_display_get_any()` + `lv_uefi_display_create()`
- Delete `Library/LvglLib/lv_uefi_display.c` (83-line custom GOP flush, now replaced)
- `LvglLib.inf`: remove `lv_uefi_display.c` from `[Sources]`; built-in files at
  `lvgl/src/drivers/uefi/` now compile because `LV_USE_UEFI=1`

Key insight: `lv_uefi_display_create()` uses GOP only — no EDID at runtime.
`LV_USE_UEFI_INCLUDE "lv_uefi_edk2.h"` resolves from `lv_uefi.h`'s own directory.

### Branch 2 — `feat/lvgl-builtin-input` ✅ DONE (hamitcan99/LvglPkg PR #3)
**Goal**: replace pointer glue with built-in `lv_uefi_absolute_pointer_indev`;
fix keyboard to not cause runaway navigation.

Changes:
- **Pointer**: `lv_uefi_absolute_pointer_indev_create(&res)` + `add_handle(ConsoleInHandle)`
  replaces the custom `LVGL_UEFI_MOUSE` struct, `GetXYZ`, `EfiMouseInit`, `mouse_read`
- **PR#17 lazy-binding**: kept — `RegisterProtocolNotify(gEfiAbsolutePointerProtocolGuid)`
  retries `add_handle` until ConSplitter's Mode becomes valid after USB binds
- **Keyboard**: the built-in `lv_uefi_simple_text_input_indev` was tried and reverted.
  Root cause: it queues PRESSED+RELEASED for each EFI keystroke via `continue_reading`,
  delivering both in the same LVGL tick. LVGL's `indev_keypad_proc` never sees the key
  as held, so `long_press_repeat` throttle never fires. EFI auto-repeat (~33 Hz) then
  caused uncontrolled navigation. Custom `keypad_read` restored: returns `PRESSED` while
  EFI buffer is non-empty, `RELEASED` when empty — correct model for UEFI's press-only
  protocol.
- `lv_uefi_keypad_drain()` preserved (called from `LvglFormRenderer.c`)
- `LV_KEY_F1..F12` defines preserved in `lv_port_indev.h` and `Include/Library/LvglLib.h`

Known temporary regressions vs pre-migration:
- ~~Mouse wheel not re-ported (built-in pointer indev discards Z axis)~~ **Fixed in Branch 3**

### Branch 3 — `feat/lvgl-builtin-cleanup` ✅ DONE (hamitcan99/LvglPkg PR #4)
**Goal**: re-port mouse wheel, remove dead built-in indev sources, update docs.

Changes:
- **Mouse wheel**: reverted pointer to custom `mouse_read` — single `GetState()` call
  per frame handles X/Y/buttons AND `CurrentZ` wheel accumulation. Race-free by design
  (see "Single-consumer constraint" in Mouse Input Design). `find_scrollable_at_point`
  + ratchet (`LVGL_WHEEL_COUNTS_PER_DETENT=8`, 40 px/detent) re-ported from pre-Branch-2
  history. Dead `SimplePointer` code path removed.
- **LvglLib.inf**: removed `lv_uefi_indev_keyboard.c`, `lv_uefi_indev_pointer.c`,
  `lv_uefi_indev_touch.c` from `[Sources]` — no longer compiled.
- Docs updated throughout.

## Current Status
- LvglDisplayEngineDxe skeleton: **done** — builds, installs protocol, wired into DSC/FDF
- LvglLib.inf fix: **done** — removed `UefiApplicationEntryPoint`, consumable by DXE_DRIVER
- FormDisplay() initial implementation: **done** — walks StatementListHead, creates LVGL widgets,
  runs event loop, returns user action to browser
- LVGL-based form UI renders on screen in QEMU when entering Setup
- Mouse input: **done** — custom `mouse_read` callback: single `GetState()` call for
  X/Y/buttons + Z-axis wheel accumulation; PR#17 `RegisterProtocolNotify` lazy-binding
  preserved so mouse works when USB binds during BDS
- Mouse wheel scrolling: **done** — `find_scrollable_at_point` + ratchet
  (`LVGL_WHEEL_COUNTS_PER_DETENT=8`, 40 px/detent) via `lv_obj_scroll_by_bounded`;
  race-free because `GetState()` is called only once per frame in `mouse_read`
- Keyboard navigation: **done** — custom `keypad_read` callback (PRESSED while EFI buffer
  non-empty, RELEASED when empty) + `OnNavKey`/`AddToNavGroup` in `LvglFormRenderer.c`
  give UP/DOWN focus, ESC form-exit, ENTER-toggles-editing for spinbox/dropdown/textarea
- String field commit: **done** — `OnStringReady` allocates a zero-filled pool buffer
  (`AllocateZeroPool(CurrentValue.BufferLen)`) and calls `HiiSetString` to create a
  fresh string token. SetupBrowserDxe's `FreePool(InputValue.Buffer)` no longer asserts,
  and `CopyMem(BufferValue, Buffer, BufferLen)` fills storage correctly.
- LVGL built-in display backend: **done** — `LV_USE_UEFI=1`, built-in GOP flush via
  `lv_uefi_display_create()`, deleted project's custom `lv_uefi_display.c` (PR #2)
- LVGL built-in pointer indev (Branch 2): adopted then reverted in Branch 3 — built-in
  `lv_uefi_absolute_pointer_indev` discards `CurrentZ` (wheel axis), and a separate
  Z-poller would race `GetState()`. Custom `mouse_read` reinstated (PR #3 → PR #4).
- F-key hotkey wiring: **done** — `HandleFunctionKey()` (`LvglFormRenderer.c:874-930`)
  walks `HotKeyListHead`, maps `LV_KEY_Fn → SCAN_Fn`, shows confirm popup for
  SUBMIT/DEFAULT actions, exits immediately for others.
- Theme pass: **done** — NovaCore palette, user-customizable via `LvglPkg/Include/LvglTheme.h`.
- Aptio-style chrome: **done** — `LvglAptioChrome.c/.h` (title bar, help pane, dynamic
  footer hotkey bar with registered key labels); `AptioWallpaper.c` (background).
- Mouse hover highlight: **done** — focused form row is highlighted on hover.
- Bottom-docked on-screen keyboard: **done** — appears when a text/password field is focused.

## Known Bugs
1. ~~**Arrow keys (UP/DOWN/LEFT/RIGHT) not working**~~ — **FIXED**. `OnNavKey`
   in `LvglFormRenderer.c` is attached to every widget via `AddToNavGroup`
   and converts UP→`lv_group_focus_prev`, DOWN→`lv_group_focus_next`, and
   ENTER→`lv_group_set_editing(true)` for spinbox/dropdown/textarea. In
   editing mode LEFT/RIGHT drive LVGL's encoder emulation to adjust the
   value; ESC exits editing.
2. ~~**ESC key not working**~~ — **FIXED**. ESC is now handled in
   `OnNavKey` (per-widget LV_EVENT_KEY callback), which does receive key
   events via `lv_group_send_data`. Non-editing ESC sets
   `BROWSER_ACTION_FORM_EXIT`; editing ESC cancels editing.
3. ~~**`EFI_IFR_ORDERED_LIST_OP` not rendered**~~ — **FIXED**.
   `CreateOrderedListWidget` in `LvglFormRenderer.c` walks
   `Statement->CurrentValue.Buffer` using local `GetArrayData`/`SetArrayData`
   helpers (ported from `MdeModulePkg/Universal/DisplayEngineDxe/ProcessOptions.c`),
   renders one row per active entry with Up/Down buttons, and on click emits
   the reordered buffer via `USER_INPUT.InputValue.Buffer` + exit so
   SetupBrowserDxe re-invokes `FormDisplay()` with the new state. Option
   value lookup reads `Option->OptionOpCode->Value` at its native width
   (u8/u16/u32/u64 per `ValueType` = first option's `OpCode->Type`) — a
   raw `.u64` read would over-read past the IFR-sized value into
   neighboring bytes and break the label match.
4. ~~**ASSERT on Enter in string textarea**~~ — **FIXED**. `OnStringReady` in
   `LvglFormRenderer.c` now allocates a real pool buffer via
   `AllocateZeroPool(CurrentValue.BufferLen)` and sets `InputValue.Buffer`,
   `InputValue.BufferLen`, and `InputValue.Value.string` (via `HiiSetString`)
   to satisfy SetupBrowserDxe's `ProcessUserInput` contract. Fallback to
   `EFI_IFR_STRING.MaxSize * sizeof(CHAR16)` when `CurrentValue.BufferLen == 0`.
5. **"Submit fail" on some HII drivers (e.g. iPXE)** — **NOT our bug**. Investigated:
   the original `DisplayEngineDxe` exhibits the same behavior. The failure is in the
   driver's own `RouteConfig()` implementation rejecting the config string. There is
   nothing to fix in the display engine for this case.
6. ~~**Fonts and colors need improvement**~~ — **ADDRESSED**. The placeholder dark
   theme (0x1A1A2E / 0x16213E) was replaced by the NovaCore palette (Aptio-style
   chrome) and a user-customizable color/font table at `LvglPkg/Include/LvglTheme.h`.
   Styling remains iterative, but the placeholder concern is resolved.
7. ~~**Function-key hotkeys not wired**~~ — **FIXED**. `HandleFunctionKey()` in
   `LvglFormRenderer.c:874-930` walks `FormData->HotKeyListHead`, maps
   `LV_KEY_Fn → SCAN_Fn` (reverse of the `keypad_read` translation), and matches by
   `ScanCode + UnicodeChar == CHAR_NULL`. Actions that require confirmation
   (`BROWSER_ACTION_SUBMIT | BROWSER_ACTION_DEFAULT`) show a confirm popup before
   exiting; others (Reset/Exit) set `UserInput->Action` and `ExitRequested` immediately.
   The function is invoked from both `OnNavKey` (`:1204`, focused widget) and
   `OnIndevFallbackKey` (`:1009`, global fallback when nothing focusable is focused).
   The footer hotkey bar (`LvglAptioChrome.c:269-337`) surfaces each registered
   hotkey's `HelpString` as a labelled chip so the user can see F9/F10 hints.
8. ~~**Mouse-wheel scrolling not ported**~~ — **FIXED** (Branch 3 / PR #4).
   Custom `mouse_read` reads `CurrentZ` in the same `GetState()` call used for X/Y,
   accumulates delta vs `mLastAbsZ`, ratchets via `LVGL_WHEEL_COUNTS_PER_DETENT=8`,
   and calls `lv_obj_scroll_by_bounded` on the scrollable ancestor under the cursor
   via `find_scrollable_at_point`. Built-in `lv_uefi_absolute_pointer_indev` was
   reverted because it discards `CurrentZ` and a separate Z-poller would race
   `GetState()` (single-consumer, clears `StateChanged` on each read).
9. **Help pane does not follow mouse hover** — `OnFocusUpdateHelp` in
   `LvglFormRenderer.c` only fires on `LV_EVENT_FOCUSED` (keyboard focus). Hovering
   a row updates the visual highlight via `OnRowHoverChange`/`BindRowHover`, but the
   right-side help pane text stays on the last keyboard-focused item. Planned fix:
   register an additional callback on `LV_EVENT_HOVER_OVER` inside `AddToNavGroup`
   (where `Ctx` is available) that calls `GetHelpUtf8` + `AptioSetHelpText`; on
   `LV_EVENT_HOVER_LEAVE` restore the keyboard-focused item's help by tracking the
   last focused `LVGL_STATEMENT_CONTEXT *` in a module-level static.
10. ~~**UP/DOWN doesn't cycle a focused dropdown**~~ — **FIXED**. Dropdown keyboard
    navigation now follows the standard BIOS-setup model: UP/DOWN on a focused
    (non-editing) dropdown moves between form rows as normal. Press ENTER to enter
    editing mode; now UP/DOWN cycles the dropdown's options **locally** with wrap-around
    (no commit, no form rebuild). Press ENTER again to confirm and commit once; ESC
    reverts the selection to what it was when editing began and exits editing without
    committing; moving focus away (mouse click, etc.) while editing commits once via
    `OnDropdownDefocused`. Implemented in `LvglFormRenderer.c`: two module statics
    (`mEditingDropdown`, `mEditingDropdownOrigSel`), `OnDropdownDefocused` handler,
    modified `OnIndevFallbackKey` (cycles only when `lv_group_get_editing`, no
    `LV_EVENT_VALUE_CHANGED`), modified `OnNavKey` (ENTER-confirm dispatches
    `LV_EVENT_VALUE_CHANGED`; ESC reverts via `mEditingDropdownOrigSel`; ENTER-to-edit
    snapshots the original selection). Note: commit `e502755` implemented the inverse
    behavior (cycle-on-focus without editing, commit-per-keystroke) and is superseded
    by this fix.

## Next Steps

### Branch 3 — `feat/lvgl-builtin-cleanup` ✅ DONE (PR #4)
All tasks complete: mouse wheel re-ported, dead built-in indev sources removed, docs
updated.

### Post-Branch-3 work ✅ DONE
1. ~~**Wire F-key hotkeys**~~ — **DONE**. `HandleFunctionKey()` in `LvglFormRenderer.c`
   walks `HotKeyListHead`, matches scan codes, returns `BROWSER_ACTION_*` with confirm
   popup for destructive actions. Footer hotkey bar surfaces registered key hints.
2. ~~**Theme/styling pass**~~ — **DONE**. User-customizable palette via
   `LvglPkg/Include/LvglTheme.h`; Aptio-style chrome with header, help pane, and
   dynamic hotkey footer (`LvglAptioChrome.c`); background wallpaper (`AptioWallpaper.c`);
   mouse hover highlight for form rows; bottom-docked on-screen keyboard.

### Next up
1. **End-to-end test** — build OVMF + LvglPkg, run in QEMU, and verify form navigation,
   value changes, save/discard flow, and F9 (Load Defaults) / F10 (Save) hotkeys.
2. **True 1:1 absolute mouse tracking** — out of scope for now; synthesized absolute is
   sufficient. Would require a custom HID-class AbsolutePointer driver for `usb-tablet`.
3. **Help pane follows mouse hover** — see Known Bug #9. Extend `AddToNavGroup` with
   `LV_EVENT_HOVER_OVER`/`LV_EVENT_HOVER_LEAVE` callbacks and a static to track the
   last focused `LVGL_STATEMENT_CONTEXT`.
4. ~~**UP/DOWN cycles a focused dropdown**~~ — **DONE**. See Known Bug #10 above.