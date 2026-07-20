# HIL Failure Modes — bugs a hardware-in-the-loop loop needs to catch

## Status: design phase — no bench captures yet

This repo is a technical writeup, in progress. **Nothing in `docs/` right now
is a captured trace or a confirmed repro** — each doc is a stub: the bug
description and why it's real, written before touching a bench. Scope traces,
logic-analyzer captures, and logs get added to each doc as they're actually
captured on real hardware, and each stub says exactly that until they are.
No section here claims a result it doesn't have evidence for.

## What this is

Six concrete failure modes where firmware that compiles clean, passes lint,
and passes a mocked/simulated test suite still fails on real silicon —
aimed at an AI coding agent that writes embedded firmware and needs to know
what its HIL loop has to actually catch, not just what a compiler catches.

Selected for being (a) genuinely common in embedded systems work, not
contrived, and (b) specifically the kind of bug that a static/compile-time
check structurally cannot see, because the failure mode is about *timing*,
*hardware state*, or *silicon behavior* — not program logic.

1. [Peripheral init ordering](docs/01-peripheral-init-ordering.md)
2. [Shared-bus timing/race (I2C multi-device)](docs/02-shared-bus-race.md)
3. [Brownout / power-on-reset sequencing](docs/03-brownout-sequencing.md)
4. [Interrupt priority inversion](docs/04-interrupt-priority-inversion.md)
5. [DMA aliasing / cache coherency](docs/05-dma-cache-aliasing.md)
6. [Clock config drift (PLL/wait-state mismatch)](docs/06-clock-config-drift.md)

## Hardware this needs (not yet acquired — see below)

None of this has been reproduced on a bench yet. Minimum viable kit to
actually capture the six failure modes above:

| Item | Why | Rough cost |
|---|---|---|
| STM32 Nucleo-F411RE (or similar Cortex-M4, no cache) | peripheral-init, shared-bus, interrupt-priority, brownout repros | ~$15 |
| STM32 Nucleo-H743ZI2 (Cortex-M7, has D-cache) | the DMA/cache-aliasing bug specifically needs a cached part | ~$25 |
| 8-channel logic analyzer (Saleae-compatible clone, e.g. FX2LAFW-based) | digital timing capture — bus traffic, ISR-entry GPIO toggles, DMA-complete flags | ~$15–25 |
| Bench oscilloscope, e.g. Rigol DS1054Z (4ch, 50MHz) | analog rail capture — brownout/VDD sag, SYSCLK/MCO frequency verification | ~$350 (or substitute a scope already on hand) |

Total: **~$55 without a scope already on hand, ~$400 with one bought new.**
The two Nucleo boards ship with an onboard ST-Link (programming + basic
debug) — no separate programmer needed. A logic-analyzer clone is genuinely
sufficient here (digital timing, not analog fidelity); the scope is the one
item worth not going cheap on if buying new, since rail-sag and clock
measurements are where analog bandwidth/accuracy actually matter.

## Why this exists

A technical writeup backed by real bench work, for Embedder
(embedder.com) — an AI agent that reads datasheets/schematics, writes
embedded firmware, and runs it through hardware-in-the-loop tests driving
debug probes, logic analyzers, and power profilers. The six failure modes
above are chosen to be the concrete, common cases such a HIL loop needs to
catch that a compiler/linter/simulator structurally cannot.
