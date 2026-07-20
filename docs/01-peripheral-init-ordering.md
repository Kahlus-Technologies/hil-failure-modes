# 1. Peripheral init ordering

**Status: NOT YET CAPTURED — description and reproduction plan only, no bench trace yet.**

## The bug

On most Cortex-M parts (STM32 included), a peripheral's registers are
inert until its bus clock is enabled via the clock-gating register
(`RCC->AHBENR` / `APBENR` etc.). Code that configures a peripheral
(e.g. writes I2C `CR1`, or GPIO `MODER`/`AFR` for a peripheral's pins)
*before* enabling that peripheral's clock doesn't fault — it silently
writes into a bus that's not clocked, so the write is dropped or reads
back as reset-state. Nothing observably breaks until the peripheral is
used, and even then the symptom (I2C never ACKs, a timer never ticks) is
generic enough to send you looking anywhere except register-write order.

A related variant: enabling the clock but not waiting the datasheet's
required settling delay before the first register write (some clock
domains need a few cycles to actually apply).

## Why an AI agent gets this specifically wrong

This is a textbook case of "compiles clean, passes lint, fails on silicon":
- The C is syntactically and semantically valid — every register write is
  well-typed, using real bitfield names from the vendor header.
- Static analysis has no model of *hardware clock-gating state* — it can't
  know that `I2C1->CR1 = ...` is a no-op unless `RCC->APB1ENR1 |= RCC_APB1ENR1_I2C1EN`
  already ran.
- LLM-generated peripheral init code is frequently assembled by pattern-matching
  against similar-looking init sequences (or datasheet register tables read
  out of order), and clock-enable-before-configure is exactly the kind of
  ordering constraint that's implicit in vendor example code structure, not
  explicit in any single register's documentation.

## Reproduction plan

- Board: Nucleo-F411RE.
- Write two versions of an I2C1 init: (a) correct order — `RCC` clock enable,
  then GPIO AF config, then `I2C1->CR1` config; (b) swapped — `I2C1->CR1`
  config before the `RCC` enable.
- Attempt a transaction (e.g. a simple register read from any I2C sensor,
  or even just toggling `CR1_PE`) on both.

## What would confirm it

- Logic analyzer on SCL/SDA: version (a) shows a normal START condition and
  ACK/NACK; version (b) shows either no bus activity at all, or activity
  that doesn't match the configured clock speed (because the peripheral is
  running off its reset-state config, not the one the code "set").
- A register dump (via debugger) immediately after the "misordered" init
  showing `I2C1->CR1` still at its reset value despite the write.
