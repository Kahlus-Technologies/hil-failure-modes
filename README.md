# HIL Test Plan: Six Embedded Failure Modes

## Test status

Protocols are drafted. Bench runs are next; measured traces and run notes will be added with each test.

## Test cases

Six hardware-in-the-loop cases for failures that can pass compilation, lint, and simulation but appear on real microcontrollers. The cases focus on timing, hardware state, and silicon behavior:

1. [Peripheral init ordering](docs/01-peripheral-init-ordering.md)
2. [Shared-bus timing/race (I2C multi-device)](docs/02-shared-bus-race.md)
3. [Brownout / power-on-reset sequencing](docs/03-brownout-sequencing.md)
4. [Interrupt priority inversion](docs/04-interrupt-priority-inversion.md)
5. [DMA aliasing / cache coherency](docs/05-dma-cache-aliasing.md)
6. [Clock config drift (PLL/wait-state mismatch)](docs/06-clock-config-drift.md)

## Bench setup

The initial setup uses two STM32 boards and instruments for digital timing and analog rail and clock measurements.

| Item | Test coverage | Rough cost |
|---|---|---|
| STM32 Nucleo-F411RE (or similar Cortex-M4, no cache) | Peripheral init, shared bus, interrupt priority, brownout | ~$15 |
| STM32 Nucleo-H743ZI2 (Cortex-M7 with D-cache) | DMA/cache coherency | ~$25 |
| 8-channel logic analyzer (Saleae-compatible, e.g. FX2LAFW-based) | Bus traffic, ISR-entry GPIO toggles, DMA-complete flags | ~$15–25 |
| Bench oscilloscope, e.g. Rigol DS1054Z (4ch, 50MHz) | Rail sag and SYSCLK/MCO frequency | ~$350, or use an available scope |

Estimated hardware cost: about $55 with scope access, or $400 including a new scope. Both Nucleo boards include an ST-Link for programming and basic debugging. A logic analyzer covers digital timing; an oscilloscope is used for rail-sag and clock measurements.

## Why this exists

This test set was written for Embedder (embedder.com), an AI agent that reads datasheets and schematics, writes embedded firmware, and runs it on hardware with debug probes, logic analyzers, and power profilers. The goal is to add physical failure cases to the HIL loop, where timing, power state, and silicon behavior determine the result.
