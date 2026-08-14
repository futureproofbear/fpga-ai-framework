# Rule: verify by value, never by correlation alone

Correlation/magnitude comparison is scale-, phase-, and orientation-invariant — it passes even when
a transform is conjugated, bin-reversed, transposed, or per-row mis-scaled. Feed known inputs and
diff the actual output VALUES against a bit-accurate model, element by element, and report where
divergence starts. Build the bit-accurate model and match it to a float/golden reference FIRST —
only trust a hardware-vs-model divergence once the model itself is proven correct.

**Watch for the orientation trap** before ever calling a divergence real: hardware output is
frequently the golden reference transposed / flipped / offset, not simply wrong — run an exhaustive
orientation scan before concluding a real bug exists.

**Why:** a correlation-only or magnitude-only check has, in practice, both passed on silently wrong
output and failed on correct output at the wrong orientation — either way it wastes a debugging
cycle chasing a phantom.

Full methodology + phase-exact board-free check: `.claude/skills/value-level-verification/`.
