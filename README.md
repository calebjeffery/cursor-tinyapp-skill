# Cursor Tinyapp Skill

A [Cursor Agent Skill](https://cursor.com/docs/agent/skills) for building **tiny native Windows 11 desktop apps** using the process Dave Plummer popularized on [**Dave's Garage**](https://www.youtube.com/c/davesgarage): wrap what Windows already ships, add one feature at a time, and log the byte cost after every rebuild.

This repository shares the skill files and documents how the skill was created, with full credit to Dave Plummer and the open-source lineage that inspired it.

## What this skill teaches

The agent learns to build complete, usable Win32 apps without Electron, .NET, Qt, or a C runtime:

- **Default track (`win11-x64`)** — native 64-bit assembly with `ml64` + VS 2022 Build Tools (typically a few KB to ~20 KB)
- **Size-record track (`wow64-crinkler`)** — 32-bit WoW64 + Crinkler, matching [TinyRetroPad](https://github.com/PlummersSoftwareLLC/TinyRetroPad) sizes (hundreds of bytes to ~3 KB)

Core rules from Dave's process:

1. Name the OS feature, not the app — wrap `RICHEDIT50W`, common dialogs, and system controls
2. Scaffold a **complete window** before chasing bytes
3. Add **one feature per rebuild** and record EXE size in a growth log
4. No CRT, no bundled UI engine, no `.rc` resources unless unavoidable

## Install

### Personal skill (recommended)

Copy the `skill/` folder into your Cursor skills directory:

```powershell
# Windows
Copy-Item -Recurse -Force ".\skill" "$env:USERPROFILE\.cursor\skills\tinyapp"
```

```bash
# macOS / Linux (if you edit the skill on another machine)
cp -r skill ~/.cursor/skills/tinyapp
```

Then invoke it in Cursor by mentioning **tinyapp**, **Dave's Garage tiny notepad**, **TinyRetroPad**, or similar terms.

### Project skill

To share with a team repo, copy into `.cursor/skills/tinyapp/` at the project root instead.

## Repository layout

```
cursor-tinyapp-skill/
├── README.md                 # You are here
├── LICENSE                   # MIT (skill documentation)
├── skill/
│   ├── SKILL.md              # Main agent instructions
│   ├── reference.md          # Toolchain, ABI, build recipes
│   └── examples.md           # Feature growth order and patterns
└── docs/
    ├── ATTRIBUTION.md        # Credit to Dave Plummer and lineage
    └── CREATION-PROCESS.md   # How this Cursor skill was built
```

## Credit (summary)

This skill is **not** Dave Plummer's code. It is a Cursor Agent Skill that **documents and teaches his methodology**, adapted for Windows 11 Pro 64-bit.

| Project | Author | Role |
|---------|--------|------|
| [HelloAssembly](https://github.com/PlummersSoftwareLLC/HelloAssembly) | [Dave Plummer](https://github.com/davepl) | Original `tiny.asm` foundation |
| Dave's Tiny Editor (DTE) | Matt Power | Sub-1 KB RICHEDIT editor extension |
| [TinyRetroPad](https://github.com/PlummersSoftwareLLC/TinyRetroPad) | Dave Plummer et al. | Full Notepad-style menu set (~2.5 KB) |

Key Dave's Garage videos:

- [Hello, Assembly!](https://youtu.be/b0zxIfJJLAY)
- [C vs ASM](https://youtu.be/-Vw-ONPfaFk)
- [The Challenge: Can we build Notepad in 3K in assembly language?](https://www.youtube.com/watch?v=OG91c7xsNMc)

Full attribution: [docs/ATTRIBUTION.md](docs/ATTRIBUTION.md)

## Prerequisites (for apps built with this skill)

- Windows 11 Pro 64-bit
- [Visual Studio 2022 Build Tools](https://visualstudio.microsoft.com/downloads/) with C++ build tools and Windows 11 SDK
- Optional (wow64-crinkler track only): [Crinkler](https://github.com/runestubbe/Crinkler)

## License

The skill documentation in this repository is licensed under [MIT](LICENSE).

Dave Plummer's [HelloAssembly](https://github.com/PlummersSoftwareLLC/HelloAssembly) and [TinyRetroPad](https://github.com/PlummersSoftwareLLC/TinyRetroPad) projects use their own licenses (Apache 2.0 for TinyRetroPad). This repo does not redistribute their source code.

## Contributing

Issues and pull requests welcome. If you extend the skill, keep Dave's core process intact: wrap the OS, one feature per rebuild, log bytes, readable comments.
