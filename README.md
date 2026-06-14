# RISC-V Architecture — 20-Day Learning Journey & Interactive Assembler

> A complete self-study of the RISC-V Instruction Set Architecture (ISA) from scratch — covering architecture concepts, all instruction formats, assembly programming, and binary/hex encoding — completed in 20 days, culminating in a fully functional browser-based assembler/disassembler tool.

---

## About This Project

This repository documents my end-to-end study of the RISC-V ISA. Starting with zero prior exposure to RISC-V, I spent 20 days working through the architecture systematically — from the base integer instruction set and register conventions all the way to manually encoding and decoding machine code in binary and hexadecimal. The project concludes with a standalone interactive tool built from scratch that assembles RISC-V instructions into machine code and disassembles machine code back into human-readable assembly.

---

## What I Learned

### Week 1 — Architecture Foundations

**RISC-V Overview**
RISC-V is an open-standard, RISC-based ISA originally developed at UC Berkeley. Unlike proprietary ISAs such as x86 or ARM, RISC-V is freely available and modular. The base integer ISA (RV32I) defines 47 instructions sufficient to implement a complete general-purpose processor. Extensions like M (multiply/divide), A (atomics), F (floating-point), and C (compressed) build on top of it.

**Register File**
RV32I provides 32 general-purpose 32-bit registers (`x0`–`x31`). Key conventions:

| Register | ABI Name | Role |
|----------|----------|------|
| x0 | zero | Hardwired to 0 — writes are discarded |
| x1 | ra | Return address |
| x2 | sp | Stack pointer |
| x5–x7, x28–x31 | t0–t6 | Temporaries (caller-saved) |
| x8–x9, x18–x27 | s0–s11 | Saved registers (callee-saved) |
| x10–x17 | a0–a7 | Function arguments / return values |

`x0` being hardwired to zero is one of RISC-V's most elegant design choices — it eliminates the need for a dedicated `MOV` instruction (`addi x1, x0, 5` loads the immediate 5 into x1).

**Memory Model**
RISC-V uses a byte-addressable, little-endian memory model. Instructions are 32 bits (4 bytes) wide and must be aligned to 4-byte boundaries. Memory is accessed only through load and store instructions — all computation happens in registers (load-store architecture).

---

### Week 2 — Instruction Formats

Every RISC-V instruction is exactly 32 bits. The bit layout is determined by the instruction's **format type**. There are 6 base formats:

#### R-type (Register-to-Register)
Used for arithmetic and logic operations where both operands are registers.

```
[31:25] funct7 | [24:20] rs2 | [19:15] rs1 | [14:12] funct3 | [11:7] rd | [6:0] opcode
```

The opcode is `0110011` for all R-type instructions. `funct3` and `funct7` together select the operation.

Example — `add x1, x2, x3`:
```
0000000 | 00011 | 00010 | 000 | 00001 | 0110011
funct7    rs2     rs1   funct3   rd    opcode
= 0x00310033
```

#### I-type (Immediate)
Used for arithmetic with an immediate value, loads, and `jalr`.

```
[31:20] imm[11:0] | [19:15] rs1 | [14:12] funct3 | [11:7] rd | [6:0] opcode
```

The 12-bit immediate is sign-extended to 32 bits. Range: −2048 to +2047.

Example — `addi x5, x0, 10`:
```
000000001010 | 00000 | 000 | 00101 | 0010011
  imm=10       rs1=x0  f3    rd=x5   opcode
= 0x00A00293
```

#### S-type (Store)
Stores a register value to memory. The immediate is split across two fields to keep `rs1` and `rs2` in consistent bit positions.

```
[31:25] imm[11:5] | [24:20] rs2 | [19:15] rs1 | [14:12] funct3 | [11:7] imm[4:0] | [6:0] opcode
```

Example — `sw x2, 8(x1)` (store word from x2 to address x1+8):
```
imm[11:5]=0000000 | rs2=x2 | rs1=x1 | funct3=010 | imm[4:0]=01000 | 0100011
= 0x0020A423
```

#### B-type (Branch)
Conditional branches. The immediate encodes a PC-relative byte offset. Bit positions are scrambled to maximise overlap with S-type (simplifying hardware).

```
[31] imm[12] | [30:25] imm[10:5] | [24:20] rs2 | [19:15] rs1 | [14:12] funct3 | [11:8] imm[4:1] | [7] imm[11] | [6:0] opcode
```

Note: The immediate has no bit 0 (all branches are to even addresses). Range: −4096 to +4094 bytes.

Example — `beq x1, x2, 16` (branch if x1 == x2, offset +16):
```
offset = 16 = 0b0_0000_0001_0000
imm[12]=0, imm[11]=0, imm[10:5]=000000, imm[4:1]=1000
= 0x00208663
```

#### U-type (Upper Immediate)
Loads a 20-bit immediate into the upper 20 bits of a register. Used for building large constants.

```
[31:12] imm[31:12] | [11:7] rd | [6:0] opcode
```

`lui` (opcode `0110111`) places the immediate in bits 31:12, zeroing bits 11:0.  
`auipc` (opcode `0010111`) adds the immediate to the current PC.

Example — `lui x3, 0x12345`:
```
00010010001101000101 | 00011 | 0110111
       imm             rd=x3   opcode
= 0x123452B7
```

#### J-type (Jump)
Used exclusively by `jal`. Encodes a 21-bit PC-relative offset (bit 0 implicit zero). Bit fields are again reordered to share hardware with other formats.

```
[31] imm[20] | [30:21] imm[10:1] | [20] imm[11] | [19:12] imm[19:12] | [11:7] rd | [6:0] opcode
```

Example — `jal x1, 100` (jump to PC+100, save return address in x1):
```
= 0x064000EF
```

---

### Week 3 — Assembly Programming

**Pseudo-instructions**
The RISC-V assembler defines a set of pseudo-instructions that expand to one or more real instructions:

| Pseudo | Expands to | Effect |
|--------|-----------|--------|
| `li x1, 5` | `addi x1, x0, 5` | Load immediate |
| `mv x1, x2` | `addi x1, x2, 0` | Copy register |
| `nop` | `addi x0, x0, 0` | No operation |
| `ret` | `jalr x0, 0(ra)` | Return from function |
| `j offset` | `jal x0, offset` | Unconditional jump (discard return addr) |
| `bgt rs, rt, L` | `blt rt, rs, L` | Branch if greater than |
| `call label` | `auipc ra, ...; jalr ra, ...` | Far function call |

**Function Call Convention**
```asm
# Caller: pass args in a0–a7, call function
addi a0, x0, 42       # first argument = 42
jal  ra, my_function  # jump and save return address in ra

# Callee: save registers you'll clobber
addi sp, sp, -8       # allocate stack frame
sw   ra, 4(sp)        # save return address
sw   s0, 0(sp)        # save callee-saved reg

# ... body of function ...

lw   ra, 4(sp)        # restore
lw   s0, 0(sp)
addi sp, sp, 8        # free stack frame
jalr x0, 0(ra)        # return
```

**Common Patterns**
```asm
# If-else: if (x5 != x6) goto else_label
bne  x5, x6, else_label
# ... then body ...
jal  x0, end_label
else_label:
# ... else body ...
end_label:

# Loop: for (i=0; i<10; i++)
addi x10, x0, 0      # i = 0
addi x11, x0, 10     # limit = 10
loop:
  bge  x10, x11, done  # if i >= 10, exit
  # ... loop body ...
  addi x10, x10, 1     # i++
  jal  x0, loop
done:

# Array access: a[i] where a is at base address in x12, i in x10
slli x13, x10, 2     # byte offset = i * 4
add  x13, x13, x12   # address = base + offset
lw   x14, 0(x13)     # load a[i]
```

---

### Week 4 — Binary and Hex Encoding (Manual Encoding Drill)

The final week focused on encoding and decoding instructions by hand, understanding exactly how the CPU decodes a 32-bit word. Key insight: the opcode is always in bits [6:0], which means a CPU can determine the instruction format after reading just the first 7 bits.

**Decoding algorithm:**
1. Read bits [6:0] → opcode → determines format (R/I/S/B/U/J)
2. Read bits [14:12] → funct3 → narrows down the operation
3. For R-type, read bits [31:25] → funct7 → final disambiguation
4. Extract operand fields based on the format
5. Sign-extend immediates where applicable

**Sign extension** is critical: a 12-bit immediate of `0b111111111111` is not 4095 — it is −1 when sign-extended to 32 bits.

---

## The Assembler Tool

### What It Does

`riscv_assembler.html` is a fully self-contained, browser-based tool with no dependencies and no server required. It supports:

- **Assembly → Machine Code**: Type one or more instructions, get hex, decimal, and binary output with a visual bit-field breakdown
- **Disassembly → Assembly**: Paste a 32-bit hex or binary word to recover the assembly instruction and field breakdown
- **ISA Reference**: Built-in format diagrams and opcode/funct table

### Supported Instructions

| Format | Instructions |
|--------|-------------|
| R-type | `add`, `sub`, `sll`, `slt`, `sltu`, `xor`, `srl`, `sra`, `or`, `and`, `mul`, `mulh`, `div`, `rem` |
| I-type (arith) | `addi`, `slti`, `sltiu`, `xori`, `ori`, `andi`, `slli`, `srli`, `srai` |
| I-type (load) | `lb`, `lh`, `lw`, `lbu`, `lhu` |
| I-type (jump) | `jalr`, `ecall` |
| S-type | `sb`, `sh`, `sw` |
| B-type | `beq`, `bne`, `blt`, `bge`, `bltu`, `bgeu` |
| U-type | `lui`, `auipc` |
| J-type | `jal` |

Accepts both `x0`–`x31` and ABI register names (`zero`, `ra`, `sp`, `a0`–`a7`, `s0`–`s11`, `t0`–`t6`).

### How to Use

1. Download `riscv_assembler.html`
2. Open it in any modern browser (Chrome, Firefox, Edge, Safari)
3. No installation, no internet connection required after download

### Example

Input:
```
add x1, x2, x3
addi x5, x0, 10
sw x2, 8(x1)
beq x1, x2, 16
lui x3, 0x12345
jal x1, 100
```

Output:

| Instruction | Format | Hex | Binary |
|------------|--------|-----|--------|
| `add x1, x2, x3` | R | `0x00310033` | `00000000001100010000000010110011` |
| `addi x5, x0, 10` | I | `0x00A00293` | `00000000101000000000001010010011` |
| `sw x2, 8(x1)` | S | `0x0020A423` | `00000000001000001010010000100011` |
| `beq x1, x2, 16` | B | `0x00208663` | `00000000001000001000011001100011` |
| `lui x3, 0x12345` | U | `0x123452B7` | `00010010001101000101000101110111` |
| `jal x1, 100` | J | `0x064000EF` | `00000110010000000000000011101111` |

---

## Repository Structure

```
.
├── README.md                  ← this file
└── riscv_assembler.html       ← standalone assembler/disassembler tool
```

---

## Key Takeaways

After 20 days of focused study, the most important things I learned:

**On the architecture:** RISC-V's design is remarkably clean. The fixed 32-bit instruction width, the consistent placement of `rs1`, `rs2`, and `rd` across formats, and the hardwired-zero register all reduce hardware complexity. Every design decision has a clear rationale.

**On instruction encoding:** The bit scrambling in B-type and J-type immediates (which initially seemed arbitrary) exists specifically to maximise shared circuitry with S-type and I-type instructions — the hardware can reuse the same wires for immediate extraction across formats.

**On assembly programming:** The lack of complex addressing modes (unlike x86) actually makes the code easier to reason about. Every memory access is explicit, and the calling convention is simple and regular.

**On the tool:** Building the assembler from scratch forced me to deeply understand every field of every instruction format. Debugging incorrect encodings — especially for B-type and J-type immediates — was the single most valuable learning exercise of the entire 20 days.

---

## References

- [The RISC-V Instruction Set Manual, Volume I: Unprivileged ISA](https://riscv.org/specifications/)
- [RISC-V Assembly Programmer's Manual](https://github.com/riscv-non-isa/riscv-asm-manual/blob/main/riscv-asm.md)
- [CS61C — Great Ideas in Computer Architecture (UC Berkeley)](https://cs61c.org/)
- [Digital Design and Computer Architecture: RISC-V Edition — Harris & Harris](https://www.elsevier.com/books/digital-design-and-computer-architecture-risc-v-edition/harris/978-0-12-820064-3)

---

## Author

Jotham — ECE VLSI student with a focus on computer architecture and digital design.  
This project was completed as part of self-directed study in processor architecture and low-level systems.

---

