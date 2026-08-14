# Rule: isolate one kernel/IP, drive it directly, read its output directly

When localizing a bug in a multi-stage pipeline, don't infer "everything else works" from an
end-to-end result — isolate the smallest decisive test case for exactly one kernel/IP block, drive
it with a known input directly, and read its output directly. Pick the smallest case that still
reproduces the symptom (a single frame/burst/transaction beats a large multi-transaction run, which
is more likely to trip an unrelated resource/link wedge than to add diagnostic value).

Before any risky teardown step in a debug session (killing a stuck test harness, resetting a link),
capture the decisive pass/fail signals FIRST, so the run still yields usable data even if the
teardown itself hangs or fails.

**Why:** a large, multi-transaction diagnostic run costs more debug-session risk (wedged links,
stuck buses) than a minimal one, for no extra localizing power — the isolation, not the run size, is
what finds the bug.

OEM-specific mechanics (exact JTAG/debug-link commands): see the matching `rules/oem/<vendor>/`
iso-test-harness skill.
