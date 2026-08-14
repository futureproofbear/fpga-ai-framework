# CLAUDE.md — fpga-ai-framework

This repo is consumed as a git submodule (conventionally `.framework/`) inside an application
repo. It carries no project-specific RTL, constraints, or specs — only reusable methodology,
toolchain mechanics, and chip-family facts. See `README.md` for the three-tier layout.

## How an application repo uses this

The application's own root `CLAUDE.md` should declare its hardware target and then explicitly
include the rule files that apply, e.g.:

```markdown
## Hardware Target Definition
- **Vendor / OEM:** AMD Xilinx (Vivado 2023.2)
- **Device Family:** Kintex UltraScale
- **Target Part:** xcku115-flvb2104-2-e

## Active Framework Context
- @.framework/rules/generic/reference-first-verification.md
- @.framework/rules/generic/value-level-verification.md
- @.framework/rules/oem/amd_vivado/toolchain_gotchas.md
- @.framework/rules/target/amd/kintex_ultrascale/transceiver_rules.md
```

Only include what's actually relevant to the current target — a project on Microchip PolarFire SoC
includes `rules/oem/microchip_libero/` + `rules/target/microchip/<die>/` instead, never both OEM
trees at once.

## Adding agents/skills to an application repo

Symlink individual folders (not the whole `.claude/agents`/`.claude/skills` directory, so the
application keeps room for its own project-local agents/skills alongside these):

```bash
mkdir -p .claude/agents .claude/skills
ln -s ../../.framework/.claude/agents/architectural-critic.md .claude/agents/architectural-critic.md
ln -s ../../.framework/.claude/skills/value-level-verification .claude/skills/value-level-verification
```

## Contribution discipline (read before editing this repo from inside an application checkout)

- **Tier placement is a real decision, not a formality.** Before adding something, ask: does this
  apply to any FPGA project regardless of vendor (`rules/generic`)? Any project on this vendor's
  toolchain regardless of chip (`rules/oem/<vendor>_<toolchain>`)? Or only this chip family
  regardless of board (`rules/target/<vendor>/<family>`)? If the honest answer is "only this board"
  or "only this project's architecture," it does NOT belong here — keep it in the application repo.
- **State the mechanism, not just the symptom.** A rule earns its place by explaining *why* (the
  tool/silicon behavior that causes it), not just "X happened once."
- **Don't fabricate specifics you haven't verified.** A toolchain/chip fact belongs here once it's
  actually been hit or independently confirmed (e.g. against a datasheet, `report_timing_summary`,
  `get_package_pins`) — not because it sounds plausible. If a fact is provisional, say so in the
  file rather than stating it as settled.
- **Update, don't duplicate.** If a fix belongs in an existing rule/skill file, edit that file
  rather than adding a near-duplicate one under a different name.
