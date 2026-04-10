# CLAUDE.md — edk2 + LvglPkg Project Context

## Project Goal
Replace EDK2's native HII Form Browser (text-based UI) with an LVGL-based graphical renderer.
The approach is **Runtime IFR Parser** — read HII database at runtime, parse IFR opcodes, and render the UI through LVGL instead of EDK2's DisplayEngineDxe.

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
                         → IFR Parser reads HII Database
                         → LVGL renders forms
```

### HII Architecture (what we're replacing)
- VFR files → compiled to IFR binary at build time
- IFR binary → published to HII Database via `EFI_HII_DATABASE_PROTOCOL`
- HII Database → read by `SetupBrowserDxe` + `DisplayEngineDxe`
- **Plan**: Replace `DisplayEngineDxe` with our LVGL renderer
- **Key protocol**: `EFI_FORM_BROWSER2_PROTOCOL` is the interface boundary

### Runtime IFR Parser Approach
1. Locate `EFI_HII_DATABASE_PROTOCOL`
2. Call `ExportPackageLists` to get all registered packages
3. Walk IFR opcodes (EFI_IFR_OP_HEADER based) to build form tree
4. Map form tree → LVGL widgets
5. Handle callbacks via `EFI_HII_CONFIG_ACCESS_PROTOCOL`

## Mouse/Input Status
- **SimplePointer**: ✅ Working (usb-mouse device in QEMU)
- **AbsolutePointer**: ⚠️ Partially working — `UsbMouseAbsolutePointerDxe` added to OVMF
  but QEMU `usb-tablet` uses `Subclass:0, Protocol:0` while driver expects
  `Subclass:1 (Boot), Protocol:2 (Mouse)`. Use `usb-mouse` for now.
- **Keyboard**: ✅ Working via `EFI_SIMPLE_TEXT_INPUT_EX_PROTOCOL`

### Mouse Fix Applied (lv_port_indev.c)
Original code used `ConsoleInHandle` to get pointer protocol — this only gets
the ConSplitter aggregate, not the real device. Fixed to use `LocateHandleBuffer`
with `DevicePath` filter to find actual USB device handles.

## Known Issues / TODO
- [ ] AbsolutePointer with usb-tablet: `UsbMouseAbsolutePointerDxe` doesn't bind
      to QEMU usb-tablet (descriptor mismatch: Subclass/Protocol 0x00 vs expected 0x01/0x02)
- [ ] Mouse wheel support
- [ ] IFR Parser — not started yet (next major milestone)
- [ ] DisplayEngineDxe replacement — after IFR parser

## Key Source Files
```
# LVGL UEFI port
LvglPkg/Library/LvglLib/lv_uefi_display.c   ← GOP flush callback
LvglPkg/Library/LvglLib/lv_port_indev.c     ← Mouse + keyboard input
LvglPkg/Library/LvglLib/LvglLib.c           ← Init/deinit, main loop

# HII/IFR (what we'll be working with next)
MdePkg/Include/Uefi/UefiInternalFormRepresentation.h  ← All IFR opcodes
MdePkg/Include/Protocol/HiiDatabase.h                 ← HII DB protocol
MdePkg/Include/Protocol/FormBrowser2.h                ← Form Browser protocol
MdeModulePkg/Universal/DisplayEngineDxe/              ← Will be replaced
MdeModulePkg/Universal/SetupBrowserDxe/               ← Form logic + IFR parser
MdeModulePkg/Universal/DriverSampleDxe/               ← Best VFR/HII example
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

LvglPkg fork:
  branch: master ← active development

# Sync upstream edk2
git checkout master
git fetch upstream
git merge upstream/master
git checkout dev
git rebase master
```

## Fixes Contributed / Pending PR to YangGangUEFI/LvglPkg
1. Remove unused `Status` variable in `lv_uefi_display.c` (`-Werror=unused-but-set-variable`)
2. Add `EFIAPI` to `LvglUefiDemo` in `LvglDemoApp.c` (calling convention mismatch)
3. Add forward declaration + `EFIAPI` wrapper for `lv_demo_keypad_encoder` in `LvglDemos.c`
4. Fix mouse init to use `LocateHandleBuffer` instead of `ConsoleInHandle`

## Next Steps
1. Open PR to YangGangUEFI/LvglPkg with the fixes above
2. Study `SetupBrowserDxe` IFR parser implementation
3. Write standalone IFR parser as a UEFI application (proof of concept)
4. Integrate IFR parser output with LVGL widget creation
