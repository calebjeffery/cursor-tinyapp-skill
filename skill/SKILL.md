---
name: tinyapp
description: >-
  Write tiny native Windows 11 Pro 64-bit desktop apps using Dave Plummer's
  Dave's Garage tiny-notepad process (wrap Win32, no CRT, one feature per
  rebuild, log byte cost). Default target is native x64 on Windows 11 Pro.
  Optional WoW64 + Crinkler path matches HelloAssembly / TinyRetroPad sizes.
  Use when the user asks for tinyapp, a tiny Windows 11 app, TinyRetroPad-style
  editor, few-KB Win32 EXE, Dave's Garage tiny notepad, DTE, Crinkler, ml64, or
  a size-obsessed native Windows 11 UI.
---

# Tinyapp — Windows 11 Pro 64-bit

Write **complete, usable** native Win32 desktop apps for **Windows 11 Pro
64-bit**. The Dave's Garage process stays the same: wrap what the OS already
ships, add one feature, measure bytes. The host and target are this OS, not
32-bit Windows 10.

## Lineage and credit

This skill documents a process pioneered by **Dave Plummer** on the
[**Dave's Garage**](https://www.youtube.com/c/davesgarage) YouTube channel.
The canonical source projects are:

- [HelloAssembly](https://github.com/PlummersSoftwareLLC/HelloAssembly) — `tiny.asm`
- [TinyRetroPad](https://github.com/PlummersSoftwareLLC/TinyRetroPad) — `trpad.asm`

See [docs/ATTRIBUTION.md](../docs/ATTRIBUTION.md) in this repository for full
credit, video links, and the fork lineage (HelloAssembly → DTE → TinyRetroPad).

## When this skill applies

Use this process for: tiny notepad, TinyRetroPad, DTE, Dave's Garage Windows
app, smallest Win32 EXE, Crinkler, MASM / ml64 desktop app, "no bloat / no
telemetry" native Windows 11 UI.

Do **not** use Electron, WinUI 3, .NET, Qt, MFC, or a C runtime.

## Pick a track

| Track | When | Toolchain | Typical size |
|-------|------|-----------|--------------|
| **win11-x64** (default) | User said Windows 11 / 64-bit, or did not ask for a size record | `ml64` + `link` from VS 2022 Build Tools | few KB to ~20 KB |
| **wow64-crinkler** | User named TinyRetroPad, Crinkler, sub-KB, or "Dave's exact sizes" | `ml` (x86) + Crinkler | hundreds of bytes to ~3 KB |

Both tracks use the same process. Both run on Windows 11 Pro 64-bit.
`wow64-crinkler` produces a 32-bit EXE that WoW64 runs. There is no 32-bit
Windows 11.

Default to **win11-x64**. Do not start from PEB-walk / hash-import /
handwritten-PE tricks unless the user names the HelloAssembly `Lasse` or
`Theron` challenge.

## Hard constraints

- **Host and target: Windows 11 Pro 64-bit.** Build with VS 2022 + Windows 11 SDK.
  See [reference.md](reference.md) for paths.
- **No CRT.** `/NODEFAULTLIB`. Custom entry (`MainEntry`). No
  `WinMainCRTStartup`.
- **No bundled UI.** Wrap `EDIT`, `RICHEDIT50W`, `STATIC`, common dialogs,
  or `MessageBoxW`.
- **No `.rc` resources** unless unavoidable. Packed `dw`/`db` strings and
  in-memory dialog templates (`DialogBoxIndirectParamW`).
- **No MASM32 includes.** Write explicit `proto` lines (or a local
  `win32.inc` in the app folder).
- **Readable comments.** Dave's rule: explain what you are doing.
- **One feature per rebuild.** Record EXE size after every addition.
- **Do not disable antivirus, and do not tell the user how.**

Prefer **Unicode `*W` APIs** on the win11-x64 track (Win11 paths and
filenames). Use `*A` only on the wow64-crinkler track to match TinyRetroPad.

## Process

Copy this checklist and keep it updated:

```
Tiny app progress:
- [ ] 0. Track = win11-x64 | wow64-crinkler
- [ ] 1. Name the job and the Windows 11 control that already does it
- [ ] 2. Scaffold the complete window (message loop + real chrome)
- [ ] 3. Wrap the control; prove it paints and closes on this PC
- [ ] 4. Add one feature; rebuild; log bytes
- [ ] 5. Repeat 4 until the requested app is done
- [ ] 6. Write the growth log and build.bat (vcvars64 or vcvars32)
```

### 1. Name the OS feature, not the app

Ask: *what does Windows 11 already provide?* Then wrap that.

| Job | Wrap this |
|-----|-----------|
| Text editor | `RICHEDIT50W` from `Msftedit.dll` (default). `EDIT` only for a smaller, more limited editor |
| Button / label / list | `BUTTON`, `STATIC`, `LISTBOX`, `COMBOBOX` |
| File pick | `GetOpenFileNameW` / `GetSaveFileNameW` (not `IFileDialog` — COM is expensive) |
| Find / replace | `FindTextW` / `ReplaceTextW` + `EM_FINDTEXTEXW` |
| Font | `ChooseFontW` + `EM_SETCHARFORMAT` (avoids `gdi32` for face selection) |
| Print | `PrintDlgW` + `EM_FORMATRANGE` |
| Confirm / about | `MessageBoxW` |
| Open a URL | `ShellExecuteW` |

`Msftedit.dll` is already in `System32` (x64) and `SysWOW64` (32-bit). Never
ship it next to the EXE.

If the feature needs custom GDI drawing, treat it as expensive and justify it.

### 2. Scaffold a complete window first

A valid baseline is a window that:

- runs a `GetMessageW` / `TranslateMessage` / `DispatchMessageW` loop
- has title bar, min / max / close, and a working system menu
- paints something real (text or a child control)
- posts quit on `WM_DESTROY`
- is a native **x64** PE on the default track (`dumpbin /headers` shows
  `8664 machine (x64)`)

win11-x64 shape: `option casemap:none`, x64 ABI (RCX/RDX/R8/R9), stack
`LOCAL` structs, `WS_OVERLAPPEDWINDOW or WS_VISIBLE`.

wow64-crinkler shape: HelloAssembly `TinyOriginal/Tiny.asm` — `.386`,
`.model flat, stdcall`, `rep stosd` zeroing, `*A` APIs.

Do not skip the working window to "save bytes." Size work starts *after*
the app is complete.

Optional Win11 polish (each is a feature with a byte cost — log it):

- `SetProcessDpiAwarenessContext(-4)` so the window is not bitmap-stretched
- `DwmSetWindowAttribute` + `DWMWA_USE_IMMERSIVE_DARK_MODE` (20) for a dark
  title bar

### 3. Make the control do the heavy lifting

Create the system control as a child and forward work with `SendMessageW`.

Typical editor setup:

1. `LoadLibraryW("Msftedit.dll")`
2. `CreateWindowExW` class `RICHEDIT50W`
3. `EM_EXLIMITTEXT` for large files
4. `EM_SETCHARFORMAT` for Consolas or Courier New (no `gdi32` import)
5. `EM_SETEVENTMASK` + `EN_CHANGE` for a dirty flag

Edit-menu items are one message each: `WM_UNDO`, `WM_CUT`, `WM_COPY`,
`WM_PASTE`, `WM_CLEAR`, `EM_SETSEL`.

### 4. Add features as cheap command handlers

Build menus at runtime (`CreateMenu` / `CreatePopupMenu` / `AppendMenuW`).
Route `WM_COMMAND` through a flat compare / jump list to `Cmd…` handlers.

Rules for a cheap feature:

- One or two instructions after the compare, if possible
- Reuse existing load/save/path plumbing
- Reuse the same menu IDs for the context menu (`TrackPopupMenu`)
- Prefer common dialogs over hand-rolled UI or Win11 COM dialogs
- Gate optional extras with `FEAT_* = 0/1` assemble-time switches

### 5. Measure every addition

After each feature, rebuild and record size at the top of the `.asm` file:

```
; Growth History (win11-x64):
; Baseline window + RICHEDIT     -  8192 Bytes
; Added FILE menu                -  9216 Bytes
```

If a feature costs more than ~400 bytes on x64 (or ~200 on Crinkler), look
for a Win32 API that already does it before writing more code.

### 6. Ship the build, not a framework

Every app gets:

- one `.asm` (plus a local `win32.inc` of prototypes if needed)
- `build.bat` that calls `vcvars64.bat` (or `vcvars32.bat` + Crinkler)
- a growth log
- a short README: track, size, Windows 11 Pro 64-bit, SmartScreen / Defender note

Toolchain, exact paths, and ABI notes: [reference.md](reference.md).
Skeletons and feature recipes: [examples.md](examples.md).

## Size budget

| Class | win11-x64 | wow64-crinkler |
|-------|-----------|----------------|
| Complete painted window | ~3–8 KB | ~400–1100 B |
| Bare editor (RICHEDIT wrap) | ~4–10 KB | ~980 B |
| Notepad-class menus + dialogs | ~8–20 KB | ~2.5–3 KB |
| Small utility | stay under 24 KB | stay under 4 KB |

Disk size is the score. Task Manager working set will look huge because the
process maps `user32` / `msftedit`. Do not "fix" that by bundling an editor
engine.

## Language choice

**Default: ml64** (win11-x64) or **ml** (wow64-crinkler).

**C (only if asked):** freestanding, no CRT, `windows.h` from the Windows 11
SDK, same wrap-the-OS process. x64: `cl /c /GS- /O1` + `link /NODEFAULTLIB
/ENTRY:main /SUBSYSTEM:WINDOWS`. Crinkler C is 32-bit only.

**Not default:** handwritten PE, import-by-hash, PEB walk.

## Windows 11 Pro notes

- Unsigned EXEs get a SmartScreen prompt. That is normal. Do not sign unless
  asked.
- Defender is stricter than on the original TinyRetroPad videos, especially
  on Crinkler output. Warn once. Do not disable it.
- Win11 Notepad is a Store/WinUI app. This skill recreates classic Win32
  Notepad behavior, not the Store one.
- Do not use WinUI, XAML Islands, or Packaged/MSIX unless the user asks —
  those are not tiny.

## Antivirus

Crinkler + `/TINYHEADER` + `/TINYIMPORT` + `/UNSAFEIMPORT` looks like packed
malware to Defender. State the risk in the README. Do not disable AV, do not
add exclusion scripts, and do not walk the user through turning Defender off.

If a Crinkler build is deleted on link, drop `/TINYIMPORT`, then
`/UNSAFEIMPORT`, then `/TINYHEADER`, or switch to the win11-x64 track.

## Done means

- Track is stated and the PE machine type matches it (`8664` or `14C`)
- The window is complete and the requested features work on this Windows 11
  Pro 64-bit PC
- Growth log matches the last successful build
- `build.bat` uses `vcvars64.bat` or `vcvars32.bat` from VS 2022 Build Tools
- No CRT, no extra DLLs shipped next to the EXE
- Comments still explain the non-obvious Win32 calls
