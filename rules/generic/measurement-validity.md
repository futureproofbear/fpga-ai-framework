# Rule: a measurement must be able to fail before you may believe it passed

A check that cannot distinguish "correct" from "the check did nothing" is not
evidence. Before reporting any measured result — a resource count, an accuracy
figure, a regression comparison — establish that the measurement *had the
ability to come out differently*.

This is the measurement-side counterpart to `hls-output-distrust.md`'s "a gate
must require positive evidence, never the absence of a match". That rule is
about gates in a build flow; this one is about numbers you are about to quote.

## The failure modes, all observed in practice

**1. The knob was not connected.** A parameter sweep reported two configurations
as *identical* — a plausible, publishable-looking result. The override had in
fact bound nothing: the parameter was declared on a submodule but not exposed on
the top level the override targeted, so every run used the default. A no-op knob
and a genuine null result are indistinguishable from the output alone.

> **Check:** make the tool state the value it actually used. Synthesis logs
> report parameter binding (`Parameter X bound to: 18`); read that, don't infer
> it from the result. Or probe with a deliberately absurd value that *must*
> change the outcome — if it doesn't, the knob is dead.

**2. The design under test was optimised away.** A resource sweep returned zero
multipliers and zero logic for every configuration, because a coefficient file
failed to load and the entire datapath constant-folded to nothing. Zero is a
number; it looked like data.

> **Check:** bound the plausible range in advance and reject results outside it.
> A datapath cannot use zero multipliers. Make the script refuse, loudly, rather
> than print a degenerate value as a finding.

**3. The comparison was against the wrong reference.** A reset-behaviour test
compared the second pass against a *float model* with exact equality and
reported failure — but the differences were the ordinary ±1 LSB fixed-point
quantisation the first pass also carried, and which its own tolerance absorbed.
The behaviour under test was never actually isolated.

> **Check:** compare against the reference that isolates the variable. To test
> whether state is cleared between runs, compare **implementation against
> itself** across the two runs, not implementation against a model — that
> removes every difference except the one being tested.

**4. The metric saturated instead of measuring.** A registration check reported a
perfect `0 px` shift in every band — at `0 sigma`, with an arithmetic overflow
warning. Squaring large magnitudes in float32 overflowed, the correlation peak
became `inf`, and `argmax` returned index 0. A degenerate measurement that looks
like a flawless result is the most dangerous kind.

> **Check:** report the confidence alongside the value (sigma, sample count,
> residual). A result with no spread is a broken measurement until proven
> otherwise.

## The general form

Ask, before quoting a number:

- **Could this have come out differently?** If the answer is no under any input,
  the measurement is vacuous.
- **What would a broken setup produce?** If that is indistinguishable from the
  result you have, you have not measured anything.
- **Does the tool confirm what it actually did**, rather than what it was asked
  to do? Prefer the log line that states the bound value, the loaded file, the
  applied constraint.
- **Is the value physically plausible?** Zero resources, exactly-identical
  results, perfect correlation and zero variance all warrant suspicion before
  celebration.

## Corollary: an invariant that *should not* change is itself a check

When applying a transformation that is meant to be exact — an algebraic identity,
a refactor, a width change proven lossless — the accuracy figure must come out
**unchanged**. Reporting the same number to full precision before and after is
strong evidence the transformation was applied correctly. A changed number means
a mistake, and an *improved* number usually means the baseline was wrong.

See also: `hls-output-distrust.md` (gates that pass on missing/stale input),
`reference-first-verification.md` (verifying against the right authority in the
first place).
