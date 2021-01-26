# Fuel VM Specification

- [Introduction](#introduction)
- [Parameters](#parameters)
- [Semantics](#semantics)
- [Opcodes](#opcodes)
- [Transaction Execution](#transaction-execution)
- [Call Frames](#call-frames)
- [Ownership](#ownership)

## Introduction

This document provides the specification for the Fuel Virtual Machine (FuelVM). The specification covers the types, opcodes, and execution semantics.

## Parameters

| name         | type     | value   | note   |
| ------------ | -------- | ------- | ------ |
| `VM_MAX_RAM` | `uint64` | `2**20` | 1 MiB. |

## Semantics

FuelVM instructions are exactly 32 bits (4 bytes) wide and comprise of a combination of:
* Opcode: 8 bits
* Register/special register (see below) identifier: 6 bits
* Immediate value: 12, 18, or 24 bits, depending on operation

Of the 64 registers (6-bit register address space), the first `16` are reserved:
| value  | register | name            | description                                                                                                            |
| ------ | -------- | --------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `0x00` | `$z`     | zero            | Zero-containing register (convenience register frequently found in register machines).                                 |
| `0x01` | `$of`    | overflow        | Register containing high bits of multiplication, remainder in division, or overflow of signed addition or subtraction. |
| `0x02` | `$pc`    | program counter | The program counter. Memory address of the current instruction.                                                        |
| `0x03` | `$fp`    | frame pointer   | Memory address on top of current call frame (points to fee memory).                                                    |
| `0x04` | `$fpp`   | prev frame ptr  | Previous call frame's frame pointer.                                                                                   |
| `0x05` | `$hp`    | heap pointer    | Memory address of the current bottom of the heap (points to free memory).                                              |
| `0x06` | `$hpp`   | prev heap ptr   | Previous call frame's heap pointer.                                                                                    |
| `0x07` | `$err`   | error           | Error codes for particular operations.                                                                                 |
| `0x08` | `$gas`   | gas             | Remaining gas.                                                                                                         |
| `0x09` |          |                 |                                                                                                                        |
| `0x0A` |          |                 |                                                                                                                        |
| `0x0B` |          |                 |                                                                                                                        |
| `0x0C` |          |                 |                                                                                                                        |
| `0x0D` |          |                 |                                                                                                                        |
| `0x0E` |          |                 |                                                                                                                        |
| `0x0F` |          |                 |                                                                                                                        |

Integers are represented in [big-endian](https://en.wikipedia.org/wiki/Endianness) format, and all operations are unsigned. Boolean `false` is `0` and Boolean `true` is `1`.

Registers are 64 bits (8 bytes) wide. Words are the same width as registers.

Persistent state (i.e. storage) is a key-value store with 32-byte keys and 32-byte values. Each contract has its own persistent state that is independent of other contracts. This is committed to in a Sparse Binary Merkle Tree.

## Opcodes

A complete list of opcodes in the Fuel VM is documented [here](./opcodes.md).

## Transaction Execution

If script bytecode is present, transaction validation requires execution.

A single monolithic memory of size `VM_MAX_RAM` bytes is allocated, indexed by individual byte. A stack and heap memory model is used, allowing for dynamic memory allocation. The stack begins at `0` and grows upward. The heap begins at `VM_MAX_RAM-1` and grows downward.

Before execution begins, the following is pushed on the stack sequentially:
1. Fuel block height (`uint64`, word-aligned).
1. Block producer address (`byte[32]`, word-aligned).
1. Transaction gas limit (`uint64`, word-aligned).
1. Transaction hash (`byte[32]`, word-aligned).
1. Block hash for the previous 256 blocks, starting from the previous block (`byte[32][256]`, word-aligned). Block hash is zero (`0x00**32`) for negative block heights.
1. The [transaction, serialized](./tx_format.md).

`$pc` is initialized to the start of the transaction's script bytecode and execution begins.

## Call Frames

Cross-contract calls push a _call frame_ onto the stack, similar to a stack frame used in regular languages for function calls (which may be used by a high-level language that target the FuelVM). The distinction is as follows:
1. Stack frames: store metadata across trusted internal (i.e. intra-contract) function calls. Not supported natively by the FuelVM, but may be used as an abstraction at a higher layer.
1. Call frames: store metadata across untrusted external (i.e. inter-contract) calls. Supported natively by the FuelVM.

Call frames are needed to ensure that the called contract cannot mutate the running state of the current executing contract. They segment access rights for memory: the currently-executing contracts may only write to their own call frame, their own heap, and return value ranges assigned to them.

A call frame consists of the following, word-aligned:

| bytes | type                 | value             | description                                                                     |
| ----- | -------------------- | ----------------- | ------------------------------------------------------------------------------- |
|       |                      |                   | **Unwritable area begins.**                                                     |
| 8     | `uint32`             | writable offset   | Offset from start of this call frame to start of writable area, in bytes.       |
| 8     | `uint32`             | out offset        | Offset from start of this call frame to out count, in bytes.                    |
| 8     | `uint32`             | code offset       | Offset from start of this call frame to code, in bytes.                         |
| 8     | `uint64`             | gas               | Gas remaining from previous call frame after forwarding gas to this call frame. |
| 32    | `byte[32]`           | to                | Contract ID for this call.                                                      |
| 8*64  | `byte[8][64]`        | regs              | Saved registers from previous call frame.                                       |
| 1     | `uint8`              | in count          | Number of input values.                                                         |
| 1     | `uint8`              | out count         | Number of return values.                                                        |
| 2     | `uint16`             | code size         | Code size in bytes (not padded to word alignment).                              |
| 16*   | `(uint32, uint32)[]` | in (addr, size)s  | Array of memory addresses and lengths in bytes of input values.                 |
| 16*   | `(uint32, uint32)[]` | out (addr, size)s | Array of memory addresses and lengths in bytes of return values.                |
| 1*    | `byte[]`             | code              | Zero-padded to 8-byte alignment, but individual instructions are not aligned.   |
|       |                      |                   | **Unwritable area ends.**                                                       |
| *     |                      |                   | Call frame's stack.                                                             |

## Ownership

Whenever memory is written to (i.e. with [`SB`](./opcodes.md#sb-store-byte) or [`SW`](./opcodes.md#sw-store-word)), or write access is granted (i.e. with [`CALL`](./opcodes.md#call-call-contract)), ownership must be checked.

The owned memory range for a call frame is:
1. `[$fpp + MEM[$fpp + 0], $fp)`: the writable stack area of the call frame.
1. `($hp, $hpp]`: the heap area allocated by this call frame or its children.
1. For each `(addr, size)` pair specified as return values in the call frame, the range `[addr, addr + size)`.