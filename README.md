# fpga-ai-framework

A version-controlled, three-tier Claude Code framework for FPGA/SoC development, meant to be
included as a **git submodule** (conventionally at `.framework/`) inside individual application
repositories rather than copy-pasted into each one.

## Why a separate repo

An application repo (RTL, testbenches, board constraints, project-specific docs) and the
methodology/toolchain knowledge that any FPGA project on the same vendor/chip can reuse are
different lifecycles. Keeping the latter in its own repo means:

- **Zero leakage** — an application's `.xdc`/`.pdc` constraints, proprietary RTL, and project specs
  never end up in here, and this framework's content never ends up baked into an application's own
  git history.
- **Bi-directional evolution** — a lesson learned while debugging one application (a Vivado
  timing-closure gotcha, a PolarFire SoC errata workaround) gets committed inside `.framework/` from
  within the application repo and pushed upstream to this repo directly, so the next application
  (on the same vendor/chip) starts with that knowledge already loaded.

## Three tiers

```
.claude/agents/    generic + OEM-specific subagent personas
.claude/skills/    generic + OEM-specific task-scoped procedural skills (SKILL.md)
rules/generic/     Tier 1 -- vendor-agnostic FPGA/SoC engineering methodology
rules/oem/         Tier 2 -- one folder per vendor toolchain (amd_vivado, microchip_libero, ...)
rules/target/      Tier 3 -- one folder per vendor, one sub-folder per chip FAMILY (not board)
templates/         boilerplate build scripts / IP wrappers / TB harnesses, added as they prove out
```

- **Tier 1 (`rules/generic/`, vendor-agnostic `.claude/agents`+`.claude/skills`)** — true for any
  FPGA/SoC project regardless of vendor: reference-first verification, HLS-output distrust,
  value-level verification, kernel-isolation testing, hardware-aware architectural critique.
- **Tier 2 (`rules/oem/<vendor>_<toolchain>/`, OEM-specific agents/skills)** — true for any project
  on that vendor's toolchain, not tied to one exact chip: Vivado / Vitis HLS / Vivado Hardware
  Manager mechanics for AMD-Xilinx; Libero SoC / SmartHLS / SmartDebug / FlashPro6 mechanics for
  Microchip.
- **Tier 3 (`rules/target/<vendor>/<chip_family>/`)** — true only for one chip family/die
  revision, not the toolchain in general and not any one board: Kintex UltraScale's GTH-only
  transceiver architecture, MPFS250T_ES engineering-sample silicon errata.

A board (a specific PCB with a specific set of connectors, clock generators, and populated part)
is deliberately **not** a tier here — board-specific pinouts/bring-up facts belong in the
*application* repo, since a board isn't reusable across projects the way a chip family or a
toolchain is.

## What's in it today

**Design side** (write it right the first time):
- `rules/generic/hdl-style.md` — synthesis/simulation agreement, reset discipline, state machines,
  parameterization, handshake contracts.
- `rules/generic/cdc-guidelines.md` — the four crossing types and the correct structure for each;
  why a timing PASS is not a CDC proof.
- `rules/generic/timing-closure-basics.md` — the build gate, how gates fail silently, reading a
  failing path before changing anything.
- `.claude/agents/generic-rtl-architect.md`, `.claude/agents/generic-tb-specialist.md` — the
  authoring personas.
- `.claude/skills/cdc-checker/` — the CDC audit workflow.

**Verification / debug side** (find out what actually happened):
- `rules/generic/{reference-first-verification,value-level-verification,hls-output-distrust,
  kernel-isolation-testing}.md` and their paired skills.
- `.claude/agents/{architectural-critic,fpga-ref-verifier,ingestion-triage,synthesis-repair,
  doc-accuracy}.md`.

**OEM tier** — `rules/oem/amd_vivado/` + `.claude/agents/vivado-*` + `.claude/skills/{vivado-*,
vitis-hls-kernel-authoring}/`; `rules/oem/microchip_libero/` + `.claude/agents/{libero-build,
microchip-*}` + `.claude/skills/{smarthls-kernel-authoring,smartdebug-active-probe,
microchip-iso-test-harness,flashpro6-jtag-recovery}/`.

**Chip-family tier** — `rules/target/amd/kintex_ultrascale/`,
`rules/target/microchip/polarfire_mpfs250t/`.

**Maintenance** — `.claude/agents/framework-curator.md` sweeps a consuming project for reusable
findings that weren't captured in the moment, classifies them against the tier rules below, and
writes the accepted ones in.

**Project workflow** — the `openspec-*` skills and `/opsx:*` commands (generic spec-driven change
management; requires the separate `openspec` CLI).

## Using this in an application repo

```bash
git submodule add <this-repo-url> .framework
```

Then in the application's own `CLAUDE.md`, declare the hardware target and pull in exactly the
rule files that apply — see the template in an application repo's own root `CLAUDE.md` for the
`@.framework/rules/...` include pattern. Symlink the specific agent/skill folders you need from
`.framework/.claude/{agents,skills}/` into the application's own `.claude/{agents,skills}/` so
Claude Code's local skill/agent discovery picks them up.

## Evolving this framework

Work happens in the application repo. When something reusable is learned (not a project-specific
detail), extract it into the right `rules/` tier or add/update a skill inside `.framework/`, then:

```bash
cd .framework
git add -A
git commit -m "..."
git push origin main
```

The application repo's own commit only ever updates the submodule pointer — the knowledge itself
lives here, once, for every application that pulls this framework in next.
