---
name: mcu-cpp-kiss
description: Simplify and debloat memory-constrained MCU C++ firmware. Use for KISS refactors that reduce flash/RAM/stack/latency or remove over-engineering, template/STL/runtime bloat, dynamic allocation, unused drivers, or complex IRQ/DMA/EEPROM abstractions while preserving behavior.
---

# MCU C++ KISS

Simplify MCU C++ firmware without changing behavior. Bias toward small, explicit, measurable changes. For nontrivial work, read `references/mcu_cpp_kiss_refactor_guide.md`.

## Workflow

1. Establish the target: flash reduction, RAM reduction, stack reduction, latency reduction, readability, or all of these.
2. Inspect build context, linker scripts, drivers, IRQs, DMA paths, config/storage, map files, and size reports.
3. Measure first when possible:
   - `arm-none-eabi-size firmware.elf`
   - `arm-none-eabi-nm --size-sort -S firmware.elf`
   - linker map file
   - stack usage output if available
4. Attack measured or obvious bloat first: exceptions, RTTI, iostreams, float formatting, dynamic allocation, heavy templates, duplicated drivers/state machines, unused HAL/peripherals, large logs/strings/tables, generic parsers/formatters.
5. Make the smallest behavior-preserving change, rebuild, compare size/behavior, and report deltas plus risks.

## KISS Rules

- Prefer one obvious control path over layers of indirection.
- Prefer cohesive functions with a clear job; do not split straight-line hardware logic into many tiny helpers unless it removes duplication, clarifies ownership/timing, or enables meaningful tests.
- Prefer fixed-size storage and explicit ownership over generic containers.
- Prefer plain functions and small structs over virtual hierarchies in driver code.
- Prefer simple state machines over callback chains when timing and ownership matter.
- Prefer thin templates over large templated implementations.
- Prefer integer/fixed-point math over floating point unless hardware and requirements justify it.
- Prefer compile-time constants only when they reduce code or clarify invariants.
- Remove unused configurability instead of preserving theoretical extension points.
- Keep IRQ/DMA paths direct, bounded, and easy to audit.

Do not:

- Rewrite large subsystems for style.
- Over-split straight-line hardware code into one-use helpers.
- Hide timing, ownership, or register access behind abstractions.
- Merge unrelated modules when coupling or ownership clarity worsens.
- Remove validation, CRCs, bounds checks, hardware acknowledgements, or error handling unless the user accepts the safety tradeoff.
- Optimize from guesses when a map file or size report can answer the question.

## Refactor Targets

Prioritize these in MCU C++ projects:

- Replace templated heavy code with a tiny template wrapper over one non-template implementation.
- Replace virtual interfaces in low-level drivers with compile-time selection, function pointers, or direct calls where appropriate.
- Replace string formatting/logging with fixed event IDs, counters, or compile-time gated logs.
- Replace scattered EEPROM parameter access with a central parameter store and per-module sub-structs.
- Remove unused peripheral drivers from the build instead of only disabling them at runtime.
- Collapse duplicated state machines only when the shared implementation is smaller and clearer.
- Inline or keep local simple one-use helpers when extraction would add call overhead, code size, or navigation cost without reducing real duplication.
- Move large lookup tables behind measurement: keep only if smaller/faster than computation.
- Convert heap-backed code to caller-owned buffers or static pools.

## Output

- a concise summary of simplifications
- before/after size numbers if available
- tests or builds run
- any behavior intentionally preserved
- any risks, especially IRQ latency, DMA ownership, EEPROM compatibility, or stack growth

When only reviewing or planning, lead with the highest-value simplification opportunities and cite files/lines where possible.
