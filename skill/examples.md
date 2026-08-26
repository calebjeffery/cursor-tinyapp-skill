# Tinyapp — Windows 11 Pro 64-bit Examples

Default track is **win11-x64**. Use wow64-crinkler examples only when the
user wants TinyRetroPad-class sizes.

## Example 1: Baseline complete window (win11-x64)

User: "smallest real Windows 11 app, Dave's Garage style"

Deliver:

1. `tiny.asm` + local prototypes (`win32.inc` or top of file)
2. `build.bat` that calls `vcvars64.bat`, `ml64`, `link /NODEFAULTLIB`
3. Growth log starting at the first successful EXE size
4. `dumpbin /headers` showing `8664 machine (x64)`

Shape (do not paste HelloAssembly Tiny.asm — that is 32-bit):

```
option casemap:none
includelib kernel32.lib
includelib user32.lib
includelib gdi32.lib          ; only because this baseline paints text

GetModuleHandleW proto :qword
; ...other *W protos...

WindowWidth  equ 640
WindowHeight equ 480

.data
AppName dw "Dave's Tiny App", 0

.code
MainEntry proc
  local hInstance:qword
  local wc:WNDCLASSEXW
  local msg:MSG
  local hwnd:qword
  ; xor ecx, ecx / call GetModuleHandleW
  ; zero wc (rep stosq); fill cbSize, CS_HREDRAW or CS_VREDRAW, WndProc,
  ;   hInstance, COLOR_3DSHADOW+1, class name
  ; RegisterClassExW / CreateWindowExW WS_OVERLAPPEDWINDOW or WS_VISIBLE
  ; UpdateWindow
  ; message loop (GetMessageW / TranslateMessage / DispatchMessageW)
MainEntry endp

WndProc proc hWnd:qword, uMsg:dword, wParam:qword, lParam:qword
  ; WM_DESTROY -> PostQuitMessage
  ; WM_PAINT   -> BeginPaint, SetBkMode TRANSPARENT, GetClientRect,
  ;               DrawTextW DT_SINGLELINE or DT_CENTER or DT_VCENTER, EndPaint
  ; else DefWindowProcW
WndProc endp
end
```

`WNDCLASSEXW` / `MSG` field layouts must be the x64 ones (pointer fields
are 8 bytes; watch `cbSize`). If a struct is wrong, `RegisterClassExW`
fails and you get no window — fix the struct before chasing bytes.

Done when min/max/close and the centered title text work on this PC.

## Example 2: Tiny editor (the notepad process, win11-x64)

User: "tiny notepad" / "TinyRetroPad-style editor" on Windows 11

Grow in this order. Rebuild and log bytes after each line.

1. Baseline window (drop `gdi32` once the child exists)
2. `LoadLibraryW("Msftedit.dll")` + child `RICHEDIT50W`
3. `EM_EXLIMITTEXT` + Consolas via `EM_SETCHARFORMAT`
4. File menu: New / Exit
5. Open / Save As via `GetOpenFileNameW` / `GetSaveFileNameW`
6. Dirty flag + "Save changes?" on New / Open / Exit
7. Edit menu: Undo/Cut/Copy/Paste/Delete/Select All as one
   `SendMessageW` each
8. Find / Replace (modeless common dialogs)
9. Word wrap (`EM_SETTARGETDEVICE`)
10. Font dialog
11. Print (`PrintDlgW` + `EM_FORMATRANGE`)
12. Status bar (`STATIC` + `EM_EXGETSEL` / `EM_EXLINEFROMCHAR`)

Optional Win11 extras, each its own logged feature:

13. DPI awareness
14. Dark title bar (`dwmapi`)

Stop when the user's requested subset works. Do not add tabs, regex,
spellcheck, or a WinUI theme unless asked.

TinyRetroPad's published 32-bit Crinkler sizes are **not** the x64 budget.
On win11-x64 expect roughly 3–4× those numbers. Still log every step.

## Example 3: Same editor, wow64-crinkler

User: "I want the 2.5 KB TinyRetroPad sizes"

1. Confirm Crinkler is on `PATH`. If not, stop and say it must be
   downloaded. Do not invent a 64-bit Crinkler.
2. Use `vcvars32.bat` + `ml /c /coff` + Crinkler.
3. Follow HelloAssembly / TinyRetroPad `*A` source shape.
4. Run the 32-bit EXE on Windows 11 Pro (WoW64).
5. Log bytes against the TinyRetroPad growth table:

```
FILE menus                 1375
EDIT menus                 1428
Open/Save As               1517
HELP                       1557
Save prompt                1622
Time/Date                  1668
Word wrap                  1694
Context menu               1779
Font dialog                1910
Find/Replace               2143
Print                      2476
```

## Example 4: Cheap feature — Edit → Paste

win11-x64:

```
CmdPaste:
  mov rcx, hEdit
  mov edx, WM_PASTE
  xor r8, r8
  xor r9, r9
  call SendMessageW
  ret
```

`WM_COMMAND` compares `IDM_EDIT_PASTE` and jumps here. No clipboard code.

## Example 5: Cheap feature — File → Open

1. Fill one `OPENFILENAMEW` on the stack (`lStructSize`, `hwndOwner`,
   `lpstrFilter` = `dw "All Files",0,"*.*",0,0`, `lpstrFile` = shared
   `CmdFile` buffer, `nMaxFile` = `MAX_CMD_PATH`, `Flags` =
   `OFN_FILEMUSTEXIST or OFN_PATHMUSTEXIST`)
2. `GetOpenFileNameW`
3. Reuse the same loader as drag-and-drop (`CreateFileW` / `ReadFile` /
   `EM_SETTEXTEX` or `SetWindowTextW`)
4. Clear `fDirty`; refresh the title

Do not write a custom file-picker or `IFileDialog`.

## Example 6: Optional feature switch

```
FEAT_DARKMODE = 0

CmdDarkMode:
IF FEAT_DARKMODE
  ; DwmSetWindowAttribute + EM_SETBKGNDCOLOR
ENDIF
  ret
```

With the switch at 0 the handler and strings must disappear from the
object file. If they still land in the EXE, the `IF` is in the wrong
place.

## Example 7: Utility that is not an editor

User: "tiny color picker" / "tiny countdown" / "tiny sticky note"

Same process, win11-x64 unless they asked for Crinkler:

1. Name the Win32 piece (`ChooseColor`, `SetTimer`, `EDIT` + `STATIC`)
2. Complete window
3. One feature, measure, repeat

A color picker that only calls `ChooseColor` + copies `#RRGGBB` to the
clipboard should stay well under 8 KB x64. If the draft grows past 24 KB,
a control or common dialog is being reimplemented.

## Anti-examples

- Targeting 32-bit Windows, or assuming `C:\masm32\include` exists
- Shipping `msftedit.dll` next to the EXE
- Adding a custom text layout engine
- Using WinUI 3 / Packaged Win11 Notepad as the stack
- Starting from a handwritten PE because "smaller"
- A CMake + vcpkg + imgui scaffold
- Disabling Defender so Crinkler can link
- Calling 32-bit Crinkler the "Windows 11 64-bit" default
