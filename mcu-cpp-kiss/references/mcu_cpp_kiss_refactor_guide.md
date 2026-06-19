# STM32F042 C++ KISS Refactor Guide

Target: STM32F042x4/x6, Cortex-M0, 48 MHz max, 16/32 KB flash, 6 KB SRAM, 5 DMA channels, CRC unit, no D-cache. KISS means fewer retained bytes, clearer ownership, bounded IRQ/DMA behavior, and no decorative abstraction.

## Measure First

```sh
arm-none-eabi-size firmware.elf
arm-none-eabi-nm --size-sort -S firmware.elf
arm-none-eabi-objdump -h firmware.elf
```

Inspect map deltas for `.text`, `.rodata`, `.data`, `.bss`, stack, startup/static init, HAL/LL objects, USB/CAN/touch middleware, printf/float support, and unused peripheral drivers. Use `--print-gc-sections` and `--print-memory-usage` to explain retained code.

## Highest-Value Cuts

- Avoid new HAL in low-level paths by default. Prefer LL or raw registers for simple peripheral setup, IRQ/DMA control, and hot paths.
- Replace existing HAL where map/timing/control-flow evidence shows bloat or hidden work; keep HAL only when it clearly reduces risk and its cost is acceptable.
- Remove unused STM32CubeF0 modules, middleware, IRQ handlers, weak callbacks, and peripheral init from the build.
- Compile out release strings; use event IDs/counters instead of formatting.
- Remove float math from hot/ISR paths; fixed-point-convert values before logging and never keep `%f` in release logs.
- Replace heap/STL containers with caller-owned buffers or static pools.
- Move heavy template bodies into one non-template implementation.
- Replace hidden globals/local-static init with explicit startup calls.
- Consolidate flash-emulated settings behind one store with CRC/version/sequence/wear policy.

## Keep Code Cohesive

Do not split straight-line hardware sequences into one-use helpers. Keep a function whole when it is short, linear, ordering-sensitive, or one hardware transaction. Extract only to remove real duplication, name a non-obvious invariant, isolate hardware-independent logic, or make a meaningful test seam.

## Safe Patterns

Thin template:

```cpp
void write_bytes(const void* p, uint16_t n);
template <class T>
inline void write_object(const T& v) {
    static_assert(std::is_trivially_copyable<T>::value, "plain object required");
    write_bytes(&v, sizeof(v));
}
```

Explicit DMA state:

```cpp
enum class DmaOwner : uint8_t { CpuCanWrite, DmaTxActive, DmaRxActive, CpuCanRead, Error };
```

Explicit startup:

```cpp
void system_init() {
    clock_init();
    gpio_init();
    dma_init();
    uart_init();
}
```

## Do Not Cut

- Bounds checks, CRCs, flash power-loss protection, DMA ownership, interrupt acknowledgement, voltage/timing checks around flash writes.
- `volatile` on MMIO/ISR flags, but do not spread it to whole objects.
- Critical sections for compound ISR/main shared updates.
- Error paths needed to return DMA ownership deterministically.

## F042 Review Heuristics

- 6 KB SRAM means stack, DMA buffers, ring buffers, and config copies must be budgeted together.
- 16 KB flash parts need a stricter feature gate than 32 KB parts; USB/CAN/printf can dominate.
- Cortex-M0 has no FPU; `float`, `double`, division, and `%f` formatting must justify their timing and flash cost.
- No internal EEPROM: persistent parameters are flash-emulated unless external EEPROM exists.
- No D-cache: cache-maintenance code is wrong/noise for this target.
- Four NVIC priority levels: do not design elaborate priority schemes.
- CRC unit exists; prefer it for stored config/image integrity when it reduces code.

## Checklist

- [ ] Target is flash/RAM/stack/latency/readability, not style-only cleanup.
- [ ] Map/size evidence or obvious retained-code reason identified.
- [ ] HAL/middleware and weak-callback retention reviewed.
- [ ] Templates, STL, exceptions, RTTI, heap, float formatting, static init reviewed.
- [ ] IRQ/DMA ownership and timing remained explicit.
- [ ] No one-use helper fragmentation added.
- [ ] Behavior preserved; safety removals explicitly accepted.
- [ ] Rebuilt, compared size, and reported deltas/risks.

## References

- STM32F042 datasheet: https://www.st.com/resource/en/datasheet/stm32f042c4.pdf
- RM0091 reference manual: https://www.st.com/resource/en/reference_manual/rm0091-stm32f0x1stm32f0x2stm32f0x8-advanced-armbased-32bit-mcus-stmicroelectronics.pdf
- PM0215 programming manual: https://www.st.com/resource/en/programming_manual/pm0215-stm32f0-series-cortexm0-programming-manual-stmicroelectronics.pdf
- STM32CubeF0 HAL/LL: https://www.st.com/en/embedded-software/stm32cubef0.html
