# How This Cursor Skill Was Created

This document describes how the **tinyapp** Cursor Agent Skill was built from
Dave Plummer's tiny-notepad process, and what was added for modern Windows 11
and AI-assisted development.

## Starting point: Dave's Garage tiny notepad

Dave Plummer's approach on [Dave's Garage](https://www.youtube.com/c/davesgarage)
reframes app development as **OS integration** rather than framework assembly:

- Windows already ships text editing (`RICHEDIT50W` in `Msftedit.dll`), file
  dialogs (`GetOpenFileName`), printing (`PrintDlg` + `EM_FORMATRANGE`), and
  menus (`CreateMenu` / `AppendMenu`).
- Your EXE only needs enough code to **call** those pieces.
- Every new menu item is a deliberate cost — rebuild, measure, log.

The published growth table from the TinyRetroPad lineage (32-bit + Crinkler)
looks like this:

| Milestone | Size (bytes) |
|-----------|-------------|
| FILE menus | 1,375 |
| EDIT menus | 1,428 |
| Open / Save As | 1,517 |
| HELP | 1,557 |
| Save-changes prompt | 1,622 |
| Time / Date (F5) | 1,668 |
| Word wrap | 1,694 |
| Context menu | 1,779 |
| Font dialog | 1,910 |
| Find / Replace | 2,143 |
| Print | 2,476 |

That table became the backbone of [examples.md](../skill/examples.md) Example 3.

## From YouTube series to agent instructions

Cursor Agent Skills are markdown files with YAML frontmatter. The agent reads
them when the user mentions trigger terms (tiny notepad, TinyRetroPad, Dave's
Garage, etc.).

Converting Dave's process into a skill required:

### 1. Extract invariant rules

Rules that appear in every Dave Plummer tiny-app video and repo:

- No C runtime (`/NODEFAULTLIB`, custom `MainEntry`)
- No resource compiler for menus/strings when runtime menus work
- Readable comments — explain non-obvious Win32 calls
- Complete window before size optimization
- One feature per rebuild with a growth log in the `.asm` file header

These became the **Hard constraints** and **Process** sections in
[SKILL.md](../skill/SKILL.md).

### 2. Split reference from workflow

[SKILL.md](../skill/SKILL.md) stays under ~500 lines (Cursor skill best
practice). Detailed material moved to:

- [reference.md](../skill/reference.md) — VS paths, ml64 ABI, Crinkler flags,
  common-dialog matrix, size tactics
- [examples.md](../skill/examples.md) — ordered feature recipes, cheap handlers,
  anti-examples

Progressive disclosure keeps the agent's context window focused.

### 3. Adapt for Windows 11 Pro 64-bit

Dave's published size records target **32-bit + Crinkler**. Most Windows 11
users want native **x64**. The skill defines two tracks:

| Track | Why it exists |
|-------|---------------|
| `win11-x64` (default) | Native 64-bit PE with `ml64`; same process, ~3–4× larger floor |
| `wow64-crinkler` | Matches HelloAssembly / TinyRetroPad when the user names those sizes |

The default changed from Dave's original 32-bit focus because Windows 11 has no
32-bit edition — WoW64 is optional, not the default host.

### 4. Encode Windows 11 realities Dave's videos predated or skimmed

Added without changing the core process:

- SmartScreen on unsigned EXEs (normal; do not auto-sign)
- Defender sensitivity to Crinkler `/TINYIMPORT` (warn; never disable AV)
- Win11 Store Notepad vs classic Win32 behavior this skill targets
- Optional DPI awareness and dark title bar as logged features
- Explicit ban on WinUI 3 / MSIX / Electron escape hatches

### 5. Write agent-safe guardrails

Skills run autonomously. Extra guardrails prevent harmful shortcuts:

- **Do not** tell the user to disable Defender or add AV exclusions
- **Do not** ship `Msftedit.dll` beside the EXE
- **Do not** start from handwritten PE / PEB-walk unless the user names the
  HelloAssembly `Lasse` or `Theron` challenge
- **Do not** assume MASM32 (`C:\masm32\include`) exists

## Skill file structure

```
skill/
├── SKILL.md       # Frontmatter + workflow + constraints (agent reads first)
├── reference.md   # Toolchain and API reference (read when building)
└── examples.md    # Growth order and code shapes (read when implementing)
```

Frontmatter `description` includes trigger terms so Cursor can discover the
skill:

```yaml
name: tinyapp
description: >-
  Write tiny native Windows 11 Pro 64-bit desktop apps using Dave Plummer's
  Dave's Garage tiny-notepad process ...
```

## Validation checklist

Before publishing, the skill was checked against:

- [ ] SKILL.md under 500 lines
- [ ] Description in third person with WHAT + WHEN
- [ ] All file references one level deep from SKILL.md
- [ ] Attribution links to HelloAssembly, TinyRetroPad, and Dave's Garage
- [ ] Both tracks documented with distinct build.bat shapes
- [ ] Growth log examples match TinyRetroPad published milestones
- [ ] No redistribution of Dave's `.asm` source — methodology only

## What was not copied

This repository does **not** include:

- `tiny.asm`, `trpad.asm`, or other PlummersSoftwareLLC source files
- Crinkler binaries
- Prebuilt EXEs

Clone the upstream repos for canonical source:

- https://github.com/PlummersSoftwareLLC/HelloAssembly
- https://github.com/PlummersSoftwareLLC/TinyRetroPad

## Future improvements

Possible extensions (contributions welcome):

- A minimal sample `win11-x64` app in a separate example repo
- CI that verifies skill markdown structure
- Links to additional Dave's Garage episodes as they cover tiny apps

## Author of this skill packaging

The Cursor skill files and documentation in this repository were authored to
**teach and credit** Dave Plummer's process. The underlying methodology and
size-record achievements belong to Dave Plummer, Matt Power (DTE), and the
HelloAssembly / TinyRetroPad contributors.
