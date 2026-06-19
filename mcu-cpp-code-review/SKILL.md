---
name: mcu-cpp-code-review
description: Review STM32F042/Cortex-M0 bare-metal C++ firmware, especially IRQs, 5-channel DMA, MMIO, ISR/main shared state, fixed buffers, 16/32 KB flash, 6 KB SRAM, and STM32CubeF0 HAL/LL code. Use for audits of races, DMA ownership, volatile/critical-section correctness, ISR latency, stack/heap/flash use, template/STL bloat, flash-emulated settings, and embedded C++ risks.
---

# STM32F042 C++ Code Review

Review STM32F042 C++ for interrupt/DMA correctness, deterministic behavior, and flash/RAM discipline. For nontrivial reviews or exact policy, read `references/cortex_m0_cpp_irq_dma_guidelines.md`.

## Review Workflow

1. Inspect the changed code and nearby driver/build context before judging it.
2. Identify ISRs, DMA start/complete paths, shared state, MMIO access, buffer lifetimes, and build flags.
3. Prioritize concrete defects over style; cite exact files and lines.
4. Avoid broad rewrites unless needed for a race, lifetime bug, latency risk, or size regression.

## Priorities

- **P0**: Can corrupt memory, write through an invalid DMA buffer, wedge interrupts, brick boot/startup, or cause deterministic data loss in normal operation.
- **P1**: Likely race, ISR/DMA ownership bug, interrupt latency problem, unsafe read-modify-write, stack/heap hazard, or realistic size regression.
- **P2**: Robustness issue that can become a bug: missing ISR contract, ambiguous ownership, unclear NVIC priority, narrowing/overflow risk, template/code-size risk, weak validation.
- **P3**: Minor clarity or optional cleanup; do not let these dominate.

## STM32F042 Checks

- Treat the budget as tight: 16/32 KB flash, 6 KB SRAM, Cortex-M0 up to 48 MHz, 5 DMA channels, no D-cache.
- Prefer STM32CubeF0 LL or direct register code in hot/size-critical paths; HAL is acceptable only when measured and useful.
- Do not assume RTOS APIs, BASEPRI masking, D-cache maintenance, EEPROM, or M7-style cache coherency.
- Treat `LDREX`/`STREX` lock-free designs as unavailable unless the exact target proves otherwise.
- Do not blindly disable interrupts. For shared state, first identify writer, reader, and preemption: STM32F042 has 2 priority bits, priority 0 is highest and 3 lowest. Use a minimal `PRIMASK` section only when a compound update can be observed or interrupted by a higher-priority IRQ path or by thread/ISR preemption.
- Critical sections are snapshot/update only; unmask before parsing, HAL calls, loops, waits, logging, or flash writes. Natural-width single loads/stores are the only simple ISR/main communication.
- Keep `volatile` limited to MMIO and ISR-observed flags; never treat it as atomicity.
- Require DMA buffers with stable lifetime, explicit ownership, and documented alignment; reject active DMA into stack storage.
- Require ISRs to acknowledge hardware, publish compact events, and defer heavy work to foreground code.
- Flag blocking, heap allocation, float math, complex formatting, virtual dispatch, unbounded parsing, and hidden flash bloat in ISR/driver paths.
- Treat Cortex-M0 floating point as software-emulated: allow only outside hot/ISR paths when timing and map deltas are acceptable; prefer fixed-point and forbid `%f` in release logging.
- Flag implicit narrowing, signed/unsigned mixing, overflow-prone counters, and shifts/masks that rely on undefined or implementation-defined behavior.
- Flag dynamic initialization of globals or local statics in startup/driver paths unless it is intentional, bounded, and measured.
- If code says EEPROM, check whether it is actually flash emulation and review page erase, wear, power-loss safety, and option-byte separation.

## Size Checks

When build files are in scope, check for:

```text
-Os
-ffunction-sections
-fdata-sections
-Wl,--gc-sections
```

Recommend `-fno-exceptions`, `-fno-rtti`, and measured `-flto` only when compatible. For size-sensitive changes, inspect:

```sh
arm-none-eabi-size firmware.elf
arm-none-eabi-nm --size-sort -S firmware.elf
arm-none-eabi-objdump -h firmware.elf
```

Use linker map deltas first; add `--print-gc-sections` or `--print-memory-usage` when diagnosing retained code. Do not block on missing size reports unless the change plausibly affects flash/RAM.

Use code-review format:

```markdown
Findings
- [P1] Short title - path/to/file.cpp:123
  Explain the concrete trigger, impact, and minimal fix.

Open Questions
- Only include questions that affect correctness or acceptance.

Summary
- Briefly state overall risk and what was reviewed.
```

If there are no findings, say so and mention remaining test or measurement gaps, such as missing map delta, ISR latency measurement, or DMA error-path coverage.
