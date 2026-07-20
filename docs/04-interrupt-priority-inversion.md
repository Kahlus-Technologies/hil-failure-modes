# 4. Interrupt priority inversion

**Status: NOT YET CAPTURED — description and reproduction plan only, no bench trace yet.**

## The bug

Two ISRs share a resource — a ring buffer, a shared state struct — where the
lower-priority ISR is generated/reviewed as if it were the only writer, and
the higher-priority ISR can preempt it mid-update. Without correct priority
grouping (NVIC priority levels actually reflecting which ISR must never be
interrupted mid-critical-section) or without disabling the specific
interrupt (not a blanket `__disable_irq()`, which has its own cost) around
the shared-resource access, the higher-priority ISR can observe or write the
shared structure in a torn state — a ring buffer head/tail pair updated
inconsistently, a multi-byte value read half-updated.

This is a genuine *priority inversion* variant when the low-priority ISR
holds an implicit lock (mid-update) that the high-priority ISR needs to
respect but structurally can't, because interrupts don't wait — the
high-priority one just runs, mid the low-priority one's non-atomic update.

## Why an AI agent gets this specifically wrong

- The code for each ISR, read in isolation, looks correct — this is a bug
  that ONLY exists in the *interleaving* of two functions that are each
  individually fine. Static analysis of a single function has no view of
  "what else can run between these two lines."
- An LLM generating ISR code is very likely to generate each handler
  correctly against its own spec (read sensor, push to buffer / pop from
  buffer, process) without a global view of NVIC priority configuration
  across the whole firmware image — that configuration usually lives in a
  totally separate init function generated in a different context/pass.
- Unit tests that call each ISR's underlying function directly (a common
  and otherwise reasonable way to test interrupt handlers without hardware)
  cannot represent preemption at all — they run one handler to completion,
  then the other, never interleaved.

## Reproduction plan

- Board: Nucleo-F411RE.
- A low-priority ISR (e.g. driven by a slow timer) does a multi-step,
  non-atomic update to a shared ring-buffer struct (head index, tail index,
  data write — three separate memory operations).
- A high-priority ISR (e.g. driven by a fast timer, or an external interrupt
  toggled rapidly) reads the same struct.
- Configure NVIC priorities so the "high" ISR can genuinely preempt the
  "low" one mid-update, and drive both at a rate high enough to make the
  interleaving likely within a bench-testable timeframe.

## What would confirm it

- Logic analyzer on two GPIO pins toggled at each ISR's entry/exit, showing
  an actual preemption event (high-priority ISR's entry edge landing inside
  the low-priority ISR's entry/exit window).
- A corrupted-read counter (or a checksum mismatch) on the shared struct
  that increments specifically when a logged preemption event occurred
  during the vulnerable window, and stays at zero when priorities are
  configured so preemption can't happen mid-update — isolating the bug to
  the interleaving itself, not the buffer logic.
