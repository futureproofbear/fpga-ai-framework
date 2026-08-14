---
name: cdc-checker
description: >-
  Audit a design's clock-domain crossings: enumerate the domains, find every signal that crosses,
  classify each crossing by type, verify the structurally correct synchronizer is present, and
  confirm each has the matching timing exception in the constraints. Use before a first build,
  after adding a clock/interface, or when chasing an intermittent hardware bug that timing and
  simulation both call clean. Triggers: "check CDC", "clock domain crossing", "is this crossing
  safe", "intermittent/random failure", "metastability", "async fifo review", "report_cdc".
---

# cdc-checker

A structural audit workflow for clock-domain crossings. CDC bugs are intermittent, temperature/
voltage/lot-dependent, invisible to plain RTL simulation, and routinely coexist with a clean timing
report — so they must be caught structurally, by inspection and by a CDC tool, not by testing.

Read `rules/generic/cdc-guidelines.md` for the underlying rules; this skill is the procedure.

## Step 1 — enumerate the clock domains

List every clock in the design and where it comes from (external pin, PLL/MMCM/CCC output, a
divided/gated clock in RTL). Note for each pair whether they are:
- **Synchronous** (same source, integer ratio, known phase) — a crossing here may be timeable
  directly and may not need a synchronizer at all, but it DOES need the right timing constraint.
- **Asynchronous** (independent sources, or a non-integer ratio) — every crossing needs a
  synchronizer.

A clock the timing tool doesn't know about is not analyzed at all. Confirm each is declared.

## Step 2 — find every crossing

Grep for the structural signature rather than trusting a module list:
- Modules that take more than one clock port — these ARE CDC modules by definition.
- Signals assigned in a block clocked by A and read in a block clocked by B.
- Any instantiated async FIFO / dual-clock memory — note its depth and whether its full/empty
  handling is correct at the actual rate ratio.
- Reset nets: which domain is each reset generated in, and which domains does it reach?

Run the toolchain's own CDC report as well (Vivado `report_cdc`, or the equivalent) — it catches
structural cases (unsynchronized crossing, multi-bit into independent synchronizers) that neither
RTL simulation nor a timing report will. Treat its findings as design bugs, not advisories.

## Step 3 — classify each crossing and check the structure matches

| Crossing type | Required structure | Common wrong answer |
|---|---|---|
| Single-bit level/control | 2-flop (3 for high ratio) synchronizer in destination | none at all |
| Single-bit pulse, fast → slow | toggle at source + sync + edge-detect, or handshake | plain 2-flop (pulse can be missed entirely) |
| Multi-bit data | async FIFO, or handshake/MCP with a stable held bus | **per-bit 2-flop synchronizers** — produces values that never existed at the source |
| Gray-coded counter | per-bit sync IS safe (only one bit changes) | assuming this generalizes to arbitrary multi-bit data |
| Reset into a domain | reset synchronizer (async assert, sync de-assert) per domain | releasing a global async reset unsynchronized |

The multi-bit row is the highest-value check in this table — it is the mistake that most often
survives review, because per-bit synchronizers LOOK like correct, careful CDC code.

## Step 4 — verify the constraint matches the structure

Every crossing needs an explicit exception, and the right one:
- `set_false_path` — only for a genuinely async crossing already protected by a proper
  synchronizer. Never to silence a real failure on a path that must close.
- `set_max_delay -datapath_only` — for an MCP/handshake data bus needing bounded skew so its bits
  land within one destination cycle. **Often the correct choice where a bare false path was used**,
  because a false path lets the tool route the bus bits arbitrarily far apart.

An unconstrained crossing left to default synchronous analysis either fails spuriously (wasting a
"fix" on a non-problem) or passes silently — the second being the dangerous one. Either way, the
timing result carries no CDC information.

## Step 5 — report

For each crossing: source domain → destination domain, signal(s), crossing type, structure found,
verdict (CORRECT / WRONG STRUCTURE / UNSYNCHRONIZED / UNCONSTRAINED), and for anything not correct,
the specific fix (which structure, which constraint). Rank by severity: unsynchronized multi-bit
data first, then unsynchronized control, then missing/wrong constraints on otherwise-correct
structures.

State explicitly what you could NOT determine (e.g. a crossing inside an encrypted/black-box IP,
or a clock whose source you couldn't trace) rather than implying full coverage.

See also: `rules/generic/cdc-guidelines.md` (the rules), `rules/generic/timing-closure-basics.md`
(constraint mechanics), `.claude/agents/generic-rtl-architect.md` (writing crossings correctly in
the first place), and your OEM tier's constraint rules for vendor-specific syntax.
