# 16-bit Arithmetic Logic Unit (ALU)

## Overview

This is a 16-bit ALU (Arithmetic Logic Unit) that performs both arithmetic and logic operations on two 16-bit inputs (A and B). The ALU supports 8 different operations controlled by a 6-bit opcode.

## Specifications

- **Data Width**: 16 bits
- **Opcode Width**: 6 bits
- **Input Operands**: A, B (16 bits each)
- **Supported Operations**: ADD, SUB, AND, NOT, NAND, OR, NOR, XOR

## Architecture

The ALU consists of two main functional units:

1. **Arithmetic Unit**: Performs addition and subtraction
2. **Logic Unit**: Performs logical operations (AND, NAND, OR, NOR, XOR, NOT)

### Input Conditioning

Before operations are performed, the inputs can be conditionally inverted:
- Input A can be selected as **A** or **!A**
- Input B can be selected as **B** or **!B**

This is controlled by bits 0 and 1 of the opcode using 2-input multiplexers.

## Opcode Format

The 6-bit opcode controls the ALU operation as follows:

| Bit | Name | Function |
|-----|------|----------|
| b0  | SEL_A | Selects A (0) or !A (1) using a 2-input MUX |
| b1  | SEL_B | Selects B (0) or !B (1) using a 2-input MUX |
| b2  | OP_SEL | **Arithmetic**: ADD (0) or SUB (1)<br>**Logic**: MUX selector bit 0 |
| b3  | LOGIC_SEL1 | Logic unit MUX selector bit 1 |
| b4  | LOGIC_SEL2 | Logic unit MUX selector bit 2 |
| b5  | UNIT_SEL | Selects Arithmetic (0) or Logic (1) unit |

### Opcode Bit Assignment
```
  b5   b4   b3   b2   b1   b0
  |    |    |    |    |    |
  |    |    |    |    |    +--- SEL_A: 0=A, 1=!A
  |    |    |    |    +-------- SEL_B: 0=B, 1=!B
  |    |    |    +------------- Arithmetic: 0=ADD, 1=SUB
  |    |    |                   Logic: MUX selector b2
  |    |    +------------------ Logic: MUX selector b3
  |    +----------------------- Logic: MUX selector b4
  +---------------------------- 0=Arithmetic, 1=Logic
```

## Logic Unit Multiplexer

When the Logic Unit is selected (b5 = 1), bits b2, b3, and b4 form a 3-bit selector for an 8-input multiplexer:

| MUX Input | b4 | b3 | b2 | Operation |
|-----------|----|----|-----|-----------|
| in0 | 0 | 0 | 0 | !A |
| in1 | 0 | 0 | 1 | !B |
| in2 | 0 | 1 | 0 | A AND B |
| in3 | 0 | 1 | 1 | A NAND B |
| in4 | 1 | 0 | 0 | A OR B |
| in5 | 1 | 0 | 1 | A NOR B |
| in6 | 1 | 1 | 0 | A XOR B |
| in7 | 1 | 1 | 1 | *Reserved* |

## Operation Truth Table

### Arithmetic Operations

| Operation | b5 | b4 | b3 | b2 | b1 | b0 | Opcode (binary) | Opcode (hex) |
|-----------|----|----|----|----|----|----|-----------------|--------------|
| A + B     | 0  | x  | x  | 0  | 0  | 0  | 000000          | 0x00         |
| A - B     | 0  | x  | x  | 1  | 0  | 0  | 000100          | 0x04         |
| A + !B    | 0  | x  | x  | 0  | 1  | 0  | 000010          | 0x02         |
| !A + B    | 0  | x  | x  | 0  | 0  | 1  | 000001          | 0x01         |

*Note: For arithmetic operations, bits b4 and b3 are don't care (x)*

### Logic Operations

| Operation | b5 | b4 | b3 | b2 | b1 | b0 | Opcode (binary) | Opcode (hex) |
|-----------|----|----|----|----|----|----|-----------------|--------------|
| NOT A (!A)| 1  | 0  | 0  | 0  | 0  | 0  | 100000          | 0x20         |
| NOT B (!B)| 1  | 0  | 0  | 1  | 0  | 0  | 100100          | 0x24         |
| A AND B   | 1  | 0  | 1  | 0  | 0  | 0  | 101000          | 0x28         |
| A NAND B  | 1  | 0  | 1  | 1  | 0  | 0  | 101100          | 0x2C         |
| A OR B    | 1  | 1  | 0  | 0  | 0  | 0  | 110000          | 0x30         |
| A NOR B   | 1  | 1  | 0  | 1  | 0  | 0  | 110100          | 0x34         |
| A XOR B   | 1  | 1  | 1  | 0  | 0  | 0  | 111000          | 0x38         |

*Note: b1 and b0 can be used to invert inputs before logic operations*

## Block Diagram
```
         +------------------+
    A ---| 2:1 MUX (SEL_A) |---+
         +------------------+   |
              (b0)              |     +-------------------+
                                +---->|                   |
                                      |  Arithmetic Unit  |--+
                                +---->|   (ADD/SUB)       |  |
         +------------------+   |     +-------------------+  |
    B ---| 2:1 MUX (SEL_B) |---+              (b2)          |
         +------------------+                               |    +-------+
              (b1)                                          +--->|       |
                                                            |    | 2:1   |---> Result
         +------------------+                               |    | MUX   |    (16-bit)
         |                  |                               +--->|       |
         |  Logic Unit      |-------------------------------+    +-------+
         |  (8:1 MUX)       |                                       (b5)
         |                  |
         +------------------+
           (b4, b3, b2)
```

## Usage Examples

### Example 1: Add A and B
```
Opcode: 000000 (0x00)
Operation: A + B
```

### Example 2: Subtract B from A
```
Opcode: 000100 (0x04)
Operation: A - B
```

### Example 3: AND operation
```
Opcode: 101000 (0x28)
Operation: A AND B
```

### Example 4: XOR operation
```
Opcode: 111000 (0x38)
Operation: A XOR B
```

### Example 5: NOT A
```
Opcode: 100000 (0x20)
Operation: !A
```

## Implementation Notes

- The ALU uses 2-input multiplexers for input conditioning (A/!A and B/!B selection)
- The Logic Unit is implemented using an 8-input multiplexer
- The final output is selected between Arithmetic and Logic units using bit b5
- All operations are performed on 16-bit data

## Contributing

Feel free to contribute to this project by submitting issues or pull requests.

## License

## License

This project is licensed under the Apache License - see the [LICENSE](https://github.com/IR-OTNO-code/16-bit-ALU/blob/main/LICENSE) file for details.
