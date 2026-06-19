# STM32F042 C++ IRQ/DMA Review Guide

Target: STM32F042x4/x6, Cortex-M0, up to 48 MHz, 16/32 KB flash, 6 KB SRAM with parity, 5 DMA channels, CRC unit, no D-cache, four NVIC priority levels. Treat "EEPROM" as flash emulation unless the board has external EEPROM.

## Review Invariants

- Bare metal by default: no RTOS, no BASEPRI, no D-cache maintenance, no lock-free `LDREX`/`STREX` designs.
- Use C++ as a small profile: `constexpr`, `static_assert`, fixed-width types, `enum class`, plain structs/classes, tiny RAII guards, fixed storage, thin templates.
- Avoid exceptions, RTTI, heap use, iostreams, recursion, hidden global constructors, virtual dispatch, float math in hot paths, float formatting, and large header-only libraries in firmware paths.
- Prefer STM32CubeF0 LL or direct registers for hot/size-critical code; HAL is acceptable only when it pays for itself in clarity and measured size/latency.
- Preserve safety mechanisms: bounds checks, CRCs, DMA ownership, interrupt acknowledgement, flash-write power-loss handling.

## Size and Build

Check release builds for:

```text
-Os -ffunction-sections -fdata-sections -Wl,--gc-sections
```

Use only after compatibility/size checks:

```text
-flto -fno-exceptions -fno-rtti -fno-threadsafe-statics
```

Measure:

```sh
arm-none-eabi-size firmware.elf
arm-none-eabi-nm --size-sort -S firmware.elf
arm-none-eabi-objdump -h firmware.elf
```

Inspect `.text`, `.rodata`, `.data`, `.bss`, stack usage, retained HAL/middleware, USB/CAN stacks, printf/float support, startup/static init, vector table, and linker map deltas. Use `-Wl,--print-gc-sections` and `-Wl,--print-memory-usage` when explaining retained code.

## Floating Point

- STM32F042/Cortex-M0 has no FPU; `float`/`double` arithmetic and `%f` formatting pull software helper/runtime code.
- Flag float in ISRs, DMA callbacks, drivers, control loops, and logging paths unless timing and map deltas prove it acceptable.
- Prefer fixed-point units such as mV, uA, centi-degrees, ppm, or Q-format values; convert to text with integer formatting.
- Treat `double` as prohibited by default.

## IRQ Rules

- ISR = acknowledge source, capture minimal state, publish event, exit.
- Required comment: trigger, max work, shared state, DMA interaction.
- Keep work bounded; no blocking, allocation, float math, complex logging, virtual dispatch, protocol parsing, or flash writes.
- Centralize NVIC setup; document numeric priority and urgency. F0 has only four priority levels, so priority policy must be simple.
- Handler mode uses the main stack; review ISR stack growth and nested interrupt assumptions.

## DMA Rules

- DMA buffers must be static or owned by long-lived objects; never stack.
- State ownership explicitly: `CpuCanWrite`, `DmaTxActive`, `DmaRxActive`, `CpuCanRead`, `Error`.
- Do not read RX or mutate TX buffers while DMA owns them.
- Keep channel ownership auditable across the 5 channels.
- Completion ISR publishes a compact event; foreground handles parsing/retry/error policy.
- Align buffers/descriptors as required by the reference manual and peripheral width.

## Shared State

- `volatile` only for MMIO and ISR-observed flags; it is not atomicity.
- Simple shared operations: naturally aligned byte/halfword/word single load/store.
- Compound updates, counters, ring push/pop pairs, and multi-field publishes require single-writer ownership or a tiny `PRIMASK` critical section.
- Critical sections should only snapshot or publish shared state; move parsing, copying large buffers, HAL/LL calls, flash writes, waits, and loops outside the masked region.
- One writer per ring index; define overflow policy.
- Do not publish stack pointers or reuse DMA buffers before ownership returns.

## MMIO, Arithmetic, Init

- Use CMSIS/ST headers; isolate raw register casts in drivers.
- Preserve reserved bits and follow rc_w1/rc_w0/toggle semantics from RM0091.
- Use `__DMB`, `__DSB`, `__ISB` only with a hardware-ordering comment.
- Review narrowing, signed/unsigned mixing, shifts, masks, and wraparound counters.
- Use fixed underlying types for register-facing or persisted enums.
- Prefer explicit startup initialization over hardware-touching globals or nontrivial local statics.

## Flash-Emulated Settings

- STM32F042 has flash and option bytes, not true internal EEPROM.
- Review flash page erase/write granularity, wear, power-loss recovery, CRC/version/sequence, and separation from option bytes.
- Never write flash from an ISR. Publish an event and commit from foreground when timing and voltage are safe.

## Checklist

- [ ] Size/map delta reviewed for code, rodata, RAM, stack.
- [ ] No unexpected HAL/middleware/std-library/float helper or `%f` formatting symbols.
- [ ] ISR source acknowledged correctly and work is bounded.
- [ ] Shared ISR/main state uses single-writer or minimal snapshot/publish critical sections.
- [ ] DMA buffer lifetime, alignment, channel, and ownership are explicit.
- [ ] No active DMA uses stack or ambiguous ownership.
- [ ] Register writes preserve reserved bits and clear flags correctly.
- [ ] Arithmetic/init/template bloat risks reviewed.
- [ ] Flash-emulated settings are power-loss and wear safe.

## References

- STM32F042x4/x6 datasheet: https://www.st.com/resource/en/datasheet/stm32f042c4.pdf
- STM32F0x1/F0x2/F0x8 reference manual RM0091: https://www.st.com/resource/en/reference_manual/rm0091-stm32f0x1stm32f0x2stm32f0x8-advanced-armbased-32bit-mcus-stmicroelectronics.pdf
- STM32F0 Cortex-M0 programming manual PM0215: https://www.st.com/resource/en/programming_manual/pm0215-stm32f0-series-cortexm0-programming-manual-stmicroelectronics.pdf
- STM32CubeF0 HAL/LL package: https://www.st.com/en/embedded-software/stm32cubef0.html
- CMSIS intrinsics: https://arm-software.github.io/CMSIS_6/main/Core/group__intrinsic__CPU__gr.html
- GCC/ld options: https://gcc.gnu.org/onlinedocs/gcc/ and https://sourceware.org/binutils/docs/ld/Options.html
