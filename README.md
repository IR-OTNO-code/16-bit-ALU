# 16-bit Arithmetic Logic Unit (ALU)

A fully functional 16-bit ALU designed and simulated in [Logisim Evolution](https://github.com/logisim-evolution/logisim-evolution). Supports 11 arithmetic and logic operations controlled by a 6-bit opcode, with a 4-bit status flag output (NZCV).

---

## Table of Contents

- [Overview](#overview)
- [Specifications](#specifications)
- [Architecture](#architecture)
- [Opcode Format](#opcode-format)
- [Operation Reference](#operation-reference)
- [Flag Register](#flag-register)
- [Usage Examples](#usage-examples)
- [Testing](#testing)
- [Implementation Notes](#implementation-notes)

---

## Overview

The ALU takes two 16-bit inputs (`in0`, `in1`) and a 6-bit opcode, and produces a 16-bit result (`Data_out`) along with a 4-bit flag register reflecting the state of the result.

Input conditioning allows either operand to be inverted before it reaches the functional units, enabling operations like `NOT A`, `NOT B`, `A + !B`, and `!A + B` without dedicated inversion hardware inside each unit.

---

## Specifications

| Property | Value |
|---|---|
| Data width | 16 bits |
| Opcode width | 6 bits |
| Flag output | 4 bits (NZCV) |
| Input operands | `in0` (A), `in1` (B) |
| Simulator | Logisim Evolution v4.0.0 |

---

## Architecture

The ALU is composed of three hierarchical subcircuits:

```
              ┌─────────────────────────────────────────────┐
              │                   ALU                        │
              │                                              │
  in0 ──┬──► [MUX A] ──► a_cond ──┬──► [AU] ──► au_result  │
        │     (SEL_A)              │                 │       │
        │                         └──► [LU] ──► lu_result   │
  in1 ──┼──► [MUX B] ──► b_cond ──┘         │       │       │
        │     (SEL_B)                        │    [MUX OUT]──┼──► Data_out
        │                                    │       │       │
  opcode┴────────────────────────────────────┘   [FLAGS] ───┼──► flag
        │                                                    │
        └────────────────────────────────────────────────────┘
```

### Subcircuits

**ALU (top level)** — Routes operands through input conditioning MUXes, dispatches to AU or LU, selects the final output, and assembles the flag register.

**AU (Arithmetic Unit)** — A ripple-carry adder built from 16 chained 1-bit adders. Computes `a_cond + b_cond + C_in`. Subtraction is performed via two's complement: `A − B = A + ~B + 1`. Outputs the 16-bit result, carry-out (C), and overflow (V).

**LU (Logic Unit)** — An 8-input, 16-bit-wide multiplexer. The 3-bit selector (opcode bits `b3b2b1`) chooses from eight logic functions applied to the conditioned inputs.

### Input Conditioning

Before any operation, each operand passes through a 2-to-1 MUX:

| Opcode bit | Controls | `0` → | `1` → |
|---|---|---|---|
| `b5` | SEL_A | A passes through | A is inverted (`!A`) |
| `b4` | SEL_B | B passes through | B is inverted (`!B`) |

The conditioned values `a_cond` and `b_cond` are fed into both the AU and LU.

---

## Opcode Format

```
  b5   b4   b3   b2   b1   b0
  │    │    │    │    │    │
  │    │    │    │    │    └─── Unit select: 0 = Arithmetic, 1 = Logic
  │    │    │    │    └──────── Logic MUX selector, bit 1
  │    │    │    └─────────── Logic MUX selector, bit 2
  │    │    └──────────────── Arithmetic: C_in (0 = ADD, 1 = SUB)
  │    │                      Logic: MUX selector, bit 3
  │    └───────────────────── SEL_B: 0 = B,  1 = !B
  └────────────────────────── SEL_A: 0 = A,  1 = !A
```

> **Note:** For arithmetic operations (`b0 = 0`), bits `b2` and `b1` are don't-cares.  
> For logic operations (`b0 = 1`), bit `b3` doubles as MUX selector bit 2 (MSB of the 3-bit select).

---

## Operation Reference

### Arithmetic Operations

| Operation | b5 | b4 | b3 | b2 | b1 | b0 | Opcode |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| A + B | 0 | 0 | 0 | 0 | 0 | 0 | `000000` |
| A − B | 0 | 1 | 1 | 0 | 0 | 0 | `011000` |
| !A + B | 1 | 0 | 0 | 0 | 0 | 0 | `100000` |
| A + !B | 0 | 1 | 0 | 0 | 1 | 0 | `010010` |

Subtraction `A − B` works by conditioning B as `!B` (`b4 = 1`) and setting `C_in = 1` (`b3 = 1`), so the AU computes `A + ~B + 1`.

### Logic Operations

| Operation | b5 | b4 | b3 | b2 | b1 | b0 | Opcode |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| NOT A (`!A`) | 1 | 0 | 0 | 0 | 0 | 1 | `100001` |
| NOT B (`!B`) | 0 | 1 | 0 | 0 | 1 | 1 | `010011` |
| A AND B | 0 | 0 | 0 | 1 | 0 | 1 | `000101` |
| A NAND B | 0 | 0 | 0 | 1 | 1 | 1 | `000111` |
| A OR B | 0 | 0 | 1 | 0 | 0 | 1 | `001001` |
| A NOR B | 0 | 0 | 1 | 0 | 1 | 1 | `001011` |
| A XOR B | 0 | 0 | 1 | 1 | 0 | 1 | `001101` |

For NOT A and NOT B, the inversion is handled upstream by the SEL_A/SEL_B conditioning MUXes. The LU MUX at `sel = 000` and `sel = 001` simply passes `a_cond` and `b_cond` through respectively — the negation has already been applied.

### Logic Unit MUX Map

When `b0 = 1`, bits `b3b2b1` form a 3-bit selector for the LU:

| sel (b3 b2 b1) | Operation on conditioned inputs |
|:---:|---|
| `000` | Pass `a_cond` (used for NOT A when `b5=1`) |
| `001` | Pass `b_cond` (used for NOT B when `b4=1`) |
| `010` | `a_cond AND b_cond` |
| `011` | `a_cond NAND b_cond` |
| `100` | `a_cond OR b_cond` |
| `101` | `a_cond NOR b_cond` |
| `110` | `a_cond XOR b_cond` |
| `111` | Reserved |

---

## Flag Register

The 4-bit flag output uses the **NZCV** format:

| Bit | Name | Description |
|:---:|---|---|
| 3 (MSB) | **N** — Negative | Set when `Data_out[15] = 1` (result is negative in two's complement) |
| 2 | **Z** — Zero | Set when `Data_out = 0x0000` |
| 1 | **C** — Carry | Set when an arithmetic operation produces a carry out of bit 15. Always `0` for logic operations. |
| 0 (LSB) | **V** — Overflow | Set when an arithmetic operation produces a signed overflow (carry into bit 15 ≠ carry out of bit 15). Always `0` for logic operations. |

> **C and V are always `0` for logic operations.** The AU runs continuously in the background, but its C and V outputs are gated off by `AND(NOT b0)` before they reach the flag register, ensuring clean flag behaviour for all logic ops.

### Flag Examples

| Operation | Result | N | Z | C | V | Flag |
|---|---|:---:|:---:|:---:|:---:|:---:|
| `0x0001 + 0x0001` | `0x0002` | 0 | 0 | 0 | 0 | `0000` |
| `0x0000 + 0x0000` | `0x0000` | 0 | 1 | 0 | 0 | `0100` |
| `0x7FFF + 0x0001` | `0x8000` | 1 | 0 | 0 | 1 | `1001` |
| `0xFFFF + 0x0001` | `0x0000` | 0 | 1 | 1 | 0 | `0110` |
| `0x8000 + 0x8000` | `0x0000` | 0 | 1 | 1 | 1 | `0111` |
| `0xFFFF AND 0xFFFF` | `0xFFFF` | 1 | 0 | 0 | 0 | `1000` |
| `0x5A5A XOR 0xA5A5` | `0xFFFF` | 1 | 0 | 0 | 0 | `1000` |

---

## Usage Examples

### ADD: `A + B`
```
in0    = 0000 0000 0000 0101   (5)
in1    = 0000 0000 0000 0011   (3)
opcode = 000000
──────────────────────────────
Data_out = 0000 0000 0000 1000  (8)
flag     = 0000  (N=0, Z=0, C=0, V=0)
```

### SUB: `A − B`
```
in0    = 0000 0000 0000 0101   (5)
in1    = 0000 0000 0000 0011   (3)
opcode = 011000
──────────────────────────────
Data_out = 0000 0000 0000 0010  (2)
flag     = 0010  (N=0, Z=0, C=1, V=0)
```

### NOT A
```
in0    = 0101 1010 0101 1010   (0x5A5A)
in1    = (don't care)
opcode = 100001
──────────────────────────────
Data_out = 1010 0101 1010 0101  (0xA5A5)
flag     = 1000  (N=1, Z=0, C=0, V=0)
```

### AND
```
in0    = 1111 0000 1111 0000   (0xF0F0)
in1    = 1111 1111 0000 0000   (0xFF00)
opcode = 000101
──────────────────────────────
Data_out = 1111 0000 0000 0000  (0xF000)
flag     = 1000  (N=1, Z=0, C=0, V=0)
```

### XOR
```
in0    = 0101 1010 0101 1010   (0x5A5A)
in1    = 1010 0101 1010 0101   (0xA5A5)
opcode = 001101
──────────────────────────────
Data_out = 1111 1111 1111 1111  (0xFFFF)
flag     = 1000  (N=1, Z=0, C=0, V=0)
```

---

## Testing

A Logisim Evolution test vector file is included: `alu_test_vectors_v6.txt`

It covers **56 test cases** across all 11 operations:

| Category | Operations | Vectors |
|---|---|:---:|
| Arithmetic | ADD, SUB | 14 |
| Arithmetic (conditioned) | !A + B, A + !B | 6 |
| Bitwise logic | AND, NAND, OR, NOR, XOR | 28 |
| Inversion | NOT A, NOT B | 7 |
| **Total** | | **56** |

### Running the Tests

1. Open `logism_Simulation.circ` in Logisim Evolution v4.0.0+
2. Navigate to the `ALU` circuit
3. Go to **Simulate → Test Vector...**
4. Load `alu_test_vectors_v6.txt`
5. All 56 vectors should report **Pass**

### Test Vector Format

```
# in0[16]  in1[16]  opcode[6]  Data_out[16]  flag[4]
# All values in binary. Flag = NZCV (N=bit3, Z=bit2, C=bit1, V=bit0)

0000000000000001 0000000000000001 000000 0000000000000010 0000
```

---

## Implementation Notes

- The ripple-carry adder in the AU is built from 16 individual 1-bit adders chained LSB to MSB (adder at Logisim `y=990` is bit 0; adder at `y=120` is bit 15).
- The **V (overflow)** flag is computed as `carry_into_bit15 XOR carry_out_of_bit15`, tapping the carry outputs of the bit-14 and bit-15 adders respectively.
- The **Z (zero)** flag is derived from a 16-input NOR gate applied to all bits of `Data_out`.
- The **N (negative)** flag is taken directly from `Data_out[15]` (the MSB).
- **C and V are gated to `0`** for logic operations using `AND(NOT b0)` on the AU's C and V outputs before they enter the flag combiner. This prevents the AU's background arithmetic from corrupting the flags during logic ops.
- The LU is implemented as a single 8-to-1 multiplexer, 16 bits wide.
- Input conditioning (SEL_A, SEL_B) is shared between the AU and LU — the same conditioned operands feed both units simultaneously.

---

## License

This project is licensed under the [Apache License 2.0](LICENSE).
