# MCU C++ KISS Refactor Guide

Use this expanded checklist when simplifying and debloating C++ firmware for small MCUs.

## Goal

Target firmware that is easy to audit, hard to misuse, small enough for flash/RAM, and deterministic under IRQ/DMA load. Good simplification removes:

- unused code
- duplicated logic
- hidden runtime behavior
- unnecessary abstraction layers
- excessive helper-function fragmentation
- template/codegen bloat
- dynamic allocation
- string/formatting bloat
- unclear ownership

Do not remove without explicit user acceptance:

- validation
- bounds checks
- CRCs
- power-loss protection
- DMA ownership state
- interrupt acknowledgement
- hardware error handling

## Measurement

Useful commands:

```sh
arm-none-eabi-size firmware.elf
arm-none-eabi-nm --size-sort -S firmware.elf
arm-none-eabi-objdump -h firmware.elf
```

Build/link outputs to inspect:

- linker map file
- `.text` for executable code
- `.rodata` for strings/tables/constants
- `.data` for initialized RAM
- `.bss` for zero-initialized RAM
- stack usage reports
- list of linked object files and libraries

Size-oriented flags:

```text
-Os
-ffunction-sections
-fdata-sections
-Wl,--gc-sections
```

Consider `-flto`, `-fno-exceptions`, and `-fno-rtti` only after compatibility checks.

## Common Bloat Sources

- `printf` with floats or wide formatting support
- `iostream`, locale, regex, filesystem, thread/future libraries
- exceptions and RTTI
- virtual hierarchies in low-level code
- header-only/template-heavy libraries
- multiple template instantiations for many types or buffer sizes
- duplicated driver instances compiled for unused peripherals
- HAL modules linked even when not used
- large debug strings and command shells
- lookup tables, fonts, protocol names, and diagnostic text
- heap allocators and generic containers

## Safe Refactor Patterns

### Thin Templates

```cpp
void write_bytes(const void* data, uint16_t length);

template <typename T>
inline void write_object(const T& value) {
    static_assert(std::is_trivially_copyable<T>::value, "plain object required");
    write_bytes(&value, sizeof(value));
}
```

Keep formatting, retries, state machines, logging, and parsing out of templates. Use explicit instantiation for a small fixed type set.

### Explicit State Machine

Prefer small enums and one transition function over nested callback chains when IRQ/DMA behavior must be audited.

```cpp
enum class TxState : uint8_t {
    Idle,
    PreparingDma,
    DmaActive,
    Complete,
    Error,
};
```

### Cohesive Functions

Keep code together when:

- the operation is short and linear
- the helper would have only one caller
- the helper name would merely restate the code
- the code touches one hardware transaction or IRQ/DMA handoff
- splitting would hide ordering requirements

Extract a helper when:

- it removes meaningful duplication
- it isolates a hardware-independent calculation
- it names a non-obvious invariant
- it creates a small unit that can be tested without hardware
- it keeps an ISR or DMA callback bounded and clearer

Avoid splitting short register sequences into one-use wrappers unless the wrappers are reused or encode real hardware constraints.

```cpp
void start_uart_dma_tx(const uint8_t* data, uint16_t length) {
    uart_tx.owner = DmaOwner::DmaTxActive;
    uart_tx.length = length;

    DMA0->SRC = reinterpret_cast<uint32_t>(data);
    DMA0->DST = reinterpret_cast<uint32_t>(&UART0->TXDATA);
    DMA0->COUNT = length;

    // Descriptor writes must be visible before enabling the DMA channel.
    __DMB();
    DMA0->CTRL = DMA_CTRL_ENABLE;
}
```

### Caller-Owned Buffers

```cpp
bool encode_packet(uint8_t* out, uint16_t capacity, uint16_t* written);
```

Avoid heap-backed return containers in normal firmware paths.

### Compile-Time Gates for Logs

```cpp
#if ENABLE_TRACE
void trace_event(uint16_t id, uint16_t value);
#else
inline void trace_event(uint16_t, uint16_t) {}
#endif
```

Prefer release event IDs/counters over retained diagnostic strings.

### Central Persistent Settings

Use one parameter store for EEPROM layout, CRC, versioning, and commit policy. Modules own setting meaning and defaults, not EEPROM addresses.

## IRQ and DMA Constraints

Do not simplify by making concurrency less explicit.

Keep:

- short ISR top halves
- deterministic interrupt acknowledgement
- compact event publication
- explicit DMA buffer ownership
- stable DMA buffer lifetime
- natural-width shared flags or critical sections
- foreground handling for heavy processing

Flag these as unsafe:

- heap allocation in ISRs
- blocking waits in ISRs
- complex formatting/logging in ISRs
- active DMA into stack buffers
- CPU mutation of a buffer while DMA owns it
- `volatile` used as a substitute for atomicity

## Decisions

- Prioritize large avoidable symbols proven by the map file.
- Simplify abstractions that hide ownership or timing.
- Leave tiny duplicated blocks alone when they are clearer than generic code.
- Use generic code only when it reduces size and keeps behavior obvious.
- Keep three obvious lines local unless extraction names a real invariant or removes meaningful duplication.
- Keep ordering-sensitive register writes visible or document ordering at the call site.
- Require explicit acceptance for byte savings that weaken safety.
- Keep changes smaller when tests or hardware validation are weak.

## Checklist

- [ ] Identify the target: flash, RAM, stack, latency, or readability.
- [ ] Inspect map/size reports if available.
- [ ] Find IRQ, DMA, EEPROM, startup, and linker/build interactions.
- [ ] Identify behavior that must not change.
- [ ] Remove unused code from the build, not only runtime paths.
- [ ] Keep public APIs smaller and more explicit.
- [ ] Avoid over-splitting code into one-use tiny helpers.
- [ ] Keep templates thin or explicitly instantiated.
- [ ] Avoid new heap use, exceptions, RTTI, or heavy standard library dependencies.
- [ ] Keep DMA ownership and ISR handoff explicit.
- [ ] Keep validation and error handling unless intentionally traded off.
- [ ] Rebuild.
- [ ] Compare size numbers.
- [ ] Run available tests.
- [ ] Check IRQ/DMA paths for races and lifetime bugs.
- [ ] Report size deltas and any residual risk.

## Review Output Template

```markdown
Simplifications
- Removed/changed X because Y; behavior preserved by Z.

Size
- Before: ...
- After: ...
- Delta: ...

Validation
- Ran: ...
- Not run: ...

Risks
- ...
```
