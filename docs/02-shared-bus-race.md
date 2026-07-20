# 2. Shared-bus timing/race (I2C multi-device)

**Status: NOT YET CAPTURED — description and reproduction plan only, no bench trace yet.**

## The bug

Two or more I2C devices on one bus, driven by firmware that treats each
transaction as independent and instantaneous. Real I2C slaves can clock-stretch
(hold SCL low to signal "not ready yet"), and back-to-back transactions to
different devices without respecting the bus's actual busy/idle state can
collide with a device still finishing its previous operation (e.g. a sensor
mid-conversion, or an EEPROM mid-write-cycle that NAKs any new transaction
until it completes). The failure is intermittent — it depends on the specific
timing of two devices' operations relative to the firmware's transaction
schedule, so it may pass every time on a fast, lightly-loaded bus and fail
only once real device latencies + real bus capacitance are in the loop.

## Why an AI agent gets this specifically wrong

- A driver written against a mocked I2C HAL (send bytes in, get bytes back,
  no timing model) has no way to exhibit — or even represent — clock
  stretching or device-busy NAKs. The unit test suite is structurally blind
  to this class of bug because the mock IS the bug's absence.
- Datasheet timing requirements (e.g. "wait 5ms after write before next
  access") are easy to miss when an agent is synthesizing a driver from a
  register map without cross-referencing every device's separate timing
  section, especially when the two devices are looked up/generated in
  separate passes.
- Even when a delay is added, a fixed `delay(5)` that happens to work in
  simulation can be wrong against the real device's actual (possibly
  variable, possibly slower at temperature/voltage extremes) timing.

## Reproduction plan

- Board: Nucleo-F411RE. Two I2C devices on one bus (e.g. an EEPROM +
  a sensor, or two of the same sensor at different addresses) — device
  choice pending what's actually on hand once bench setup starts.
- Fire back-to-back transactions to both devices with no inter-transaction
  delay, immediately after triggering an operation on one that's known to
  clock-stretch or hold busy (a write cycle on an EEPROM is a reliable one).

## What would confirm it

- Logic analyzer on SCL/SDA showing either an extended clock-stretch (SCL
  held low longer than the "ideal" transaction would suggest) or a NAK on
  the second device's address byte, correlated against the first device's
  known busy window.
- A firmware-side transaction failure/retry counter that increments under
  back-to-back load but reads zero under the same test with an inter-transaction
  delay inserted — showing the bug is timing-dependent, not logic-dependent.
