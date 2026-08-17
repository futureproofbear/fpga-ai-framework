# Rule: at an IF of exactly fs/4, the down-conversion AND half the filter are free

When a real signal is digitised at an IF of exactly `fs/4`, the whole
down-convert-and-decimate front end collapses to wiring, adds and one real
filter. This is a well-known structure, but the *second* saving is routinely
left on the table — including in a first implementation here, which cost 4×
the multipliers until it was found.

## The two savings

**1. The mixer needs no multiplier.** At `f_IF = fs/4` the down-conversion is

```
exp(-j*2*pi*(fs/4)*n/fs) = exp(-j*pi*n/2) = (-j)^n  in  {+1, -j, -1, +j}
```

— sign flips and I/Q swaps only. If the data arrives de-interleaved across `L`
parallel lanes with lane `p` carrying `x[Lm+p]`, **and `L` is a multiple of 4**,
then `n mod 4 = p mod 4` is fixed per lane, so each lane's rotation is a
build-time constant: no phase accumulator, no counter, no ROM. At `L` not a
multiple of 4 (e.g. 2) the same mixer needs a small counter — check it rather
than assuming.

**2. The filter's imaginary path collapses.** This is the one that gets missed.
Applying `(-j)^n` to a **real** input:

| n | rotation | result |
|---|---|---|
| even | ±1 | `(±r, 0)` — **purely real**, imaginary part identically 0 |
| odd | ∓j | `(0, ∓r)` — **purely imaginary**, real part identically 0 |

So after mixing, the even and odd sample phases carry *disjoint* components.
Pair that with a halfband decimator, whose polyphase split puts a full sub-filter
on one phase and a **single centre tap** (exactly 0.5) on the other, and the two
structures compose:

```
I(out) = FIR(h_even) over the even samples     <- the only multipliers
Q(out) = 0.5 * (delayed odd sample)            <- an arithmetic shift
```

**The imaginary output path costs no multiplier at all.**

## The mistake to avoid

An implementation that carries I and Q symmetrically through both polyphase
branches — the obvious structure, and what a generic complex-FIR block does —
spends **half its multipliers computing `x * 0`**. It is functionally correct, so
nothing fails; it is simply twice the size it needs to be. Synthesis will not
generally rescue you, because the zeros arrive through registers it cannot prove
are constant.

Measured on one real design (43-tap halfband, 4 outputs/clock, 20-bit
coefficients):

| implementation | multiplies/clock | DSP |
|---|---|---|
| I and Q through both branches | 22 × 4 × 2 = 176 | 172 |
| + fast-IQ (drop the ×0 paths) | 22 × 4 = 88 | — |
| + fold the symmetric FIR | 11 × 4 = 44 | **44** |

**3.9× fewer multipliers, bit-exact.**

## Stacking with coefficient symmetry

A halfband's sub-filter is symmetric, so pre-adding the mirrored sample pair lets
one multiply serve two taps — independent of, and multiplicative with, the fast-IQ
saving:

```
y = sum_{k=0}^{N-1} h[k]*x[n-k]
  = sum_{k=0}^{N/2-1} h[k]*( x[n-k] + x[n-(N-1-k)] )
```

## Discipline

- Both transformations are **exact algebraic identities**, so the accuracy figure
  against the reference model must come out **unchanged** — that invariance is
  the check that they were applied correctly (see `measurement-validity.md`).
- **Assert the precondition in the RTL.** The whole structure depends on
  `f_IF == fs/4` exactly. A collect or mode at a different IF invalidates it
  silently, producing a plausible but wrong spectrum. Fail loudly at elaboration
  rather than trusting a comment.
- **Note the group delay.** A type-I halfband (length `4q+3`) has a group delay
  of `q + 1/2` output samples — *always* half a sample, which cannot be removed
  by an integer slice. Book it (fold it into a downstream stored spectrum as a
  phase ramp, or into a delay calibration); do not discover it later as a
  constant range/time offset.
