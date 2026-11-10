# Context Management Rules

Maintain project continuity across context compaction and
separate sessions.

## Project Layout

The end product lives in `src/`. Everything that is built,
run, or shipped (product source code and the assets it needs
at runtime) goes there and nowhere else.

- `src/`: the end product.
- `tests/`: tests for the code in `src/`.
- `scripts/`: helper and tooling scripts that support
  development but are not part of the product.
- `docs/`: documentation only, no product code.

Do not put product code in the repository root or in any
other directory. Work is not deliverable until it is in
`src/`.

## Project State

Keep `docs/agent/PROJECT_STATE.md` current at all times (not
just before compaction, which may happen without warning).
Update it when a task starts or finishes, a decision is made,
a bug or blocker is found, or requirements change.

It must always contain:
1. Current objective.
2. Important architectural decisions.
3. Completed work.
4. Work in progress (preserve unfinished work).
5. Outstanding tasks.
6. Known issues and blockers.
7. Important files and their purposes.

Never mark a task complete without verification.

## Session Start / After Compaction

1. Read `PROJECT_STATE.md` and review `docs/architecture/`.
2. Check `git status`, the current branch, `git worktree list`,
   and recent commits.
3. Re-establish the objective, outstanding tasks, last
   completed milestone, and next action.
4. Continue from the last verified state. Do not repeat
   completed work unless verification shows it is necessary.

## Version Control (Git)

1. Track all work in git. If not a repo: `git init -b main`.
2. Keep `main` working and buildable.
3. Commit as you go: small, logical commits whose messages say
   what changed and why.
4. Use `feature/<name>` or `fix/<name>` branches for
   non-trivial or experimental work; merge to `main` once
   verified.
5. Use `git worktree` for parallel branches or isolated
   experiments; remove worktrees once merged or abandoned.
6. Never commit build artifacts, temp files, or secrets;
   maintain `.gitignore`.
7. Never rewrite or discard history on `main`.

## Architecture Documentation

Record decisions in `docs/architecture/`, one Markdown file
per topic (e.g. `overview.md`, `interfaces.md`, `data.md`,
`build.md`, `decisions.md`). Cover what applies:
- Purpose, features, and domain rules or behavior.
- User-facing interfaces (UI flows, CLI, API, public library
  surface).
- Architecture: components, data flow, data models and
  storage, dependencies, configuration, and platform-specific
  concerns.
- Build, test, and deployment.
- Key decisions, with alternatives considered and why.

Update the docs in the same change as any code that alters
documented behavior.

## Task Completion

A task is done only when it is verified, the product code it
produced is in `src/`, `PROJECT_STATE.md` and
`docs/architecture/` are updated, and code and docs are
committed together on the appropriate branch.

## Memory

Save durable decisions and project-specific knowledge to
Qwen's persistent memory. Do not save temporary debugging
output or transient details.

## Long-Running Commands

Use background execution (`is_background: true`, a tool
parameter, not a shell argument) for commands expected to take
over 30 seconds, when the tool supports it: servers, watchers,
long test suites, benchmarks, polling loops, large data jobs,
and long builds. Run short commands in the foreground.

1. Check background tasks with `/tasks`. Background does not
   mean fire-and-forget: wait for completion when the result
   is needed.
2. Inspect output, errors, and exit codes before declaring
   success.
3. Never restart a stateful operation just because its
   foreground run timed out or warned.
4. For deployments, migrations, pushes, and other side-effecting
   operations, verify the original run's status before retrying.

## Reverse-Engineering Tools (MCP)

The ROM to use is located in the `rom/` folder.

- **rea** MCP — static analysis/decompilation. Use the `ghidra`
  provider with `target-format: raw-binary`, `architecture: arm`,
  `entry-point: 0x08000000` (GBA ROM base). Hopper cannot open raw
  GBA ROMs through rea. First Ghidra analysis is slow; use a
  snapshot and background execution.
- **mgba** MCP — live emulation (memory read/write, input,
  screenshots, savestates). Only works while mGBA is open with the
  ROM loaded and `~/tools/mcp-mgba/lua/bridge.lua` running
  (Tools → Scripting → Load script).
- **gearboy** MCP — Game Boy/Color emulator only. Do **not** use it
  for this GBA ROM.
- CLI fallbacks: `radare2`/`rizin` (`-a arm -b 16 -m 0x08000000`).

### Other Systems

The specific values in the bullets above (`architecture: arm`,
`0x08000000`, mGBA) are the GBA settings. The ROM in `rom/` may
be for another system. Identify the system first from the file
extension and header, then use that system's entry below. Never
assume GBA, and never load a ROM into an emulator built for a
different system.

Static analysis, any system:

- rea raw-binary accepts only `architecture` `arm`, `6502`,
  `m68k`, `x86`, `x86_64`, or `arm64`, and maps file offset 0
  at `entry-point`. Strip cartridge headers and extract
  container formats into their code binaries before loading.
- For CPUs rea cannot load, run Ghidra headless:
  `~/tools/ghidra_12.1.4_PUBLIC/support/analyzeHeadless <dir>
  <name> -import <file> -loader BinaryLoader -loader-baseAddr
  <addr> -processor <language-id>`. Add `-loader-fileOffset
  <offset>` to skip a header. The installed Ghidra has no SM83
  (Game Boy) or 65816 (SNES) language.
- `radare2`/`rizin` cover every CPU below: `-a <arch> -b <bits>
  -m <addr>`, plus `-e cfg.bigendian=true` for big-endian CPUs.

Live emulation: only GB/GBC (gearboy) and GBA (mgba) have an
emulator MCP. For every other system, do static analysis only
and tell the user that live inspection is unavailable.

Per system:

- **GB / GBC** (`.gb`, `.gbc`) — Sharp SM83. Bank 0 at `0x0000`,
  switchable bank at `0x4000`, entry `0x0100`. Static:
  `radare2 -a gb` only. Live: gearboy MCP.
- **GBA** (`.gba`) — the bullets above.
- **NDS** (`.nds`) — ARM946E-S (ARM9, main) plus ARM7TDMI (ARM7,
  sub). The file is a container, so never load it whole. Header
  fields give each binary's file offset, entry, load address,
  and size: ARM9 at `0x20`/`0x24`/`0x28`/`0x2C`, ARM7 at
  `0x30`/`0x34`/`0x38`/`0x3C`. Carve each binary and load it
  with rea `architecture: arm` and `entry-point` set to its
  load address (ARM9 is usually `0x02000000`). Retail ARM9
  binaries are often BLZ-compressed, and more code lives in
  overlays (ARM9 overlay table offset/size at `0x50`/`0x54`).
  Ghidra languages: `ARM:LE:32:v5t` (ARM9), `ARM:LE:32:v4t`
  (ARM7).
- **NES** (`.nes`) — Ricoh 2A03 (NMOS 6502). Skip the 16-byte
  iNES header. PRG ROM maps into `0x8000` to `0xFFFF`, with the
  reset vector at `0xFFFC`. rea `architecture: 6502` loads one
  PRG image (16 KiB at `0xC000` or 32 KiB at `0x8000`) and does
  not model mappers, so load banks separately. Ghidra language:
  `6502:LE:16:default`.
- **SNES** (`.sfc`, `.smc`) — Ricoh 5A22 (65C816), LoROM or
  HiROM mapping. `.smc` may carry a 512-byte copier header.
  Static: `radare2 -a snes` only.
- **Genesis / Mega Drive** (`.md`, `.gen`, `.bin`) — Motorola
  68000, big-endian, ROM at `0x000000`. rea `architecture:
  m68k`, `entry-point: 0x0`; it does not follow the vector
  table, so read the reset PC from offset `0x4` yourself.
  Ghidra language: `68000:BE:32:default`.
- **N64** (`.z64`, `.v64`, `.n64`) — NEC VR4300 (MIPS),
  big-endian. Convert `.v64`/`.n64` byte order to `.z64` first.
  Entry address at header `0x08` (commonly `0x80000400`); game
  code starts at file offset `0x1000`. Ghidra language:
  `MIPS:BE:64:64-32addr`.
- **Any other system** — look up its CPU, endianness, load
  address, and container format before loading anything, then
  apply the static-analysis rules above.
