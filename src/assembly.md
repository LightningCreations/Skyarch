# Assembly Syntax

## Operands

The following operand syntax types are used in instructions

| Short   | Name             | Description                              |
|---------|------------------|------------------------------------------|
| `GPR`   | GPR operand      | A General Purpose Register               |
| `IGPR`  | Invertible GPR Operand | A potentially inverted gpr operand |
| `SGPR`  | Shifted GPR operand | A General Purpose Register Left-shifted by a constant |
| `SIGPR` | Shifted Invertible GPR operand | A potentially inverted gpr operand Left-shifed by a constant |
| `IOR`   | I/O Register operand | An I/O Transfer Register (Map 3)     |
| `ANYREG`| Any Register     | Any Register operand                     |
| `UIMM16`| Immediate (unsigned 16-bit) | 16-bit Immediate operand      |
| `SIMM16`| Immediate (signed 16-bit) | 16-bit Immediate operand        |
|`PCREL16`| PC Relative Address (16-bit) | 16-bit offset from IP in bytes|
| `OFF20` | Jump Offset (signed 20-bit) | 20-bit jump offset in words   |
| `UIMM32`| Immediate (unsigned 32-bit) | 32-bit immediate operand      |
| `SIMM32`| Immediate (signed 32-bit) | 32-bit immediate operand        |
| `BITW`  | Bit Width        | Width of a value in bits                 |
| `BYTESZ`| Byte Size        | Size of a value in bytes                 |
|`ABSIMM2`| Immediate (unsigned 2-bit) | 2-bit absolute (non-relocated) immediate |
|`ABSIMM3`| Immediate (unsigned 3-bit) | 3-bit absolute (non-relocated) immediate |
|`ABSIMM5`| Immediate (unsigned 5-bit) | 5-bit absolute (non-relocated) immediate |
|`ABSIMM6`| Immediate (unsigned 6-bit) | 6-bit absolute (non-relocated) immediate |
|`ABSIMM8`| Immediate (unsigned 8-bit) | 8-bit absolute (non-relocated) immediate |
| `CC`    | Condition Code Name | A condition Code                      |
| `PRIO`  | Interrupt Priority  | An interrupt priority level           |


### GPR operands

A GPR operand is written as `r` followed by the index number of the register.

### Shifted GPR

A Shifted GPR operand is written as `GPR << ABSIMM5`.

A shifted GPR operand produces two variables. The Operand Specification is written as `<GPR> << <ABS>`

### Invertible GPRs

An invertible GPR is written as either `GPR` (non-inverted) or `~GPR` (inverted). An IGPR produces two variables. The Operand Specification is written as `<INV> <GPR>`.

A Shifted invertible GPR operand is written the same as `GPR << ABSIMM5`, except that `GPR` may be inverted. An SIGPR produces three variables, written as `<INV> <GPR> << <ABS>`

### I/O Register Operand

An I/O Register Operand is written as `io` followed by the index number of the register.

### Any Register

A Register in any map is either a GPR operand, and I/O Register operand, a system configuration register, a system information register, or a coprocessor register.

An interrupt register may be written as `intrN` or an alias name.
 Registers 1-3 may be written as `intretN`, Registers 4-7 are written `intdN` where `N=r%4`, Registers 8-11 are written `intsN` where `N=r%4`.

A system information register is written as `info` followed by the index number of the register. 

A coprocessor register is written as `co` followed by the coprocessor number between 0 and 3, followed by `r`, followed by the register number.
If the Assembler is aware of the particular coprocessor expected in a given coprocessor slot, it may use the alias name provided by the Coprocessor's Assembly Supplement. 
For example, if the Assembler is aware of the presence of a FPU (according to <float-coproc.md>) in slot 0, it may alias `f0` as `co0r0`.

### Immediates

Immediate operands take an integer or a symbol. n-bit Unsigned Immediates require an unsigned quantity in `[0,2^n)`. Signed Immediates require a signed quantity in `[-2^(n-1), 2^(n-1))`. 

Other than Absolute Immediates (like `ABSIMM6`), immediate operands can have symbols. Unless specified with the `@pcrel` modifier, Immediate operands always treat symbols as absolute address in relocations. The relocation requires that the address fits in the specified size of immediate (a link error occurs if it does not). using the special modifiers `HI` and `LO` allows you to instead specify the lower or upper 16-bits of the symbol.

Unless modified, immediate relocations uses (given the appropriate n):
* `R_MICRON_<n>`
* `R_MICRON_LO16` (`LO <sym>`)
* `R_MICRON_HI16` (`HI <sym>`)

With the `@pcrel` modifier (or for `PCREL16`, see below), the relocation uses (given the appropriate n):
* `R_MICRON_PC16` 
* `R_MICRON_LOPC16` (`LO <sym>@pcrel`)
* `R_MICRON_HIPC16` (`HI <sym>@pcrel`)

### Offsets

The `PCREL16` and `OFF15` values are special cases of immediates. 
PCREL16 is identical to a `SIMM16` relocation, except that it defaults to resolving the specified symbol using a pc-relative relocation.

`OFF15` is a 15-bit immediate that resolves a 17-bit pc-relative relocation or a 17-bit signed integer offset, and discards the lower two bits to encode the instruction. It has two constraints, in addition to the constraints that would apply to a 22-bit signed immediate:
* Absolute Expressions must be divisible by 4, and
* Relocation Expressions must produce a 4-byte aligned quantity. The Assembler may error if a misalignment is reliably detected (for example, a 3-byte offset from a symbol known to be 4-byte aligned).

`OFF15` uses `R_MICRON_JMPOFF` relocation.

### Sizes/Widths

A size or width expression is an absolute immediate that encodes a size (in bytes) or a width (in bits). 

A `BITW` operand is a 5-bit unsigned absolute immediate that can take on any value between `1` and `32`.  `32` is encoded as `0`. As a special case, the following synthetic constants are defined for `BITW` operands with the following values:
* `byte`: 8
* `half`: 16
* `word`: 32.

A `BYTESZ` operand is a 2-bit immediate with a special encoding of `lg(sz)` where `sz` is the real size in bytes. The value must be a power of 2 less than 8. The same synthetic constants are defined above to be `1`, `2`, and `4` respectively.

### Interrupt Priorities

An interrupt priority is a priority level for the `IRET` instruction. It may be written as an `ABSIMM2` or one of the following keywords. A value of `0` assembles, but is an illegal instruction at runtime.

* `trap`: 1
* `async`: 2
* `irq`: 3


### Condition Code

Condition Codes appear only in mnemonics, and use the following 1 or 2 character short forms:

| Short Form | Condition Name | Number | Canonical |
|------------|----------------|--------|-----------|
| `NV`       | Never          | 0      | Yes       |
| `C`        | Carry          | 1      | Yes       |
| `B`        | Below          | 1      | No        |
| `Z`        | Zero           | 2      | Yes       |
| `EQ`       | Equal          | 2      | No        |
| `O`        | Overflow       | 3      | Yes       |
| `CE`       | Carry/Equal    | 4      | Yes       |
| `BE`       | Below or Equal | 4      | No        |
| `LT`       | Less           | 5      | Yes       |
| `LE`       | Less or Equal  | 6      | Yes       |
| `N`        | Negative       | 7      | Yes       |
| `S`        | Signed         | 7      | No        |
| `P`        | Positive       | 8      | Yes       |
| `NS`       | Not Signed     | 8      | No        |
| `GT`       | Greater Than   | 9      | Yes       |
| `NLE`      | Not Less or Equal| 9    | No        |
| `GE`       | Greater or Equal | 10   | Yes       |
| `NLT`      | Not Less       | 10     | No        |
| `A`        | Above          | 11     | Yes       |
| `NBE`      | Not Below or Equal | 11 | No        |
| `NCE`      | Not Carry or Equal | 11 | No        |
| `NO`       | Not Overflow   | 12     | Yes       |
| `NZ`       | Not Zero       | 13     | Yes       |
| `NE`       | Not Equal      | 13     | No        |
| `NC`       | Not Carry      | 14     | Yes       |
| `NB`       | Not Below      | 14     | No        |
| `AE`       | Above or Equal | 14     | No        |
| `AL`       | Always         | 15     | Yes       |

## Instruction Syntax

The following charts describes how assemblers should interpret and assemble given instruction syntax forms and mnemonics.

Each Chart has the following information:

* Mnemonic: The name of the instruction, which should be interpreted case-insentively. The special variables `<c>` and `<x>` may be written here. `c` is as if defined `CC <c>` and `x` is as if defined `ABSIMM2 <x>`.
* Operands: A list of operands written as the Short ID followed by a variable in `<>` (e.g. `GPR <d>`) 
* Opcode: The Opcode of the instruction. This may reference the special variable `x` if defined in the Mnemonic.
* Special Payload Encoding: A list of variable assignments of the form `<var>=<val>` where `var` is an encoding variable defined for the opcode in the ISA Spec, and `val` is either an integer expression or a variable. Implicitly the name of each encoding variable that is defined as a syntax variable in the Mnemonic or the Operands list is assigned to the value of that syntax variable. The special encoding variable `P` refers to the entire 24-bit payload (where its contents are undefined/ignored)
* Canonical: Describes whether or not the instruction specification is canonical  Canonical Specifications are the primary (or only) way to describe a particular encoding (without a `.instr` directive). Non-canonical encodings may describe a more efficient way to write or read a particular encoding, or may be useful in niche circumstances. Disassemblers and machine-code generators (such as assembly printing from a compiler) should prefer the canonical specification if it has no context otherwise.

### `UND`

| Mnemonic | Operands        | Opcode | Special Payload Encoding | Canonical |
|----------|-----------------|--------|--------------------------|-----------|
| `UND`    | -               | `0x00` | `P=0`                    | Yes       |

### `PAUSE`

| Mnemonic | Operands        | Opcode | Special Payload Encoding | Canonical |
|----------|-----------------|--------|--------------------------|-----------|
| `PAUSE`  | `ABSIMM6 <k>`   | `0x01` | N/A                      | Yes       |
| `NOP`    | -               | `0x01` | `k=0`                    | No        |

### `MOV`

| Mnemonic | Operands              | Opcode | Special Payload Encoding  | Canonical |
|----------|-----------------------|--------|---------------------------|-----------|
| `MOV<c>` | `GPR <d>, GPR <s>`    | `0x02` | `m=0, r=0, l=0`           | Yes       |
| `MOV<c>` | `GPR <d>, ANYREG <s>` | `0x02` | `m=MAP(s), r=0, l=0`      | Yes       |
| `MOV<c>` | `ANYREG <d>, GPR <s>` | `0x02` | `m=MAP(d), r=1, l=0`      | Yes       |
| `MOV`    | `GPR <d>, GPR <s>`    | `0x02` | `m=0, r=0, l=0, c=15`     | No        |
| `MOV`    | `GPR <d>, ANYREG <s>` | `0x02` | `m=MAP(s), r=0, l=0, c=15`| No        |
| `MOV`    | `ANYREG <d>, GPR <s>` | `0x02` | `m=MAP(d), r=1, l=0, c=15`| No        |
| `MOVL<c>`| `GPR <d>, GPR <s>`    | `0x02` | `m=0, r=0, l=1`           | Yes       |
| `MOVL<c>`| `GPR <d>, ANYREG <s>` | `0x02` | `m=MAP(s), r=0, l=1`      | Yes       |
| `MOVL<c>`| `ANYREG <d>, GPR <s>` | `0x02` | `m=MAP(d), r=1, l=1`      | Yes       |

### `LD`/`ST`

| Mnemonic | Operands                       | Opcode | Special Payload Encoding  | Canonical |
|----------|--------------------------------|--------|---------------------------|-----------|
| `LD`     | `GPR <d>, GPR <s>, BYTESZ <w>` | `0x03` | `p=0`                     | Yes       |
| `ST`     | `GPR <d>, GPR <s>, BYTESZ <w>` | `0x04` | `p=0`                     | Yes       |
| `LD`     | `GPR <d>, GPR <s>`             | `0x03` | `p=0,w=4`                 | No        |
| `ST`     | `GPR <d>, GPR <s>`             | `0x04` | `p=0,w=4`                 | No        |
| `PUSH`   | `GPR <d>, GPR <s>, BYTESZ <w>` | `0x04` | `p=1`                     | Yes       |
| `POP`    | `GPR <d>, GPR <s>, BYTESZ <w>` | `0x03` | `p=1`                     | Yes       |
| `PUSH`   | `GPR <d>, GPR <s>`             | `0x04` | `p=1,w=4`                 | No        |
| `POP`    | `GPR <d>, GPR <s>`             | `0x03` | `p=1,w=4`                 | No        |
| `PUSH`   | `GPR <s>`                      | `0x04` | `p=1,w=4,d=30`            | No        |
| `POP`    | `GPR <d>`                      | `0x03` | `p=1,w=4,d=30`            | No        |

### `ADDI`

| Mnemonic | Operands                       | Opcode | Special Payload Encoding  | Canonical |
|----------|--------------------------------|--------|---------------------------|-----------|
| `ADDI`   | `GPR <d>, SIMM16 <i>`          | `0x08` | `s=1,f=1,h=0`             | Yes       |
| `ADDIF`  | `GPR <d>, SIMM16 <i>`          | `0x08` | `s=1,f=0,h=0`             | Yes       |
| `ADDIU`  | `GPR <d>, UIMM16 <i>`          | `0x08` | `s=0,f=1,h=0`             | Yes       |
| `ADDIH`  | `GPR <d>, UIMM16 <i>`          | `0x08` | `s=0,f=1,h=1`             | Yes       |
| `ADDIH`  | `GPR <d>, SIMM16 <i>`          | `0x08` | `s=0,f=1,h=1`             | No        |
| `ADDIUF` | `GPR <d>, UIMM16 <i>`          | `0x08` | `s=0,f=0,h=0`             | Yes       |
| `ADDIHF` | `GPR <d>, UIMM16 <i>`          | `0x08` | `s=0,f=0,h=1`             | Yes       |
| `ADDIHF` | `GPR <d>, SIMM16 <i>`          | `0x08` | `s=0,f=0,h=1`             | No        |
| `INC`    | `GPR <d>`                      | `0x08` | `s=0,f=1,h=0,i=1`         | No        |
| `DEC`    | `GPR <d>`                      | `0x08` | `s=1,f=1,h=0,i=0xFFFF`    | No        |

### Arithmetic Ops

| Mnemonic | Operands                           | Opcode | Special Payload Encoding  | Canonical |
|----------|------------------------------------|--------|---------------------------|-----------|
| `ADD`    | `GPR <d>, SGPR <a> << <s>, GPR <b>`| `0x09` | `f=1,p=0`                 | Yes       |
| `ADD`    | `GPR <d>, GPR <a>, SGPR <b> << <s>`| `0x09` | `f=1,p=1`                 | Yes       |
| `ADD`    | `GPR <d>, GPR <a>, GPR <b>`        | `0x09` | `f=1,p=0,s=0`             | No        |
| `ADDF`   | `GPR <d>, SGPR <a> << <s>, GPR <b>`| `0x09` | `f=0,p=0`                 | Yes       |
| `ADDF`   | `GPR <d>, GPR <a>, SGPR <b> << <s>`| `0x09` | `f=0,p=1`                 | Yes       |
| `ADDF`   | `GPR <d>, GPR <a>, GPR <b>`        | `0x09` | `f=0,p=0,s=0`             | No        |
| `SUB`    | `GPR <d>, SGPR <a> << <s>, GPR <b>`| `0x0A` | `f=1,p=0`                 | Yes       |
| `SUB`    | `GPR <d>, GPR <a>, SGPR <b> << <s>`| `0x0A` | `f=1,p=1`                 | Yes       |
| `SUB`    | `GPR <d>, GPR <a>, GPR <b>`        | `0x0A` | `f=1,p=0,s=0`             | No        |
| `SUBF`   | `GPR <d>, SGPR <a> << <s>, GPR <b>`| `0x0A` | `f=0,p=0`                 | Yes       |
| `SUBF`   | `GPR <d>, GPR <a>, SGPR <b> << <s>`| `0x0A` | `f=0,p=1`                 | Yes       |
| `SUBF`   | `GPR <d>, GPR <a>, GPR <b>`        | `0x0A` | `f=0,p=0,s=0`             | No        |
| `CMP`    | `SGPR <a> << <s>, GPR <b>`         | `0x0A` | `f=0,p=0,d=0`             | No        |
| `CMP`    | `GPR <a>, SGPR <b> << <s>`         | `0x0A` | `f=0,p=1,d=0`             | No        |
| `CMP`    | `GPR <a>, GPR <b>`                 | `0x0A` | `f=0,p=0,s=0,d=0`         | No        |
| `SHL`    | `GPR <d>, GPR <a>, ABSIMM5 <s>`    | `0x09` | `f=1,p=0,b=0`             | No        |
| `SHLF`   | `GPR <d>, GPR <a>, ABSIMM5 <s>`    | `0x09` | `f=2,p=0,b=0`             | No        |


### Logic Ops

| Mnemonic | Operands                                      | Opcode | Special Payload Encoding  | Canonical |
|----------|-----------------------------------------------|--------|---------------------------|-----------|
| `AND`    | `GPR <d>, SIGPR <i> <a> << <s>, IGPR <j> <b>` | `0x0B` | `f=1,p=0`                 | Yes       |
| `AND`    | `GPR <d>, IGPR <i> <a>, SIGPR <j><b> << <s>`  | `0x0B` | `f=1,p=1`                 | Yes       |
| `AND`    | `GPR <d>, IGPR <i> <a>, GPR <j> <b>`          | `0x0B` | `f=1,p=0,s=0`             | No        |
| `ANDF`   | `GPR <d>, SIGPR <i> <a> << <s>, IGPR <j> <b>` | `0x0B` | `f=0,p=0`                 | Yes       |
| `ANDF`   | `GPR <d>, IGPR <i> <a>, SIGPR <j><b> << <s>`  | `0x0B` | `f=0,p=1`                 | Yes       |
| `ANDF`   | `GPR <d>, IGPR <i> <a>, GPR <j> <b>`          | `0x0B` | `f=0,p=0,s=0`             | No        |
| `OR`     | `GPR <d>, SIGPR <i> <a> << <s>, IGPR <j> <b>` | `0x0C` | `f=1,p=0`                 | Yes       |
| `OR`     | `GPR <d>, IGPR <i> <a>, SIGPR <j><b> << <s>`  | `0x0C` | `f=1,p=1`                 | Yes       |
| `OR`     | `GPR <d>, IGPR <i> <a>, GPR <j> <b>`          | `0x0C` | `f=1,p=0,s=0`             | No        |
| `ORF`    | `GPR <d>, SIGPR <i> <a> << <s>, IGPR <j> <b>` | `0x0C` | `f=0,p=0`                 | Yes       |
| `ORF`    | `GPR <d>, IGPR <i> <a>, SIGPR <j><b> << <s>`  | `0x0C` | `f=0,p=1`                 | Yes       |
| `ORF`    | `GPR <d>, IGPR <i> <a>, GPR <j> <b>`          | `0x0C` | `f=0,p=0,s=0`             | No        |
| `XOR`    | `GPR <d>, SIGPR <i> <a> << <s>, IGPR <j> <b>` | `0x0D` | `f=1,p=0`                 | Yes       |
| `XOR`    | `GPR <d>, IGPR <i> <a>, SIGPR <j><b> << <s>`  | `0x0D` | `f=1,p=1`                 | Yes       |
| `XOR`    | `GPR <d>, IGPR <i> <a>, GPR <j> <b>`          | `0x0D` | `f=1,p=0,s=0`             | No        |
| `XORF`   | `GPR <d>, SIGPR <i> <a> << <s>, IGPR <j> <b>` | `0x0D` | `f=0,p=0`                 | Yes       |
| `XORF`   | `GPR <d>, IGPR <i> <a>, SIGPR <j><b> << <s>`  | `0x0D` | `f=0,p=1`                 | Yes       |
| `XORF`   | `GPR <d>, IGPR <i> <a>, GPR <j> <b>`          | `0x0D` | `f=0,p=0,s=0`             | No        |
| `TEST`   | `SIGPR <i> <a> << <s>, IGPR <j> <b>`          | `0x0B` | `f=1,p=0,d=0`             | No        |
| `TEST`   | `IGPR <i> <a>, SIGPR <j><b> << <s>`           | `0x0B` | `f=1,p=1,d=0`             | No        |
| `TEST`   | `IGPR <i> <a>, GPR <j> <b>`                   | `0x0B` | `f=1,p=0,s=0,d=0`         | No        |



### Shifts

| Mnemonic | Operands                            | Opcode | Special Payload Encoding  | Canonical |
|----------|-------------------------------------|--------|---------------------------|-----------|
| `BSL`    | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0E` | `w=0, x=0,c=1`            | Yes       |
| `BSLW`   | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0E` | `w=1, x=0, c=1`           | Yes       |
| `BSLF`   | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0E` | `w=0,x=0,f=0`             | Yes       |
| `BSLWF`  | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0E` | `w=1,x=0,f=0`             | Yes       |
| `XBSL`   | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0E` | `w=0, x=1,c=1`            | Yes       |
| `XBSLW`  | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0E` | `w=1, x=1, c=1`           | Yes       |
| `XBSLF`  | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0E` | `w=0,x=1,f=0`             | Yes       |
| `XBSLWF` | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0E` | `w=1,x=1,f=0`             | Yes       |
| `SHL`    | `GPR <d>, GPR <v>, GPR <q>`         | `0x0E` | `w=0,x=0,c=1,r=0`         | No        |
| `SHLW`   | `GPR <d>, GPR <v>, GPR <q>`         | `0x0E` | `w=1,x=0,c=1,r=0`         | No        |
| `SHLF`   | `GPR <d>, GPR <v>, GPR <q>`         | `0x0E` | `w=0,x=0,f=0,r=0`         | No        |
| `SHLWF`  | `GPR <d>, GPR <v>, GPR <q>`         | `0x0E` | `w=1,x=0,f=0,r=0`         | No        |
| `ROL`    | `GPR <d>, GPR <v>, GPR <q>`         | `0x0E` | `w=1,x=0,c=1,r=v`         | No        |
| `ROLF`   | `GPR <d>, GPR <v>, GPR <q>`         | `0x0E` | `w=1,x=0,f=0,r=v`         | No        |
| `BSR`    | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0F` | `w=0, x=0,c=1`            | Yes       |
| `BSRW`   | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0F` | `w=1, x=0, c=1`           | Yes       |
| `BSRF`   | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0F` | `w=0,x=0,f=0`             | Yes       |
| `BSRWF`  | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0F` | `w=1,x=0,f=0`             | Yes       |
| `XBSR`   | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0F` | `w=0, x=1,c=1`            | Yes       |
| `XBSRW`  | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0F` | `w=1, x=1, c=1`           | Yes       |
| `XBSRF`  | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0F` | `w=0,x=1,f=0`             | Yes       |
| `XBSRWF` | `GPR <d>, GPR <v>, GPR <q>, GPR <r>`| `0x0F` | `w=1,x=1,f=0`             | Yes       |
| `SHR`    | `GPR <d>, GPR <v>, GPR <q>`         | `0x0F` | `w=0,x=0,c=1,r=0`         | No        |
| `SHRW`   | `GPR <d>, GPR <v>, GPR <q>`         | `0x0F` | `w=1,x=0,c=1,r=0`         | No        |
| `SHRF`   | `GPR <d>, GPR <v>, GPR <q>`         | `0x0F` | `w=0,x=0,f=0,r=0`         | No        |
| `SHRWF`  | `GPR <d>, GPR <v>, GPR <q>`         | `0x0F` | `w=1,x=0,f=0,r=0`         | No        |
| `ROR`    | `GPR <d>, GPR <v>, GPR <q>`         | `0x0F` | `w=1,x=0,c=1,r=v`         | No        |
| `ROLRC`  | `GPR <d>, GPR <v>, GPR <q>`         | `0x0F` | `w=1,x=0,f=0,r=v`         | No        |
| `SAR`    | `GPR <d>, GPR <v>, GPR <q>`         | `0x0F` | `w=0,x=1,c=1,r=0`         | No        |
| `SARW`   | `GPR <d>, GPR <v>, GPR <q>`         | `0x0F` | `w=1,x=1,c=1,r=0`         | No        |
| `SARF`   | `GPR <d>, GPR <v>, GPR <q>`         | `0x0F` | `w=0,x=1,f=0,r=0`         | No        |
| `SARWF`  | `GPR <d>, GPR <v>, GPR <q>`         | `0x0F` | `w=1,x=1,f=0,r=0`         | No        |

### Branches

| Mnemonic | Operands                       | Opcode | Special Payload Encoding  | Canonical |
|----------|--------------------------------|--------|---------------------------|-----------|
| `JL<c>`  | `GPR <l>, OFF15 <o>`           | `0x10` | -                         | Yes       |
| `JLR<c>` | `GPR <l>, GPR <r>`             | `0x11` | -                         | Yes       |
| `JMP<c>` | `OFF15 <o>`                    | `0x10` | `l=0`                     | No        |
| `JMPR<c>`| `GPR <r>`                      | `0x11` | `l=0`                     | No        |
| `JMP`    | `OFF15 <o>`                    | `0x10` | `l=0,c=15`                | No        |
| `JMPR`   | `GPR <r>`                      | `0x11` | `l=0,c=15`                | No        |
| `CALL`   | `GPR <l>, OFF16 <o>`           | `0x10` | `c=15`                    | No        |
| `CALLR`  | `GPR <l>, GPR <r>`             | `0x11` | `c=15`                    | No        |
| `CALL`   | `OFF16 <o>`                    | `0x10` | `l=31,c=15`               | No        |
| `CALLR`  | `GPR <r>`                      | `0x11` | `l=31,c=15`               | No        |
| `IRET`   | `PRIO <p>`                     | `0x12` | -                         | Yes       |

### I/O Transfers

| Mnemonic | Operands                         | Opcode | Special Payload Encoding  | Canonical |
|----------|----------------------------------|--------|---------------------------|-----------|
| `IN`     | `IOR <d>, ABSIMM8 <p>, BITW <w>` | `0x14` | -                         | Yes       |
| `OUT`    | `IOR <s>, ABSIMM8 <p>, BITW <w>` | `0x15` | -                         | Yes       |

### Flags Manipulation

| Mnemonic | Operands                         | Opcode | Special Payload Encoding  | Canonical |
|----------|----------------------------------|--------|---------------------------|-----------|
| `LDFLAGS`| `GPR <d>, ABSIMM5 <f>`           | `0x18` | -                         | Yes       |
| `LDFLAGS`| `GPR <d>`                        | `0x18` | `f=0x1F`                  | No        |
| `STFLAGS`| `GPR <s>, ABSIMM5 <f>`           | `0x19` | -                         | Yes       |
| `STFLAGS`| `GPR <s>`                        | `0x19` | `f=0x1F`                  | No        |
| `XVP`    | -                                | `0x1A` | -                         | Yes       |

### Exchange Register Contents

| Mnemonic   | Operands                         | Opcode | Special Payload Encoding  | Canonical |
|------------|----------------------------------|--------|---------------------------|-----------|
| `XCHG<c>`  | `GPR <a>, GPR <b>`               | `0x1C` | `l=0`                     | Yes       |
| `XCHGL<c>` | `GPR <a>, GPR <b>`               | `0x1C` | `l=1`                     | Yes       |
| `XCHG`     | `GPR <a>, GPR <b>`               | `0x1C` | `c=15,l=1`                | No        |

### Extend Register Contents

| Mnemonic | Operands                         | Opcode | Special Payload Encoding  | Canonical |
|----------|----------------------------------|--------|---------------------------|-----------|
| `EXTS`   | `GPR <d>, GPR <s>, BITW <w>`     | `0x1D` | `x=1`                     | Yes       |
| `EXTZ`   | `GPR <d>, GPR <s>, BITW <w>`     | `0x1D` | `x=0`                     | Yes       |

### Random Bit Generation

| Mnemonic | Operands                         | Opcode | Special Payload Encoding  | Canonical |
|----------|----------------------------------|--------|---------------------------|-----------|
| `RBGEN`  | `GPR <d>, GPR <e>, BITW <w>`     | `0x1E` | -                         | Yes       |
| `RBGEN`  | `GPR <d>, BITW <w>`              | `0x1E` | `e=0`                     | No        |
| `RBGEN`  | `GPR <d>, GPR <e>`               | `0x1E` | `w=0`                     | No        |
| `RBGEN`  | `GPR <d>`                        | `0x1E` | `e=0,w=0`                 | No        |

### Coprocessor Invocations

| Mnemonic   | Operands                       | Opcode    | Special Payload Encoding  | Canonical |
|------------|--------------------------------|-----------|---------------------------|-----------|
| `CPI<x>`   | `ABSIMM4 <f>, VPAYLOAD20 <p>`  | `0x20+x`  | -                         | Yes       |
| `CPI<x>EF` | `ABSIMM6 <f>, VPAYLOAD18 <p>`  | `0x28+x`  | -                         | Yes       |
| `NCPI<x>`  | `ABSIMM4 <f>, VPAYLOAD20 <p>`  | `0x30+x`  | -                         | Yes       |
| `NCPI<x>EF`| `ABSIMM6 <f>, VPAYLOAD20 <p>`  | `0x38+x`  | -                         | Yes       |

### Stop/Halt

| Mnemonic | Operands        | Opcode | Special Payload Encoding | Canonical |
|----------|-----------------|--------|--------------------------|-----------|
| `HLT`    | -               | `0x40` | -                        | Yes       |
| `STP`    | -               | `0x41` | -                        | Yes       |
