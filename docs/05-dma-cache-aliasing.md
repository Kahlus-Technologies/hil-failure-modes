# 5. DMA aliasing / cache coherency

**Status: NOT YET CAPTURED — description and reproduction plan only, no bench trace yet.**

## The bug

On MCUs with a data cache (Cortex-M7 parts like the STM32H7, unlike cache-less
M4 parts like the F4 series), a DMA peripheral writes directly to SRAM,
bypassing the CPU's data cache entirely. If a buffer that DMA writes into is
also cached (the CPU previously read it, or it's in a cacheable memory
region by default), the CPU can read *stale cached data* even though DMA has
already written fresh data to the actual SRAM — because the cache doesn't
know DMA touched that address. The fix requires either placing DMA buffers in
a non-cacheable memory region (MPU-configured), or explicitly invalidating
the cache line(s) after DMA completes and before the CPU reads. The inverse
problem exists for DMA reads from a CPU-written buffer (need a cache clean,
not invalidate, before starting the DMA read).

This is the textbook "works on the M4 dev board, breaks on the H7 board" bug
— identical C code, different silicon, because the M4 has no cache to be
wrong about.

## Why an AI agent gets this specifically wrong

- Code correctness here depends entirely on which specific silicon variant
  it runs on — a cache-invalidate call that's a correct no-op on a cacheless
  part is a *required* call on a cached one, and the C code that's "wrong"
  (missing the invalidate) compiles, links, and runs on both — it just
  silently returns stale data on one of them.
- Reference/example code is frequently written and tested against whichever
  eval board was on hand (often a cacheless M4 Discovery/Nucleo board,
  since they're cheaper and more common), so an agent pattern-matching
  against "known-good" DMA examples inherits code that's silently wrong the
  moment the target MCU changes to a cached part.
- A simulator or cache-less emulation environment cannot exhibit this bug
  at all — the bug's entire existence depends on real cache hardware
  behavior that most simulators don't model faithfully (or don't model
  as a possible source of "wrong" behavior with correct-looking code).

## Reproduction plan

- Board: Nucleo-H743ZI2 (cached — where the bug manifests) vs. Nucleo-F411RE
  (cacheless — control case, same code should "work" there).
- Configure a DMA transfer (e.g. from an ADC or a peripheral into a fixed
  SRAM buffer) into a buffer placed in cacheable memory, with the CPU having
  read that buffer at least once before the DMA transfer (populating the
  cache line) — then read it again immediately after DMA completes, with
  no invalidate call.

## What would confirm it

- On the H743: the CPU-read value after DMA completion doesn't match a
  logic-analyzer/scope capture of the actual peripheral data that fed the
  DMA transfer (or doesn't match a debugger memory-view read of the
  physical SRAM address, which bypasses cache).
- On the F411 (cacheless): the same code path shows the CPU read matching
  the DMA'd data correctly — isolating the discrepancy specifically to the
  cached part, confirming it's a cache-coherency bug and not a DMA
  configuration bug.
- After adding the correct cache-invalidate call: H743 behavior matches
  F411 behavior on the same test.
