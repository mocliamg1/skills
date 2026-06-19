# Cortex-M0 C++ IRQ/DMA Review Guide

Use this as the expanded checklist for bare-metal Cortex-M0 C++ reviews. It is a practical local standard, not MISRA/AUTOSAR.

## Core Rules

- Target bare-metal Cortex-M0 unless code proves otherwise: no RTOS assumptions, BASEPRI masking, D-cache maintenance, or M7-style cache policy.
- Optimize for deterministic execution, bounded latency, small binaries, explicit ownership, and reviewable concurrency.
- Use C++ only where it adds type safety or compile-time checking without hidden runtime cost.
- Prefer `constexpr`, `static_assert`, `enum class`, plain structs/classes, small RAII guards, fixed-size storage, and thin templates.
- Avoid exceptions, RTTI, heap allocation, hosted-library features, recursion, hidden hardware-touching global constructors, and virtual dispatch in ISR/DMA/driver paths.
- Treat `LDREX`/`STREX` lock-free designs as unavailable on Cortex-M0 unless the exact target/toolchain proves support.
- Treat this as a focused C++ subset/profile: adopt rules that buy safety, size, and determinism; do not modernize code with features that add runtime or review cost.

## Build and Size

Check release builds for:

```text
-Os
-ffunction-sections
-fdata-sections
-fno-exceptions
-fno-rtti
-Wl,--gc-sections
```

Measure before/after nontrivial changes:

```sh
arm-none-eabi-size firmware.elf
arm-none-eabi-nm --size-sort -S firmware.elf
arm-none-eabi-objdump -h firmware.elf
```

Review `.text`, `.rodata`, `.data`, `.bss`, stack usage, vector/startup code, library symbols, large strings/tables, float formatting, generic formatting, and linked HAL/vendor modules. Keep `-flto` only if measured compatible.

Useful diagnostics when investigating retained code:

```text
-Wl,--print-gc-sections
-Wl,--print-memory-usage
```

## IRQs

ISRs acknowledge hardware, capture minimal state, and publish compact foreground events.

ISR contract comment:

```cpp
// IRQ: UART0 receive
// Trigger: RX-not-empty or error flag from UART0
// Max work: read status, drain up to one byte, clear IRQ source, publish event
// Shared data: uart0_rx_head, uart0_events
// DMA: does not start or stop DMA
extern "C" void UART0_IRQHandler();
```

- Keep ISRs short and bounded.
- Clear or acknowledge the hardware interrupt source deterministically.
- Read hardware status once when practical, then act on the snapshot.
- Do not block, sleep, poll for long completion, or wait for another interrupt inside an ISR.
- Do not allocate, use virtual dispatch, do complex formatting/logging, or call unbounded code.
- Do not perform protocol parsing in an ISR unless it is tiny, bounded, and measured.
- Publish events, counters, or buffer indices for the main loop to process.
- Centralize NVIC priority setup; comment numeric priority and urgency.

```cpp
// NVIC priority 0: highest urgency.
// NVIC priority 3: lower urgency than 0.
```

## DMA Guidelines

DMA is another bus master. Treat every DMA transaction as an ownership transfer between CPU code, the DMA controller, and the peripheral.

- DMA buffers must have stable lifetime for the entire transaction.
- Never start DMA using a stack buffer.
- Separate descriptors/control blocks from data buffers when hardware supports it.
- Represent buffer ownership and alignment explicitly.
- Do not mutate a buffer while DMA owns it.
- Do not read an RX buffer until completion returns ownership to CPU code.
- Completion interrupts must publish compact events; foreground code performs expensive processing.
- Keep DMA channel configuration centralized enough that channel ownership is auditable.

Ownership sketch:

```cpp
enum class DmaOwner : uint8_t {
    CpuCanWrite,
    DmaTxActive,
    DmaRxActive,
    CpuCanRead,
    Error,
};

struct DmaBuffer {
    alignas(4) uint8_t bytes[128];
    volatile DmaOwner owner;
    volatile uint16_t length;
};

static DmaBuffer uart0_dma_rx;
```

Cortex-M0 parts normally do not have data cache. Do not add cached Cortex-M clean/invalidate calls unless the target changed and has a cache policy.

## Shared Data Between ISR, DMA, and Main

Prefer single-writer ownership and natural-width flags.

Allowed patterns:

- ISR writes a `volatile` flag; main loop reads and clears it inside a critical section.
- ISR advances a ring-buffer head; main loop advances the tail.
- DMA ISR sets a completion event; main loop consumes the completed buffer.
- Main loop prepares a DMA buffer, then transfers ownership before enabling DMA.

Avoid:

- Shared read-modify-write without interrupt protection.
- Multiple writers to the same variable.
- Multi-byte shared state updated without a critical section.
- Publishing pointers to stack data.
- Using `volatile` as a substitute for atomicity.

- Single naturally aligned byte, halfword, and word loads/stores are the only shared operations that should be treated as simple.
- Any compound operation, multi-field update, counter increment, queue push/pop pair, or read-modify-write must use a critical section or a single-writer design.

Critical-section helper:

```cpp
class IrqGuard {
public:
    IrqGuard() : primask_(__get_PRIMASK()) {
        __disable_irq();
    }

    ~IrqGuard() {
        if ((primask_ & 1u) == 0u) {
            __enable_irq();
        }
    }

    IrqGuard(const IrqGuard&) = delete;
    IrqGuard& operator=(const IrqGuard&) = delete;

private:
    uint32_t primask_;
};
```

IRQ-to-main event example:

```cpp
enum : uint32_t {
    EventUartRx = 1u << 0,
    EventDmaDone = 1u << 1,
};

static volatile uint32_t g_events;

extern "C" void UART0_IRQHandler() {
    const uint32_t status = UART0->STATUS;
    UART0->STATUS = status; // Clear handled flags according to the device manual.

    if ((status & UART_RX_READY) != 0u) {
        g_events |= EventUartRx;
    }
}

uint32_t take_events() {
    IrqGuard lock;
    const uint32_t events = g_events;
    g_events = 0;
    return events;
}

void main_loop_iteration() {
    const uint32_t events = take_events();

    if ((events & EventUartRx) != 0u) {
        service_uart_rx();
    }

    if ((events & EventDmaDone) != 0u) {
        service_completed_dma();
    }
}
```

## Register and Hardware Access

- Keep `volatile` at the hardware boundary: MMIO registers and ISR-observed flags.
- Do not make whole driver objects `volatile`.
- Avoid casting arbitrary addresses throughout application code; isolate MMIO access in drivers.
- Use named masks and typed enums for register fields.
- Follow the vendor reference manual for write-one-to-clear, read-to-clear, and reserved-bit behavior.
- Use CMSIS NVIC functions for interrupt enable, disable, pending state, and priority setup.
- Use CMSIS instruction intrinsics such as `__DMB`, `__DSB`, and `__ISB` only when hardware ordering requires them.
- Every barrier must have a comment explaining the hardware ordering requirement.

Example:

```cpp
// Ensure descriptor writes reach memory before the DMA channel is enabled.
__DMB();
DMA0->CTRL = DMA_CTRL_ENABLE;
```

## Arithmetic and Initialization

- Flag implicit narrowing, signed/unsigned mixing, unchecked counter wrap, and shifts wider than the underlying type.
- Prefer fixed underlying types for persisted or register-facing enums.
- Use `static_assert` for buffer sizes, register field widths, DMA alignment, and persistent-layout sizes.
- Prefer compile-time initialization. Flag hardware-touching global constructors and nontrivial local statics in startup/driver paths.
- If the project uses `-fno-threadsafe-statics`, verify local static initialization cannot race with interrupts or re-entry.

## Architecture and Abstractions

- Use ISR top halves plus foreground bottom halves. Foreground code owns parsing, retries, state transitions, and error policy.
- Keep DMA transaction state explicit: `Idle`, `Prepared`, `Active`, `Complete`, `Error`.
- Use one writer per ring-buffer index; define overflow policy.
- Prefer explicit polling state machines for peripheral transactions.
- Keep templates thin. Move parsing, formatting, retries, logging, and state machines into non-template `.cpp` functions.

```cpp
void write_bytes(const void* data, uint16_t length);

template <typename T>
inline void write_object(const T& value) {
    static_assert(std::is_trivially_copyable<T>::value, "DMA/register payload must be plain data");
    write_bytes(&value, sizeof(value));
}
```

## Review Checklist

### Size/Memory

- [ ] Release build size report was reviewed.
- [ ] Linker map delta was reviewed for `.text`, `.rodata`, `.data`, and `.bss`.
- [ ] No unexpected standard-library symbols were introduced.
- [ ] No float formatting was introduced.
- [ ] No heap allocation was introduced in normal operation.
- [ ] Stack growth is bounded and acceptable.
- [ ] Static initialization did not add hidden runtime or ordering risk.

### IRQ

- [ ] Every ISR has the required header comment.
- [ ] ISR work is short and bounded.
- [ ] Hardware source is cleared or acknowledged correctly.
- [ ] ISR does not block, allocate, or call complex formatting/logging.
- [ ] Shared ISR/main data is natural-width single load/store or protected by a critical section.
- [ ] NVIC priority comments use numeric priority and urgency wording.

### DMA

- [ ] DMA buffers have stable lifetime and documented alignment.
- [ ] No active DMA uses stack storage.
- [ ] Buffer ownership state is explicit.
- [ ] CPU does not read/write buffers while DMA owns them.
- [ ] DMA completion path publishes a compact event.
- [ ] Error and cancellation paths return ownership deterministically.

### Hardware

- [ ] MMIO access is isolated in driver code.
- [ ] `volatile` is limited to registers and ISR-observed flags.
- [ ] Barriers, if used, have a hardware-ordering comment.
- [ ] Reserved register bits are preserved as required by the vendor manual.
- [ ] Narrowing, overflow, shift, and enum-underlying-type risks were reviewed.

## References

- [CMSIS-Core NVIC interrupt and exception APIs](https://arm-software.github.io/CMSIS_5/Core/html/group__NVIC__gr.html)
- [CMSIS-Core intrinsic CPU instructions](https://arm-software.github.io/CMSIS_6/main/Core/group__intrinsic__CPU__gr.html)
- [GCC optimization options](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)
- [GCC C++ dialect options](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Dialect-Options.html)
- [GNU linker options, including `--gc-sections` and map files](https://sourceware.org/binutils/docs/ld/Options.html)
- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
- [AUTOSAR C++14 Guidelines](https://www.autosar.org/fileadmin/standards/R17-10_R1.2.0/AP/AUTOSAR_RS_CPP14Guidelines.pdf)
