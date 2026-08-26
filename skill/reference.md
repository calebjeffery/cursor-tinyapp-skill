# Tinyapp — Windows 11 Pro 64-bit Reference

Read this when scaffolding a build or chasing bytes. Paths are examples for
Windows 11 Pro 64-bit with VS 2022 Build Tools and SDK 10.0.26100.0.
Re-discover them if the machine changes.

## Host paths

```
VS      C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools
MSVC    ...\VC\Tools\MSVC\14.44.35207
vcvars64    ...\VC\Auxiliary\Build\vcvars64.bat
vcvars32    ...\VC\Auxiliary\Build\vcvars32.bat
ml64    ...\bin\Hostx64\x64\ml64.exe
ml      ...\bin\Hostx64\x86\ml.exe
link    ...\bin\Hostx64\x64\link.exe   (x64)
        ...\bin\Hostx64\x86\link.exe   (x86)
SDK     C:\Program Files (x86)\Windows Kits\10\Lib\10.0.26100.0\um
        \x64\kernel32.Lib
        \x86\kernel32.Lib
```

`vcvars64.bat` / `vcvars32.bat` put `ml64` / `ml`, `link`, and the SDK on
`PATH`. Prefer that over hard-coding `Hostx64` bins.

There is no MASM32 install. Write prototypes in the app (see below).
Crinkler is not on `PATH` by default; the wow64-crinkler track needs
[Crinkler](https://github.com/runestubbe/Crinkler) downloaded first.

## Track A — win11-x64 (default)

Native 64-bit PE. No Crinkler. Same Dave process, larger floor.

### build.bat

```bat
@echo off
setlocal
call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat"
ml64 /c /nologo /W3 app.asm
if errorlevel 1 exit /b 1
link /nologo /NODEFAULTLIB /ENTRY:MainEntry /SUBSYSTEM:WINDOWS /MERGE:.rdata=.text kernel32.lib user32.lib app.obj /OUT:app.exe
dumpbin /headers app.exe | findstr /C:"machine" /C:"size of image"
```

Add libs only when imported: `comdlg32.lib`, `shell32.lib`, `comctl32.lib`,
`gdi32.lib`, `dwmapi.lib` (dark title bar). Skip `gdi32.lib` if
`EM_SETCHARFORMAT` covers fonts.

Do **not** merge `.pdata` away. x64 unwind data belongs there.

Confirm the PE:

```
dumpbin /headers app.exe
```

Must show `8664 machine (x64)`.

### ml64 conventions

```
option casemap:none
includelib kernel32.lib
includelib user32.lib

GetModuleHandleW proto :qword
CreateWindowExW  proto :dword, :qword, :qword, :dword, :dword, :dword, :dword, :dword, :qword, :qword, :qword, :qword
```

- No `.386`, no `.model flat, stdcall`. ml64 is already x64.
- Windows x64 ABI: RCX, RDX, R8, R9, then stack; caller allocates 32 bytes
  of shadow space. `invoke` is fine if every `proto` is correct.
- Handles and pointers are `qword`. `WPARAM` / `LPARAM` / `LRESULT` are
  64-bit.
- `LOCAL` structs in a `proc` are fine. Zero them with `rep stosq` (or a
  short loop), not field-by-field zero stores.
- Import names have **no** `@N` stdcall suffix. Use `GetModuleHandleW`, not
  `_imp__GetModuleHandleA@4`.
- Prefer `*W` APIs. String literals are UTF-16: `AppName dw "Tiny App", 0`.
- Entry is `MainEntry`. `END` is optional in ml64; keep `/ENTRY:MainEntry`.

### Entry and window (x64)

```
MainEntry proc
  ; xor ecx, ecx / call GetModuleHandleW
  ; zero WNDCLASSEXW; fill cbSize, style, lpfnWndProc, hInstance,
  ;   hbrBackground, lpszClassName
  ; RegisterClassExW
  ; CreateWindowExW(..., WS_OVERLAPPEDWINDOW or WS_VISIBLE, ...)
  ; UpdateWindow
  ; GetMessageW / TranslateMessage / DispatchMessageW until 0
  ; return msg.wParam
MainEntry endp
```

`WndProc` handles `WM_DESTROY` → `PostQuitMessage`, then the messages the
app owns, and forwards the rest to `DefWindowProcW`.

## Track B — wow64-crinkler (size record)

32-bit EXE, runs under WoW64 on Windows 11 Pro. This is the
HelloAssembly / TinyRetroPad binary shape.

### build.bat

```bat
@echo off
setlocal
call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars32.bat"
set KIT=C:\Program Files (x86)\Windows Kits\10\Lib\10.0.26100.0\um\x86
ml /c /coff /nologo app.asm
if errorlevel 1 exit /b 1
crinkler.exe /NODEFAULTLIB /ENTRY:MainEntry /SUBSYSTEM:WINDOWS /TINYHEADER /NOINITIALIZERS /UNSAFEIMPORT /ORDERTRIES:2000 /TINYIMPORT /LIBPATH:"%KIT%" kernel32.lib user32.lib app.obj /OUT:app.exe
```

Source shape stays HelloAssembly: `.386`, `.model flat, stdcall`,
`option casemap:none`, `*A` APIs, `EXTERN _imp__GetModuleHandleA@4 :PTR`
or matching `proto STDCALL`.

If `ml` errors on missing `windows.inc`, do not install MASM32 just to
silence it — write the few `proto` / `equ` lines the file needs.

### Crinkler flags

| Flag | Why |
|------|-----|
| `/NODEFAULTLIB` | No CRT |
| `/ENTRY:MainEntry` | Must match the proc |
| `/SUBSYSTEM:WINDOWS` | No console |
| `/TINYHEADER` | Smaller header |
| `/NOINITIALIZERS` | Drops C++ static-init support |
| `/UNSAFEIMPORT` | No import-failure MessageBox |
| `/TINYIMPORT` | Compact import scheme |
| `/ORDERTRIES:1000` or `2000` | Section-order search |

If Defender quarantines the EXE, drop `/TINYIMPORT`, then `/UNSAFEIMPORT`,
then `/TINYHEADER`. Last resort: `link` from `vcvars32` with
`/NODEFAULTLIB /ENTRY:MainEntry /SUBSYSTEM:WINDOWS /MERGE:.rdata=.text
/ALIGN:16`.

HelloAssembly published sizes (32-bit, not a Win11-x64 contract):
TinyOriginal + Crinkler ~540 B; Lasse + Crinkler ~818 B; Lasse + MASM32
~1104 B; Theron handwritten PE ~383 B.

## Child control (both tracks)

```
; win11-x64
lea rcx, szMsftedit          ; L"Msftedit.dll"
call LoadLibraryW
; CreateWindowExW class L"RICHEDIT50W", child of the main window
```

Then:

- `EM_EXLIMITTEXT` — raise the 32K/64K default
- `EM_SETCHARFORMAT` + `CFM_FACE` — Consolas / Courier New without `gdi32`
- `EM_SETEVENTMASK` + `ENM_CHANGE` — dirty flag
- `EM_SETTARGETDEVICE` — word wrap on/off
- `EM_EXGETSEL` / `EM_EXLINEFROMCHAR` / `EM_LINEINDEX` — Ln/Col

On win11-x64, `Msftedit.dll` loads from `System32`. On wow64-crinkler it
loads from `SysWOW64`. Same class name: `RICHEDIT50W`.

## Menus without a resource script

```
CreateMenu
  CreatePopupMenu → AppendMenuW items → AppendMenuW onto the bar
SetMenu
```

Thin wrappers (`AppendEnabled` / `AppendDisabled`) beat repeating
`AppendMenuW` setup. A disabled item with a null string is a separator.

Save can live on the **system menu** (`GetSystemMenu` + `AppendMenuW`) and
arrive as `WM_SYSCOMMAND`.

Context menu: `TrackPopupMenu` on the same popup; same IDs, almost free.

## Common dialogs

Stay on the classic common dialogs. Win11 `IFileDialog` is COM + more
imports.

| Need | win11-x64 | wow64-crinkler |
|------|-----------|----------------|
| Open / Save As | `GetOpenFileNameW` / `GetSaveFileNameW` | `*A` |
| Find / Replace | `FindTextW` / `ReplaceTextW` (modeless) | `*A` |
| Find message | `RegisterWindowMessageW("commdlg_FindReplace")` | `*A` |
| Font | `ChooseFontW` | `ChooseFontW` |
| Page setup / print | `PageSetupDlgW` / `PrintDlgW` | `*A` |
| Custom tiny dialog | `DialogBoxIndirectParamW` + in-memory `DLGTEMPLATE` | `*A` |

Modeless find/replace requires `IsDialogMessageW` in the message loop.

File bytes: `CreateFileW` + `GetFileSize` + `GlobalAlloc` + `ReadFile` /
`WriteFile` + `CloseHandle`. Share that path between drop-file, Open, and
Save.

## Messages that replace code

| Feature | Message / API |
|---------|----------------|
| Undo / cut / copy / paste / delete | `WM_UNDO` / `WM_CUT` / `WM_COPY` / `WM_PASTE` / `WM_CLEAR` |
| Select all | `EM_SETSEL` (0, -1) |
| Insert text | `EM_REPLACESEL` |
| Find | `EM_FINDTEXTEXW` |
| Print layout | `EM_FORMATRANGE` |
| Wrap | `EM_SETTARGETDEVICE` |
| Time/date | `GetLocalTime` + `GetTimeFormatW` / `GetDateFormatW` |

## Windows 11 extras (optional, log the bytes)

| Extra | API | Notes |
|-------|-----|-------|
| DPI | `SetProcessDpiAwarenessContext(-4)` | Per-monitor v2. One call at startup. Needs `user32`. |
| Dark title bar | `DwmSetWindowAttribute(hwnd, 20, &TRUE, 4)` | Needs `dwmapi.lib`. Does not theme the RICHEDIT. |
| Dark edit background | `EM_SETBKGNDCOLOR` | Cheap if you already have the control |

Do not add a manifest just for DPI if the one call is enough.

## Size tactics that stay readable

Do these first:

1. Let a system control own the feature
2. One `SendMessage` handler per menu ID
3. Stack structs, BSS/uninitialized for large scratch
4. Share buffers and plumbing
5. Compile-time `FEAT_*` switches so unused features are gone
6. Drop unused imports (`gdi32` is the usual first win)
7. On x64, `/MERGE:.rdata=.text` and `/O1` (C) — not handwritten PE

Do these only for a named size-record challenge (wow64-crinkler / Lasse /
Theron):

- Crinkler `/TINYHEADER` `/TINYIMPORT`
- Handwritten PE header
- Import by hash / PEB `InMemoryOrderModuleList` walk

Those last two resemble malware loaders. They are not the tiny-notepad path
and they are worse on Windows 11 Defender.

## RAM vs disk

A few-KB EXE mapping `msftedit.dll` can show hundreds of MB in Task Manager.
That is the OS working set for shared DLLs, not a leak. Do not add a custom
text engine to "fix" it.

## Sources

- [HelloAssembly](https://github.com/PlummersSoftwareLLC/HelloAssembly) — Dave Plummer
- [TinyRetroPad](https://github.com/PlummersSoftwareLLC/TinyRetroPad) — Dave Plummer et al.
- [Crinkler](https://github.com/runestubbe/Crinkler) (32-bit only)
- Dave's Garage: [Hello, Assembly!](https://youtu.be/b0zxIfJJLAY), [C vs ASM](https://youtu.be/-Vw-ONPfaFk), [Can we build Notepad in 3K?](https://www.youtube.com/watch?v=OG91c7xsNMc)
