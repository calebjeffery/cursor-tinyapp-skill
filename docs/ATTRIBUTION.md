# Attribution

This Cursor Agent Skill teaches a software development **process**, not a fork of
Dave Plummer's assembly source. The process itself — wrap Win32, skip the CRT,
add one feature per rebuild, measure bytes — comes from Dave Plummer's work on
[**Dave's Garage**](https://www.youtube.com/c/davesgarage) and the open-source
projects listed below.

## Dave Plummer and Dave's Garage

**Dave Plummer** is a former Microsoft engineer who created the original
`tiny.asm` HelloAssembly project and later [TinyRetroPad](https://github.com/PlummersSoftwareLLC/TinyRetroPad), a Notepad-style text editor in roughly 2.5 KB.

His YouTube channel [**Dave's Garage**](https://www.youtube.com/c/davesgarage)
documented the challenge of building a functional text editor by:

1. Starting from the smallest possible Win32 window
2. Wrapping OS controls (`RICHEDIT50W`, common dialogs) instead of building UI from scratch
3. Adding **one menu feature at a time**
4. Recording the EXE size after every addition — treating disk bytes as the score

### Recommended videos

| Title | Link |
|-------|------|
| Hello, Assembly! | https://youtu.be/b0zxIfJJLAY |
| C vs ASM | https://youtu.be/-Vw-ONPfaFk |
| The Challenge: Can we build Notepad in 3K in assembly language? | https://www.youtube.com/watch?v=OG91c7xsNMc |

The 3K Notepad video walks through the growth log in detail — for example,
Open/Save As at 1,517 bytes, the unsaved-work prompt at 1,622 bytes, word wrap
at 1,694 bytes, and the context menu at 1,779 bytes. That byte-by-byte
discipline is the heart of the tiny-notepad process this skill encodes.

## Project lineage

```
HelloAssembly (tiny.asm)
    └── Dave's Tiny Editor (DTE) — Matt Power
            └── TinyRetroPad (trpad.asm) — Dave Plummer et al.
                    └── This Cursor skill (methodology documentation)
```

| Project | Repository | Author(s) | Contribution |
|---------|------------|-----------|--------------|
| HelloAssembly | https://github.com/PlummersSoftwareLLC/HelloAssembly | Dave Plummer | Sub-KB Win32 window; Crinkler size challenges (`Lasse`, `Theron`) |
| Dave's Tiny Editor | (bundled with DTE releases) | Matt Power | RICHEDIT-based sub-1 KB editor |
| TinyRetroPad | https://github.com/PlummersSoftwareLLC/TinyRetroPad | Dave Plummer, rbergen, contributors | Full Notepad menu set in ~2.5 KB |
| Crinkler | https://github.com/runestubbe/Crinkler | runestubbe et al. | 32-bit EXE compressor used by TinyRetroPad |

TinyRetroPad's README states:

> TinyRetroPad is a fork of **Dave's Tiny Editor (DTE)** by Matt Power, which
> is itself an extension of `tiny.asm` HelloAssembly by Dave Plummer.

This skill cites that lineage explicitly and does not claim authorship of the
original assembly work.

## What this repository adds

This repo is a **Cursor Agent Skill** — markdown instructions that teach an AI
coding agent how to apply Dave's process on **Windows 11 Pro 64-bit**, including:

- A native **x64** track (`ml64`) as the default (Dave's published size records
  are 32-bit + Crinkler; x64 has a higher size floor but the same process)
- A **wow64-crinkler** track when the user wants TinyRetroPad-class sizes
- Toolchain notes for VS 2022 Build Tools without MASM32
- Windows 11 specifics (SmartScreen, Defender, DPI, dark title bar)
- Growth-log examples and anti-patterns (no Electron, no WinUI, no AV disable scripts)

None of that replaces or supersedes Dave's repos. Use them as the canonical
source code references.

## Licenses

| Work | License |
|------|---------|
| This repository (skill docs) | MIT — see [LICENSE](../LICENSE) |
| TinyRetroPad | Apache License 2.0 |
| HelloAssembly | See repository LICENSE file |
| Crinkler | See repository LICENSE file |

When you build apps using this skill, your assembly source is your own. The
methodology is Dave's; the skill packaging is this repo's.

## How to cite

If you reference this skill in a blog post or repo:

> Tinyapp Cursor Skill — documents Dave Plummer's tiny-notepad process from
> Dave's Garage. https://github.com/calebjeffery/cursor-tinyapp-skill

If you reference the original work:

> Dave Plummer, HelloAssembly / TinyRetroPad, PlummersSoftwareLLC.
> https://github.com/PlummersSoftwareLLC/TinyRetroPad
