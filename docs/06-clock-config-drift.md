# 6. Clock config drift (PLL/wait-state mismatch)

**Stage: Protocol drafted. Next: bench validation.**

## The bug

MCU clock trees (PLL input/output dividers, VCO frequency range, system
clock source mux) have hardware constraints that are *not* enforced by the
register interface itself — a PLL configuration that's numerically valid
(every field within its bit-width range) can still violate a real constraint
like "VCO frequency must be between X and Y MHz for lock to be guaranteed"
or "flash wait-states must be increased before raising SYSCLK above a given
frequency, or fetches will be corrupted." Get the wait-state/SYSCLK
relationship wrong specifically, and the failure mode is exactly the kind
that looks like "random" instability — corrupted instruction fetches at the
new (too-fast-for-current-wait-state) clock, manifesting as anything from a
hard fault to silently wrong data, not a clean error.

## Why an AI agent gets this specifically wrong

- Every individual register write in a clock-config sequence is independently
  valid — the bug is in the *relationship* between two separately-configured
  values (SYSCLK frequency and flash wait-state count) that live in different
  registers, often set in different lines of code, sometimes even different
  functions.
- The correct wait-state count for a given SYSCLK frequency is a *table* in
  the datasheet (specific to the exact part, voltage range, and sometimes
  temperature range) — not a formula obviously derivable from the register
  map alone, making it an easy thing for an agent to miss or misread across
  a datasheet table.
- Because the failure mode is silicon timing margin, not logic — it can
  "work" on the bench at room temperature with a fresh part and fail only
  under different conditions (temperature, voltage corner, a slightly
  slower-than-typical part) — which is invisible to any simulator and even
  invisible to a first successful bench run.

## Reproduction plan

- Board: Nucleo-F411RE.
- Configure the PLL to raise SYSCLK to a frequency that requires an
  increased flash wait-state count, but deliberately leave the wait-state
  register at its previous (too-low) value.
- Run a simple, verifiable workload (e.g. a checksum computed over a fixed
  block of flash-resident data, executed in a tight loop) at the
  misconfigured clock vs. the correctly configured one.

## What would confirm it

- Oscilloscope/logic analyzer on the MCU's clock-out (MCO) pin verifying the
  actual SYSCLK frequency achieved matches the configured PLL settings (so
  the test is confirmed to be running at the intended fast clock, not
  accidentally still on the default/slow clock).
- The checksum computation producing wrong/inconsistent results (varying
  run to run) at the misconfigured wait-state count, and consistently
  correct results at the same clock frequency with the correct wait-state
  count set — isolating the discrepancy to the wait-state/frequency
  relationship specifically, not the PLL config itself.
