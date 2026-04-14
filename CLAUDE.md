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
- **AbsolutePointer**: ✅ Working with QEMU `usb-mouse` via `UsbMouseAbsolutePointerDxe`.
  Note: this is *synthesized* absolute (the driver accumulates Boot Mouse relative
  deltas internally) — not true 1:1 host→guest tracking. Cursor moves proportionally
  to host motion but does not directly mirror the QEMU window's host pointer.
- **SimplePointer**: Available as fallback in `lv_port_indev.c` if AbsolutePointer
  is absent, but currently unused.
- **Keyboard**: ✅ Working via `EFI_SIMPLE_TEXT_INPUT_EX_PROTOCOL`

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
Uses `ConsoleInHandle` to get pointer protocols via ConSplitter aggregate.
ConSplitter installs `EFI_ABSOLUTE_POINTER_PROTOCOL` on its VirtualHandle
(= `gST->ConsoleInHandle`) at driver entry, then aggregates all physical
devices as they bind. `GetState()` iterates the internal device list and
rescales coordinates to a virtual range. This is the correct UEFI pattern
— no need to use `LocateHandleBuffer` to find individual devices.

## Key Source Files
```
# LVGL UEFI port (existing)
LvglPkg/Library/LvglLib/lv_uefi_display.c   ← GOP flush callback
LvglPkg/Library/LvglLib/lv_port_indev.c     ← Mouse + keyboard input
LvglPkg/Library/LvglLib/LvglLib.c           ← Init/deinit, main loop

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
```

## LvglDisplayEngineDxe Module (created)
```
LvglPkg/LvglDisplayEngineDxe/
  LvglDisplayEngineDxe.c     ← produces/installs EFI_DISPLAY_ENGINE_PROTOCOL
  LvglDisplayEngineDxe.inf   ← module INF
  LvglFormRenderer.c         ← FormDisplay(): FORM_DISPLAY_ENGINE_FORM → LVGL widgets
  LvglFormRenderer.h         ← Renderer types and API
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
- **PR #13** (closed): Original combined PR, split into #14/#15/#16 per maintainer request

## Current Status
- LvglDisplayEngineDxe skeleton: **done** — builds, installs protocol, wired into DSC/FDF
- LvglLib.inf fix: **done** — removed `UefiApplicationEntryPoint`, consumable by DXE_DRIVER
- FormDisplay() initial implementation: **done** — walks StatementListHead, creates LVGL widgets,
  runs event loop, returns user action to browser
- LVGL-based form UI renders on screen in QEMU when entering Setup

## Known Bugs
1. **Mouse not working** — mouse cursor does not appear / respond in the display engine.
   Root cause: `LvglLibConstructor` runs during DXE dispatch (before BDS `ConnectAll`),
   so `EfiMouseInit()` finds no USB pointer protocols and returns `EFI_UNSUPPORTED` — no
   mouse indev is ever created. By the time `FormDisplay()` runs, `UefiLvglInit()` short-
   circuits (`mUefiLvglInitDone == TRUE`) and never retries.
   Fix: `lv_uefi_mouse_create()` is now idempotent (early-returns if a pointer indev
   already exists, calls `EfiMouseInit()` internally). `lv_port_indev_init()` registers
   a protocol-install notification on `gEfiAbsolutePointerProtocolGuid` via
   `gBS->RegisterProtocolNotify()` — when the USB mouse binds during BDS `ConnectAll`,
   the callback fires and creates the indev. No change needed at the renderer boundary.
   Note: LVGL's `lv_indev_set_cursor()` already reparents cursors to `layer_sys`,
   so screen switching is not an issue.
2. **Arrow keys (UP/DOWN/LEFT/RIGHT) not working** — the keypad indev reads keys correctly
   (`lv_port_indev.c` maps SCAN_UP → LV_KEY_UP etc.), but LVGL's default group navigation
   uses LV_KEY_NEXT/LV_KEY_PREV (Tab/Shift-Tab). Arrow keys only work inside widgets
   (e.g. spinbox increment). Need to either: (a) remap arrows to NEXT/PREV for group
   navigation, or (b) enable `lv_group_set_editing()` style navigation, or (c) handle
   arrows in a custom key event callback that moves focus.
3. **ESC key not working** — `OnEscPressed` is registered on the screen object with
   `LV_EVENT_KEY`, but the screen itself is not in the focus group and never receives
   key events. Fix: register ESC handler on the group or on individual focused widgets,
   or use `lv_group_add_obj()` on a hidden focusable object.
4. **Fonts and colors need improvement** — current dark theme (0x1A1A2E / 0x16213E) is
   placeholder. Text readability is poor, subtitle/label contrast is insufficient.
   Need a proper theme pass: background, panel, text, accent, and disabled colors.
   Font sizes should be consistent and appropriate for 800x600 resolution.

## Next Steps
1. Fix mouse support — ensure cursor is visible and functional on the form screen
2. Fix keyboard navigation — arrow keys should move focus between form items
3. Fix ESC key — should trigger BROWSER_ACTION_FORM_EXIT reliably
4. Theme/styling pass — readable fonts, proper color palette, grayout styling
5. End-to-end test — verify form navigation, value changes, and save/discard flow