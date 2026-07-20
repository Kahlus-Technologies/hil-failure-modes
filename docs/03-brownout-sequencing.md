# 3. Brownout / power-on-reset sequencing

**Status: NOT YET CAPTURED — description and reproduction plan only, no bench trace yet.**

## The bug

Firmware that either (a) leaves the MCU's brownout-detection threshold at a
default that doesn't match the actual supply's worst-case sag, or (b) starts
doing supply-sensitive work (a flash write, an EEPROM write, bringing up a
peripheral that draws a current spike) without accounting for the possibility
that a marginal supply — a long cable, a shared rail with another load
switching on, a battery under load — dips below the level the MCU needs to
guarantee correct operation, but not below the reset threshold. The MCU
doesn't reset (so nothing "obviously" happened) but a flash write in progress
during the dip can corrupt the sector being written, or a register write can
silently fail to latch.

## Why an AI agent gets this specifically wrong

- Brownout threshold selection isn't a code-correctness question the
  compiler/linter has any way to check — it's a question of "does this
  numeric threshold match this specific board's specific supply
  characteristics," which is a property of the physical circuit, not the
  source.
- LLM-generated init code commonly either omits brownout config entirely
  (leaving silicon defaults, which may not match the design) or copies a
  threshold value from a reference/eval-board example without re-deriving
  it against the actual target board's regulator and worst-case load.
- The failure is rare and load-dependent (a marginal supply dip during
  exactly the wrong instruction), which is exactly the profile of a bug
  that won't show up in a bench-top "power looks fine on the meter" check
  but will show up in the field.

## Reproduction plan

- Board: Nucleo-F411RE, powered via a bench supply (not the onboard
  ST-Link USB power) so the supply voltage can be deliberately manipulated.
- Trigger a flash write (e.g. writing a config value to a flash sector)
  while injecting a controlled voltage dip on VDD timed to land mid-write.
- Compare firmware with brownout detection configured at a threshold that
  covers the injected dip vs. left at a threshold that doesn't.

## What would confirm it

- Oscilloscope on VDD showing the injected dip's depth/duration relative to
  the MCU's datasheet-specified brownout thresholds and minimum operating
  voltage.
- A CRC/checksum readback of the flash sector after the write: intact under
  the correctly-configured threshold, corrupted under the misconfigured one,
  for the *same* injected dip — isolating the variable to firmware
  configuration rather than supply behavior.
