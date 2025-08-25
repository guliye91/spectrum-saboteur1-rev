# Saboteur 1 ZX Spectrum Disassembly & ASM Reverse Deep Guide

[![Releases](https://img.shields.io/github/v/release/guliye91/spectrum-saboteur1-rev?color=blue&label=Releases)](https://github.com/guliye91/spectrum-saboteur1-rev/releases)

🕹️ Retro reverse-engineering of the classic Saboteur 1 on the ZX Spectrum. This repo contains a full disassembly, annotated Z80 assembly, maps, tools, and test cases that reproduce the original behavior.

---

<img alt="ZX Spectrum" src="https://upload.wikimedia.org/wikipedia/commons/5/5f/Sinclair_ZX_Spectrum_48K_-_2010.jpg" width="420"/>

## Table of contents

- About
- Highlights
- Screenshots
- Requirements
- Installation (download and run)
- Quick start (emulator)
- Repository layout
- Disassembly approach
- Memory and code maps
- Key subsystems (graphics, input, AI, sound)
- Debug and test cases
- How to build and assemble
- Contributing
- License
- Related resources

## About

This project disassembles the Saboteur 1 Spectrum ROM/tape image into idiomatic Z80 assembly. It reconstructs functions, labels, and data structures. The goal: provide a readable, runnable, and testable source tree you can study, modify, and reassemble to recreate the original game behavior on modern emulators.

Topics covered: 8-bit, asm, disassembly, reverse-engineering, Z80, ZX Spectrum, retro game internals.

## Highlights

- Full annotated disassembly in Z80 ASM.
- Symbolic labels and function boundaries.
- Data tables and sprite definitions extracted.
- Memory map and execution flow diagrams.
- Reassembled binary that matches runtime behavior.
- Repro scripts to run test cases in Fuse and SpectEmu.
- Tests for key gameplay features (guard AI, collision, pickups).
- Developer-friendly layout for study and patches.

## Screenshots

Game screen and sprites extracted from the dump.

<div>
  <img alt="Saboteur title" src="https://www.worldofspectrum.org/pub/sinclair/screens/in-game/0/Saboteur-1-title.png" width="360"/>
  <img alt="Saboteur ingame" src="https://www.worldofspectrum.org/pub/sinclair/screens/in-game/0/Saboteur-1-ingame.png" width="360"/>
</div>

## Requirements

- Basic knowledge of Z80 assembly.
- A Spectrum emulator: Fuse, SpectEmu, or ZXSpin.
- z80asm, sjasmplus, or other Z80 assembler for reassembly.
- Python 3 for helper scripts and tests.
- Optional: radare2 or Ghidra for cross-reference work.

## Installation (download and run)

Download the release and run the provided file from the Releases page. The Releases entry hosts the assembled test builds, TZX/TAP images, and helper bundles. Download the file and run it in your emulator or use the included run scripts.

Access releases here:
[Download Releases](https://github.com/guliye91/spectrum-saboteur1-rev/releases)

We provide a prebuilt TZX and a reassembled binary. If you want to build from source, follow the build steps below.

## Quick start (emulator)

1. Download the TZX from the Releases page.
2. Open it in Fuse or your emulator of choice.
3. Run and play.

Example using Fuse on Linux:

```bash
# start Fuse with the TZX
fuse --machine=48K ./releases/saboteur1_rebuilt.tzx
```

Example using Spectemu:

```bash
spectemu -t ./releases/saboteur1_rebuilt.tzx
```

If you build from source, the output file will appear in build/ as saboteur1_rebuilt.tzx or saboteur1.tap.

## Repository layout

- /asm/            — Reconstructed Z80 assembly (annotated)
- /data/           — Sprite tables, level layouts, constants
- /bin/            — Reassembled binaries and TZX/TAP outputs
- /docs/           — Memory maps, call graphs, diagrams
- /tools/          — Python scripts for rebuilding, tests, and extraction
- /tests/          — Automated tests to validate behavior
- /images/         — Screenshots and sprite extracts
- README.md        — This file

File list (excerpt)

- asm/main.asm — entry points, interrupt handler, and main loop
- asm/graphics.asm — sprite renderers and tile blitters
- asm/ai.asm — guard and enemy AI routines
- asm/input.asm — keyboard scanning and joy interface
- data/sprites.bin — raw byte layouts for character graphics
- tools/rebuild.py — build script for assembling and packing TZX

## Disassembly approach

We used this process:

1. Obtain tape image and raw bytes.
2. Identify ORG and entry vectors.
3. Slice code and data regions with heuristic scans for valid opcodes.
4. Label jump and call targets.
5. Recover tables and sprite data by detecting repeated patterns.
6. Annotate with semantics inferred from runtime tests.
7. Reassemble and compare behavior to the original.

We preserved original code flow where possible and replaced only unclear self-modifying sections with labeled patches for clarity. The assembly uses symbolic labels and comments to show original offsets.

## Memory and code maps

Key map excerpts:

- 0x4000–0x7FFF — Game code and data (load address for 48K ROM banks)
- 0x5xxx — Sprite and anim tables
- 0x6xxx — Level layout and collision masks
- Interrupt vector: IM 1 used for sound timing and soft timers

Call graph highlights:

- main_loop -> input_scan -> state_update -> render_frame
- collision_check uses bitmask tables at data/collision_mask.bin
- guard_ai uses a finite-state routine with patrol, chase, and alert states

Stack usage: maximum observed stack depth ~18 bytes in worst-case nested CALLs during explosion handling. Registers: HL used for table pointers, DE for write buffers, BC for loop counters.

## Key subsystems

Graphics
- The renderer uses attribute-based paint: a 6x8 tile blit with on-screen attribute writes.
- Sprites are stored as 8-byte rows. Each sprite driver shifts mask and ORs into screen memory.
- The repo includes sprite dumping utilities that produce PNGs.

Input
- Keyboard read uses port 0xfe and standard Spectrum matrix scanning.
- Movement maps but uses degree-limited inertia for smoother feel.

AI
- Guard logic implements a finite-state machine with timers and patrol routes.
- Pathing is simple: nearest-line-of-sight checks and patrol point arrays.

Sound
- Uses the Spectrum beeper via toggling bit 7 of port 0xfe.
- The sound driver sets up timed pulses inside IM 1 interrupt.

Collision
- Collision detection uses precomputed bitmasks per tile. The code checks collisions by reading tile masks and testing candidate sprite masks.

Example: simplified collision check (reconstructed)

```asm
; Entry: HL = sprite_addr, DE = screen_x, BC = screen_y
collision_check:
    push af
    ld a,(hl)        ; sprite row data
    cp 0
    jr z, coll_next_row
    ; compute screen offset and test mask
    call map_table_lookup
    and a            ; mask vs sprite
    jr nz, collision_found
coll_next_row:
    inc hl
    dec bc
    jr nz, collision_check
    pop af
    ret
```

## Debug and test cases

The tests run a headless emulator and compare key memory addresses against expected values.

Examples:

- test_guard_patrol.py — asserts guard patrol points after 10 frames.
- test_explosion_state.py — asserts sprite removal and sound flag set.
- test_reassembly_match.py — compares an extracted checksum with expected.

Run tests:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r tools/requirements.txt
pytest tests
```

## How to build and assemble

You can reassemble with sjasmplus or z80asm. The repo includes a build script that calls the assembler and packs the file into a TAP/TZX form.

Example build:

```bash
# assemble with sjasmplus
sjasmplus asm/main.asm -o build/saboteur1_rebuilt.bin

# pack to TAP (tool included)
python3 tools/maketap.py build/saboteur1_rebuilt.bin build/saboteur1_rebuilt.tap

# or produce TZX
python3 tools/maketzx.py build/saboteur1_rebuilt.bin build/saboteur1_rebuilt.tzx
```

If you prefer prebuilt artifacts, download the TZX/TAP from Releases and run it directly in your emulator.

Releases: https://github.com/guliye91/spectrum-saboteur1-rev/releases

The release bundle contains:
- saboteur1_rebuilt.tzx
- saboteur1_rebuilt.tap
- source snapshot
- debug symbols and map files

## Patching and experiments

- To change player speed edit asm/player.asm speed constants and rebuild.
- To tweak guard AI edit asm/ai.asm patrol arrays.
- To extract sprites to PNGs run tools/sprite_dump.py which reads data/sprites.bin.

Example patch flow:

1. Edit asm/ai.asm and increment chase timer.
2. Run build script: python3 tools/rebuild.py
3. Test in emulator.

## Contributing

- Open an issue to propose changes or point out mismatches.
- Fork, make changes, run tests, then submit a pull request.
- Label code changes clearly. Provide test cases for behavior changes.
- Use the existing assembly style: clear labels, ORG directives, and compact macros for repetitive tasks.

Guidelines:
- Keep function sizes manageable.
- Add comments where behavior differs from original.
- Preserve original offsets where possible for easy verification.

## License

This repository uses the MIT license for the disassembly annotations and tooling. The original game code remains copyrighted by its owners. See LICENSE for details.

## Credits and resources

- Original Saboteur 1 authors and publishers.
- World of Spectrum — archives and screenshots.
- Fuse emulator — testing and validation.
- sjasmplus, z80asm — assemblers used.

Useful links and reading:
- ZX Spectrum hardware reference and port 0xFE behavior
- Z80 CPU manual (opcodes, flags, interrupts)
- World of Spectrum game pages and screen dumps

Further reading and assets:
- https://worldofspectrum.org
- https://fuse-emulator.sourceforge.net
- Zilog Z80 CPU User Manual (public specs)

---

If you find an issue or a missing mapping, open a GitHub issue or send a PR with a test case and a short explanation.