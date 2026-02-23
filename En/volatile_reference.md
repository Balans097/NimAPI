# volatile — Module Reference

## Overview

The `volatile` module provides two generic procedures — `volatileLoad` and `volatileStore` — for performing **volatile memory access** in Nim. These procedures are primarily designed for **embedded and systems programming**, where the compiler must not optimise away, reorder, or cache memory reads and writes.

### What does "volatile" mean?

In C and C++, the `volatile` keyword on a pointer or variable tells the compiler: *"do not assume anything about this memory location between accesses."* Every read must actually go to the hardware address; every write must actually reach it. Without `volatile`, an optimising compiler may:

- Cache a value in a register and never re-read it from memory.
- Eliminate what looks like a redundant write.
- Reorder reads and writes for performance.

This is disastrous when the memory location is a **hardware register**, a **memory-mapped I/O address**, or shared memory written by an interrupt service routine (ISR) or another execution context that the compiler cannot see.

### When is this module relevant?

- Reading the current value of a **hardware peripheral register** (e.g., a UART status flag, a timer counter).
- Writing a command or value to a **memory-mapped peripheral**.
- Accessing variables shared between **main code and interrupt handlers** on bare-metal systems.
- Any situation where the compiler's view of "who changes memory" is incomplete.

### Backend behaviour

| Backend | Effect |
|---|---|
| C / C++ | Emits a true `volatile` pointer cast; the compiler is forbidden from caching or reordering the access. |
| Nim VM (`nimvm`) | Falls back to a plain `src[]` / `dest[] = val` dereference — volatile semantics are not meaningful at compile time. |
| JavaScript | Falls back to a plain dereference — the JS engine has no notion of volatile memory. |

---

## Exported Procedures

---

### `volatileLoad`

```nim
proc volatileLoad*[T](src: ptr T): T {.inline, noinit.}
```

Performs a **volatile read** from the memory address pointed to by `src` and returns the value stored there as type `T`.

The procedure is generic: `T` can be any type — `uint8`, `uint32`, a `distinct` register type, a `tuple`, etc. The compiler generates a cast to a `volatile` pointer before the dereference, which guarantees that the actual hardware address is consulted on every call rather than a cached copy.

The `noinit` pragma tells Nim not to zero-initialise the `result` variable before the volatile read fills it — this avoids a spurious extra write.

**On the C backend, the generated code looks like:**

```c
result = (*( uint32 volatile* ) src);
```

**Example — reading a hardware status register:**

```nim
import volatile

# Suppose 0x40013800 is the base address of a UART peripheral.
# Bit 5 of its status register signals "transmit buffer empty".
const UART_SR = cast[ptr uint32](0x40013800'u32)

proc uartReady(): bool =
  let status = volatileLoad(UART_SR)
  return (status and (1'u32 shl 5)) != 0
```

Each call to `uartReady` will physically re-read the register rather than using a value cached by the compiler from a previous read.

**Example — reading a byte from shared memory updated by an ISR:**

```nim
import volatile

var flag {.global.}: uint8 = 0

# Interrupt service routine sets flag = 1 asynchronously.
# In main loop, we must not let the compiler cache the read:
proc pollFlag(): uint8 =
  volatileLoad(addr flag)
```

---

### `volatileStore`

```nim
proc volatileStore*[T](dest: ptr T, val: T) {.inline.}
```

Performs a **volatile write** of the value `val` into the memory address pointed to by `dest`.

Like `volatileLoad`, this is generic over `T` and works for any type. The compiler generates a cast to a `volatile` pointer before the assignment, ensuring the write actually reaches the target address and is not eliminated as "redundant" by the optimiser.

**On the C backend, the generated code looks like:**

```c
*((uint32 volatile*)(dest)) = val;
```

**Example — writing a command to a hardware control register:**

```nim
import volatile

const TIMER_CR1 = cast[ptr uint32](0x40000000'u32)

proc startTimer() =
  # Set bit 0 (CEN — counter enable) in the timer control register.
  volatileStore(TIMER_CR1, 1'u32)
```

Without `volatile`, the compiler might see that `TIMER_CR1` is "never read afterwards" and silently drop the write. With `volatileStore`, the write is guaranteed to happen.

**Example — signalling an interrupt handler via shared memory:**

```nim
import volatile

var ready {.global.}: uint8 = 0

proc signalIsr() =
  # The ISR reads `ready`; we must make sure this write is visible.
  volatileStore(addr ready, 1'u8)
```

---

## Complete Usage Example

The following example models a typical bare-metal pattern: polling a peripheral status register and writing to a data register once it is ready.

```nim
import volatile

# Memory-mapped register addresses (example values)
const
  SPI_SR   = cast[ptr uint32](0x4001300C'u32)  # Status register
  SPI_DR   = cast[ptr uint32](0x4001300C'u32)  # Data register
  TXE_BIT  = 1'u32 shl 1                        # "Tx buffer empty" flag

proc spiSendByte(data: uint8) =
  # Wait until the transmit buffer is empty — must re-read each iteration.
  while (volatileLoad(SPI_SR) and TXE_BIT) == 0:
    discard

  # Write the byte — must not be optimised away.
  volatileStore(SPI_DR, uint32(data))
```

Without volatile accesses, an optimising compiler could hoist the `SPI_SR` read out of the loop (since it sees no ordinary Nim code modifying it), resulting in an infinite loop or incorrect behaviour.

---

## Key Points to Remember

- **Use `volatileLoad` / `volatileStore` instead of plain `src[]` / `dest[] = val`** whenever the memory location can change or must change outside the compiler's knowledge: hardware registers, ISR-shared variables, spinlocks, etc.
- **Volatile is not a synchronisation primitive.** On multi-core systems it does not provide memory ordering guarantees or atomicity. For those, use atomic operations or proper fences.
- **Only affects C-like backends.** On the Nim VM and JS targets these procedures degrade to ordinary pointer dereferences — volatile semantics simply do not exist in those environments.
- **The procedures are `inline`**, so there is zero call overhead. They expand directly at the call site.
