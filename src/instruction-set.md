# Skyarch Instruction Set

## Gen Design

Word Size: 32 bit

Instruction Size/Alignment: 4 bytes

Variable Sized Instructions: No

Common Flags/Condition Code Driven

## Instruction format

8-bit (LSB) opcode followed 24-bit payload.

LSB First Order

## Registers

### Register Maps

There are 8 maps of registers:
* Map 0: General Purpose
* Map 1: System Configuration
* Map 2: I/O Transfer Registers
* Map 3: Information
* Map 4: Coprocessor Control
* Maps 8-15: Co-processor Registers.

There are 32 registers of each type. Except for Map 0, not all registers may be defined.

Co-processor Registers are available only if the applicable co-processors are connected.

### Map 0: General Purpose

Assembly syntax: `r`*`n`*.

All Registers of the Map are defined. Certain Registers have special meaning:
* `r0` is the zero-register. It reads as zero, and writes are ignored.

### Map 1: Interrupt Support

Assembly syntax: `int`*`n`* or alias.

Refer to the following table of defined registers. Some registers define a specific format



| Regno. | Aliases  | Description               |
| ------ | -------- | ------------------------- |
| 0      | `intctl` | Interrupt Status Register |
| 1      | `intret1`| Return for Priority 1 Interrupts |
| 2      | `intret2`| Return for Priority 2 Interrupts |
| 3      | `intret3`| Return for Priority 3 Interrupts |
| 4      | `ints0`  | Scratch Register          |
| 5      | `ints1`  | Scratch Register          |
| 6      | `ints2`  | Scratch Register          |
| 7      | `ints3`  | Scratch Register          |
| 8      | `intd0`  | Misc Config Register      |
| 9      | `intd1`  | Misc Config Register      |
| 10     | `intd2`  | Misc Config Register      |
| 11     | `intd3`  | Misc Config Register      |
| 31     | `inttab` | Interrupt Table Pointer   |

Reading or Writing an undefined register causes `EX[2]`. Writing an invalid value to a defined register causes `EX[5]`

#### Interrupt Control (Map 1, Register 0)

Format:
```
+0-----------------------------31+
|mm00000000000000000000000000000a|
+--------------------------------+
```

(All bits indicated as 0 must be written with 0)


| Bits | Name           | Description                                    |
| ---- | -------------- | ---------------------------------------------- |
| `m`  | Priority Mask  | Interrupts with priority value > m are blocked |
| `a`  | Abort Triggered| Set to 1 when an Abort (Ex[0]) occurs.         |

Both fields are set to `0` on startup.

###### Interrupt Priority

Interrupt Priority is used to ensure that overlapping Interrupts do not interefere.
There are 4 Priority levels, numbered in descending order of priority (0 is the highest priority, 3 is the lowest priority)

* Priority 0: Abort (Ex[0])
* Priority 1: Synchronous Exceptions (Ex[1], Ex[2], Ex[3], Ex[4])
* Priority 2: Asynchronous High Priority Event (Ex[7], EX[8-15])
* Priority 3: IRQs

An interrupt/trap is blocked when the priority level is less than `m`. The behaviour depends on the kind of exception:

* Synchronous Exceptions (other than Abort) Reset the processor if `a = 1`, else they set `a = 1` and raise `Ex[0]`
* Asynchronous Events are discarded
* IRQs are buffered (up to an implementation-specific capacity until an `intret` occurs that sets `m` to be `3`) or are discarded.


#### Interrupt/Exception Table (Map 1, Register 3)

Format:
```
+0-----------------------------32+
|000aaaaaaaaaaaaaaaaaaaaaaaaaaaaa|
+--------------------------------+
```

(All bits indicated as 0 must be written with 0)

Bits `a` contain the 29 most significant bits of an 8-byte aligned address which points to the interrupt table. 512 bytes starting from this address refer to 64 8-byte entries of the interrupt table, which use the following format, in LSB-first order using little-endian byte encoding:
```
+0-----------------------------31+
|p0tttttttttttttttttttttttttttttt|
+32----------------------------63+
|00000000000000000000000000000000|
+--------------------------------+
```

The `t` bits are the 30 most significant bits of the address to transfer control to when the specified interrupt occurs.

The `p` bit must be set for all interrupt vectors that are present and valid to execute.

##### Interrupts

The first 16 interrupt entries are reserved for hardware exceptions, these interrupts are allocated as follows (and the `n`th entry in this list is designated elsewise as `EX[n]`):

* Entry `0`: Exception Handling Fault - an exception is raised when the `t` flag is set. 
* Entry `1`: Bus Fault - accessing memory in a particular manner causes an error, or attempts to access memory that doesn't exist.
* Entry `2`: Invalid Instruction - An instruction that is executed is an unknown opcode, reserved, malformed, or invalid
* Entry `3`: Unaligned Branch Target - an indirect branch is unaligned.
* Entry `4`: Consistency - An invalid system control structure was loaded from memory, or an invalid value was written to a system register.
* Entry `7`: Non-maskable Interrupt - May be raised in response to a priority signal external to the processor that requires immediate resolution. This is handled like an IRQ, but does not obey the `i` flag. 
* Entries `8`-`15`: Co-processor Unit `n` Error - The corresponding Coprocessor unit `n` signals an error after a `CPIn` instruction (`n` is Exception number - 4).
* Entries `5`, `6`, and `16`-`31` are reserved.

The remaining entries (32-63), may be allocated as IRQ vectors.

Exceptions are raised regardless of the `i` bit. The `t` bit is set to `1` when an exception is raised. It is not modified by any other interrupt (including an NMI) being raised.

If an exception occurs raising `EX[0]`, the processor RESETs. 

### Map 2: I/O Transfer Registers

Map 2 defines a sequence of input and output shift registers for transfering data to external peripherals. 

### Map 3: Information Registers

The Information Registers Map is a Read Only Map that contains information about the CPU. All Registers Presently Read 0. Writes are illegal and raise `EX[2]`

### Map 4: Coprocessor Control

Each Co-processor has a 32-bit control word, which is defined by the Coprocessor.

Register N in Map 4 is defined if Co-processor N is present and enabled. 

Reads and writes to an undefined register or a register corresponding to a not-present or disabled coprocessor results in `EX[2]`.

#### Map 4, Register 30: Coprocessor Enable

The Coprocessor Enable register allows the system software to control what coprocessors are operating and usable from the CPU.

Format:
```
+0-----------------------------32+
|EEEEEEEE000000000000000000000000|
+--------------------------------+
```

The bits marked `E` may be set by the program when the corresponding bit of Register 31 is set. Setting the nth bit to 1 enables the coprocessor and setting it to 0 disables it.

Bits marked as 0 must not be written with 1.

#### Map 4, Register 31: Coprocessor Present

The Coprocessor Enable register allows the system software to determine what coprocessors are connected to the CPU. This register is read-only and cannot be written from the CPU. Attempting such a write with a MOV instruction raises `EX[2]`.

Format:
```
+0-----------------------------32+
|PPPPPPPP000000000000000000000000|
+--------------------------------+
```

The nth bit is set to 1 if the nth coprocessor is present. Note that it is not guaranteed that the set of enabled coprocessors is contiguous or that the set of enabled coprocessors begins at 0.


### Map 8-15: Co-processor Maps

Co-processors connected to the system may expose up to 32 registers each. Registers in map `N` are only defined if the coprocessor co-processor (Co-processor N-8) is enabled.

## Instructions

### Undefined Instructions

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| UND      | `0x00` | -                          |
| UND      | `0xFF` | -                          |

(The Payload bits are ignored by both instructions)

Timing (Execute Latency): 0 cycles

Exception Order:
* `EX[2]` (decode): Unconditionally

Behaviour: Unconditionally raises Invalid Instruction errors

```
instruction UND():
    Raise(EX[2])
```

### Pause

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| PAUSE    | `0x01` | `kkkkkk000000000000000000` |


Timing (Execute Latency): 0 cycles + k

Behaviour: Delays execution for `k` cycles, 0-63


```
instruction PAUSE(k: u6):
    SuspendForClockTicks(k)
```

### Move 

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------32` |
| MOV      | `0x02` | `dddddsssss00lccccrmmmm00` |

Payload Bits Legend:
* `d`: Destination Register
* `s`: Source Register
* `m`: Map
* `r`: Direction
* `c`: Condition Code (See Jump)
* `l`: Latency Control

Timing (Execute Latency): `1+c+t`, where:
* `c` is 0 if Condition Code is 0, 1 if Condition Code is 15 or Latency Control is 0 and the Condition Check Fails, 2 if Latency Control is 1 or the Condition Code is not 15 and the Condition Check Succeeds
* `t` is 0 if Map is 0 or Latency Control is 0 and the Condition Check Fails, 2 if Map is not 0 when Latency Control is 1 or the Condition Check Succeeds.

Behaviour: Copies data between general purpose registers and to/from general purpose registers into other registers.

```
instruction MOV(d: u5, s: u5, m: u2, dir: u1, c: ConditionCode, l: bool):
    if m!=0:
        if dir==0:
            ValidateRegisterReadable(m,d);
        else:
            ValidateRegisterWritable(m,d);
    if CheckCondition(flags, c):
        let ms, md: u2;
        if dir==1:
            md = m;
            ms = 0;
        else:
            ms = m;
            md = 0;
        if md==3:
            Raise(EX[2]);
        let val: u32;
        val = ReadRegister(ms, s);
        if md == 2 or m > 3:
            ValidateConfigurationRegisterValue(d, val);
        WriteRegister(md, d, val);
    else:
        if l:
            if m!=0:
                SuspendForClockTicks(4);
            else:
                SuspendForClockTicks(2);
```

### LD/ST

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------32` |
| `ST`     | `0x03` | `dddddsssssww0000000000mm` |
| `LD`     | `0x04` | `dddddsssssww0000000000mm` |
| `LDI`    | `0x05` | `dddddx00iiiiiiiiiiiiiiii` |
| `LRA`    | `0x06` | `dddddx00oooooooooooooooo` |


Payload Bits Legend:
* `s`: Source Register
* `d`: Destination Register
* `w`: Width
* `i`: Immediate Value
* `o`: Offset
* `q`: Scale Quantity
* `m`: Update mode
* `x`: Sign/Zero Extend

Timing: 

* `ST`, `LD`: 4 Cycles, plus Memory Delay
* `LDI`, `LRA`: 1 Cycle

Behaviour:
* `ST`: Stores `1 << w` bytes from `d` to `[s]`
* `LD`: Loads `1 << w` bytes from `[s]` into `d`
* `LDI`: Loads an immediate `i` (sign or zero exteneded) into the first (h=0) 16 bits of `d`
* `LRA`: Loads the address `IP + o` (`o` is a signed immediate if `x` is true, and an unsigned immediate otherwise) into `d`. `IP` is taken from the beginning of the next instruction

```

enum UpdateMode:
    None = 0,
    PostInc = 1,
    /* Illegal = 2 */,
    PreDec = 3,

instruction ST(s: u5, d: u5, w: u2, m: UpdateMode):
    if m == 2:
        Raise(EX[2])
    if d==0:
       Raise(EX[2]);
    let val = ReadRegister(0,s);
    let addr: u32;
    let width = 2 << w;
    if (m&2)== 2:
        addr = ReadRegister(0, d) - width;
    else:
        addr = ReadRegister(0, d);
    if width == 8:
        Raise(EX[2]);
    if addr & (width - 1):
        Raise(Ex[1])

    WriteAlignedMemoryTruncate(addr, val, width);
    let new_addr: u32;
    if m == 1:
        new_addr = addr + width;
    if m != 0:
        WriteRegister(0, d, new_addr);
    
    
instruction LD(s: u5, d: u5,w: u2, p: u2):
    if s==0:
        Raise(EX[2])
    let width = 2 << w;
    if width == 8:
            Raise(EX[2]);
    let addr: u32;
    if (m&2)== 2:
        addr = ReadRegister(0, s) - width;
    else:
        addr = ReadRegister(0, s);
    if addr & (width - 1):
        Raise(Ex[1])
    let val = ReadAlignedMemoryZeroExtend(addr, w+1);
    WriteRegister(0,d,val);
    let new_addr: u32;
    if m == 1:
        new_addr = addr + width;
    if m != 0:
        WriteRegister(0, s, new_addr);
    
instruction LRA(d: u5, x: bool, i: u15):
    let val = SignExtendOrZeroExtend(i, x) + IP;
    WriteRegister(0,d,val);
```

### Immediate Arithmetic

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| `ADDI`   | `0x08` | `dddddschiiiiiiiiiiiiiiii` |

Timing: 2
Payload Bits Legend:
* `d`: Destination Register
* `h`: High half
* `s`: Extend Sign
* `c`: Supress Flags Modification
* `i`: Immediate

Behaviour: Adds a 12-bit zero or sign-extended immediate to `d`.

### ALU Instructions

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| `ADD`    | `0x09` | `dddddaaaaabbbbbcsssssp00` |
| `SUB`    | `0x0A` | `dddddaaaaabbbbbcsssssp00` |
| `AND`    | `0x0B` | `dddddaaaaabbbbbcsssssp00` |
| `OR`     | `0x0C` | `dddddaaaaabbbbbcsssssp00` |
| `XOR`    | `0x0D` | `dddddaaaaabbbbbcsssssp00` |

Timing: 2

Payload Bits Legend:
* `a`: Source Register 1
* `b`: Source Register 2
* `d`: Destination Register
* `c`: Suppress Condition
* `p`: Shift Polarity
* `s`: Shift Quantity

Behaviour:
```
instruction {ADD, SUB, AND, OR, XOR}(a: u5, b: u5, d: u5, c: bool, s: u5, p: bool):
    let src1, src2: u32;
    if p:
        src1 = ReadRegister(0, a) << s;
        src2 = ReadRegister(0,b);
    else:
        src1 = ReadRegister(0, a);
        src2 = ReadRegister(0,b) << s;
    let dest: u32;
    let flags_val, flags_mask: u4;
    switch (instruction):
        case ADD:
            dest, flags_val = src1 + src2;
            flags_mask = 0xF;
        case SUB:
            dest, flags_val = src1 - src2;
            flags_mask = 0xF;
        case AND:
            dest = src1 & src2;
            flags_val = LogicCondition(dest);
            flags_mask = 0x3;
        case OR:
            dest = src1 | src2;
            flags_val = LogicCondition(dest);
            flags_mask = 0x3;
        case XOR:
            dest = src1 ^ src2;
            flags_val = LogicCondition(dest);
            flags_mask = 0x3;
    if not c:
        SetFlagsRegisterByMask(flags_mask, flags_val);

```

### Funnel Shifts

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| `FSL`    | `0x0E` | `dddddvvvvvqqqqqcx0wrrrrr` |
| `FSR`    | `0x0F` | `dddddvvvvvqqqqqcx0wrrrrr` |

Timing: 3

Payload Bits Legend:

* `d`: Destination Register
* `v`: Input Value
* `q`: Shift Quantity
* `c`: Suppress Condition
* `w`: Wrap Quantity
* `r`: Shift Remainder (Input value)
* `x`: Invert by Sign

Behaviour: Shifts `v` by `q` and places the value in `d`, filling the shifted in bits with bits taken from the corresponding high bits of `r`.

```
instruction FSL(d: u5, v: u5, q: u5, c: bool, x: bool, w: bool, r: u5):
    let val = ReadRegister(0, v);
    let quantity = ReadRegister(0, q);
    let remainder = ReadRegister(0, r);
    if x & SignBitOf(val):
        remainder = ~remainder;
    
    if w:
        quantity = quantity & 31;
    

    let result = ShiftInLeft(val, remainder, quantity);
    WriteRegister(0, d);

instruction FSR(d: u5, v: u5, q: u5, c: bool, x: bool, w: bool, r: u5):
    let val = ReadRegister(0, v);
    let quantity = ReadRegister(0, q);
    let remainder = ReadRegister(0, r);
    if x & SignBitOf(val):
        remainder = ~remainder;
    
    if w:
        quantity = quantity & 31;
    

    let result = ShiftInRight(val, remainder, quantity);
    WriteRegister(0, d);
```


### Branches

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| `JMP`    | `0x10` | `cccclllllooooooooooooooo` |
| `JMPR`   | `0x11` | `cccclllllrrrrr0000000000` |

Payload Bits Legend:
* `c`: Condition Code
* `l`: Link Register
* `o`: Destination Offset (Bits 2..17)
* `r`: Destination Register

Timing: `2+t+l+r` where:
* `t` is `1` if the branch is taken and `0` if it is not taken
* `l` is `1` if Link Register is non-zero and the branch is taken, and `0` otherwise
* `r` is `1` for `JMPR` and `0` for `JMP`

Behaviour: Jumps to the destination, if the condition is satisfied, saving the return address in `l` if taken:

* `JMP`: The offset is `IP + o * 4` where `o` is a signed offset. `IP` is the same as the return address and points to the beginning of the next instruction
* `JMPR`: The offset is read from `r`


```
instruction JMP(c: ConditionCode, l: u5, o: u15):
    let disp = SignExtend(o) << 2;
    let curr_ip = IP;
    if CheckCondition(flags, c):
        if l != 0:
            WriteRegister(0,l, curr_ip);
        IP = curr_ip + disp;

instruction JMPR(c: ConditionCode, l: u5, r: u5):
    let addr = ReadRegister(0,r);
    if addr & 3 != 0:
        Raise(EX[3]);
    let curr_ip = IP;
    if CheckCondition(flags, c):
        if l != 0:
            WriteRegister(0,l, curr_ip);
        IP = addr;
```

#### Condition Code

`JMP`, `JMPR`, and `MOV` all use a 4-bit condition code to encode the branch condition. This includes conditions for "Always" and "Never". 

```
enum ConditionCode is u4:
    Never = 0,
    Carry = 1,
    Zero = 2,
    Overflow = 3,
    CarryOrEqual = 4,
    SignedLess = 5,
    SignedLessOrEq = 6,
    Negative = 7,
    Positive = 8,
    SignedGreater = 9,
    SignedGreaterOrEq = 10,
    Above = 11,
    NotOverflow = 12,
    NotZero = 13,
    NotCarry = 14,
    Always = 15

function CheckCondition(flags: u32, cc: ConditionCode) is bool:
    switch (cc):
        case Never:
            return false;
        case Carry:
            return (flags & c) != 0;
        case Zero:
            return (flags & z) != 0;
        case Overflow:
            return (flags & v) != 0;
        case CarryOrEqual:
            return (flags & c|z) != 0;
        case SignedLess:
            return (((flags & v) != 0) == ((flags & n) != 0)) and (flags & z) == 0;
        case SignedLessOrEq:
            return (((flags & v) != 0) == ((flags & n) != 0)) or (flags & z) != 0;
        case Negative:
            return (flags & n) != 0;
        case Positive:
            return (flags & n) == 0;
        case SignedGreater:
            return not ((((flags & v) != 0) == ((flags & n) != 0)) or (flags & z) != 0);
        case SignedGreaterOrEq:
            return not ((((flags & v) != 0) == ((flags & n) != 0)) and (flags & z) == 0);
        case Above:
            return (flags & c|z) == 0;
        case NotOverflow:
            return (flags & v) == 0;
        case NotZero:
            return (flags & z) == 0;
        case NotCarry:
            return (flags & c) == 0;
        case Always:
            return true;
```


### I/O Transfers

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| `IN`     | `0x14` | `dddddpppppppp000000wwwww` |
| `OUT`    | `0x15` | `ssssspppppppp000000wwwww` |

Payload Bits Legend:

* s: Source Transfer Register
* d: Destination Transfer Register
* w: Transfer Bit Width
* p: Port Number

Timing: 7 + Port Delay

Behaviour: Shift `w` bits in an io transfer register in or out to an I/O Port

* `IN` : Shifts bits into the high bits of the transfer register
* `OUT`: Shifts bits out of the low bits of the transfer register

```
instruction IN(s: u5, p: u8, w: u5):
    let val = RotateRight(ReadBitsFromPort(p,w),w);
    let regval = ReadRegister(2, s);
    let resval, bitsout = ShiftRightInOut(regval, val, w);
    WriteRegister(2,s, resval);

instruction OUT(s: u5, p: u8, w: u5):
    let regval = ReadRegister(2, s);
    let resval, bitsout = ShiftRightInOut(regval, 0, w);
    WriteRegister(2,s, resval);
    WriteBitsToPort(p, w, bitsout);
```

### Flags Manipulation

| Mnemonic | Opcode   | Payload                    |
| -------- | -------- | -------------------------- |
|          | `0--7`   | `8---------------------31` |
| `LDFLAGS`| `0x18`   | `ddddd0000000000000000000` |
| `STFLAGS`| `0x19`   | `sssss0000000000000000000` |


Payload Bits Legend:
* s: Source Register
* d: Destination Register

Timing: 1

Behaviour:
* `LDFLAGS` loads the flags bits into the lower 5 bits of `d` (zero extended)
* `STFLAGS` stores the lower 5 bits of `s` into the flags bits

The Flags Bits are:

| `0---4` |
|---------|
| `cvnzp` |

* `c`: Carry
* `v`: Signed Overflow
* `n`: Negative
* `z`: Zero
* `p`: Parity

```
instruction LDFL(d: u5):
    let val = flags;
    WriteRegister(0,d, val);

instruction STFL(s: u5)
    let val = ReadRegister(0, s);
    flags = val & 0xF;
```

### Invoke Coprocessor Unit

| Mnemonic | Opcode   | Payload                    |
| -------- | -------- | -------------------------- |
|          | `0--7`   | `8---------------------31` |
| `CPIx`   | `0x20`+x | `ffffpppppppppppppppppppp` |
| `NCPIx`  | `0x28`+x | `ffffpppppppppppppppppppp` |
| `CPIxEF` | `0x30`+x | `ffffffpppppppppppppppppp` |
| `NCPIxEF`| `0x38`+x | `ffffffpppppppppppppppppp` |

(`x` is a value from `0` to `7`, representing the co-processor number to invoke, for example, `CPI0` has opcode 0x20 and `NCPI7` has opcode 0x2F)

Timing: 2 + N where:
* For `CPIx` and `CPIxEF`, `N` is the delay in cycles before the co-processor becomes ready to execute again
* For `NCPIx` and `NCPIxEF`, `N` is 0. 

Payload Bits Legend:
* `f`: Co-processor function
* `p`: Co-processor instruction payload

Behaviour: Executes the specified Coprocessor function with the specified payload
* `CPIx`/`CPIxEF`: Waits for the Co-processor to finish all operations, and raises the appropriate unit error if the Coprocessor reports it,
* `NCPIx`/`NCPIxEF`: Finishes immediately.
* `CPIx`/`NCPIx`: Allows specifying up to 16 functions with a 20-bit payload
* `CPIxEF`/`NCPIxEF`: Allows specifying up to 64 functions with a 18-bit payload (bottom 18-bits of the 20-bit payload)

```
instruction {CPI0, CPI1, CPI2, CPI3}(f: u4, p: u20):
    let coproc: u4;
    switch (instruction):
        case CPI0:
            coproc = 0;
        case CPI1:
            coproc = 1;
        case CPI2:
            coproc = 2;
        case CPI3:
            coproc = 3;
    if not IsCoprocessorEnabled(coproc):
        Raise(EX[3]);
    
    ExecuteCoprocessorInstruction(coproc, f, p);
    WaitOnCoprocessor(coproc);
    if PullCoprocessorException(coproc):
        Raise(Ex[4+coproc]);

instruction {CPI0EF, CPI1EF, CPI2EF, CPI3EF}(f: u6, p: u18):
    let coproc: u4;
    switch (instruction):
        case CPI0EF:
            coproc = 0;
        case CPI1EF:
            coproc = 1;
        case CPI2EF:
            coproc = 2;
        case CPI3EF:
            coproc = 3;
    if not IsCoprocessorEnabled(coproc):
        Raise(EX[3]);
    
    ExecuteCoprocessorInstruction(coproc, f, p);
    WaitOnCoprocessor(coproc);
    if PullCoprocessorException(coproc):
        Raise(Ex[4+coproc]);

instruction {NCPI0, NCPI1, NCPI2, NCPI3}(f: u4, p: u20):
    let coproc: u4;
    switch (instruction):
        case NCPI0:
            coproc = 0;
        case NCPI1:
            coproc = 1;
        case NCPI2:
            coproc = 2;
        case NCPI3:
            coproc = 3;
    if not IsCoprocessorEnabled(coproc):
        Raise(EX[3]);
    
    ExecuteCoprocessorInstruction(coproc, f, p);

instruction {NCPI0EF, NCPI1EF, NCPI2EF, NCPI3EF}(f: u6, p: u18):
    let coproc: u4;
    switch (instruction):
        case NCPI0EF:
            coproc = 0;
        case NCPI1EF:
            coproc = 1;
        case NCPI2EF:
            coproc = 2;
        case NCPI3EF:
            coproc = 3;
    if not IsCoprocessorEnabled(coproc):
        Raise(EX[3]);
    
    ExecuteCoprocessorInstruction(coproc, f, p);
```

### Halt/Stop CPU

| Mnemonic | Opcode   | Payload                    |
| -------- | -------- | -------------------------- |
|          | `0--7`   | `8---------------------31` |
| `HALT`   | `0x40`   | `000000000000000000000000` |
| `STOP`   | `0x41`   | `000000000000000000000000` |

Timing: 
* `HALT`: 1
* `STOP`: N/A

Behaviour: Places the CPU in a low-power state and stops executing
* `HALT`: Execution resumes after an NMI, IRQ (if enabled), Unit Error (if `sysctl.t=0`), or RESET.
* `STOP`: Execution resumes after a RESET only. No other interrupts are serviced.

```
instruction HALT() {
    SetStatus(2);
    WaitForInterrupt();
}
instruction STOP() {
    SetStatus(3);
    ShutdownCpu();
}
```

## Initial State

### Execution Address

The CPU begins executing from address 0xFF00.

### Register Contents

* GPR, I/O Transfer, and Coprocessor Registers are undefined
* System Registers are set according to the following:
   * `sysctl`: All bits 0
   * `copctl`: P bits set according to which Coprocessors are connected. Other bits are 0
   * `inttab`: 0
   * `intret`: Undefined

!{#copyright}