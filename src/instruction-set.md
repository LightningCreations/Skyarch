# Skyarch Instruction Set

## Gen Design


skyarch[general.word]
Word Size: 32 bit

skyarch[general.byte-order]
Memory Load/Store Order: Little Endian

skyarch[general.inst-size]
Instruction Size/Alignment: 4 bytes

Variable Sized Instructions: No

Common Flags/Condition Code Driven

## Instruction format

skyarch[instruction.format]
8-bit opcode in lowest byte, 24-bit payload in upper three bytes.

Bit order is notated as MSB-first, e.g. "31 down to 0". In memory, data and instructions should be stored in little endian, so the instruction will be in the first/lowest address, followed by the three payload bytes.

## Registers

### Register Maps

skyarch[register.maps]
There are 8 maps of registers:

- Map 0: General Purpose
- Map 1: System Configuration
- Map 2: I/O Transfer Registers
- Map 3: Information
- Map 4: Coprocessor Control
- Maps 8-15: Co-processor Registers.

skyarch[register.layout]
There are 32 registers of each type. Except for Map 0, not all registers may be defined.

### Map 0: General Purpose

skyarch[register.general]

skyarch[register.general.syntax]
Assembly syntax: `r`_`n`_.

skyarch[register.general.defs]
All Registers of the Map are defined. Certain Registers have special meaning:

skyarch[register.general.r0]

- `r0` is the zero-register. It reads as zero, and writes are ignored.

### Map 1: Interrupt Support

skyarch[register.interupt]

skyarch[register.interrupt.syntax]
Assembly syntax: `int`_`n`_ or alias.

Refer to the following table of defined registers. Some registers define a specific format

skyarch[register.interrupt.defs]

| Regno. | Aliases   | Description                      |
| ------ | --------- | -------------------------------- |
| 0      | `intctl`  | Interrupt Status Register        |
| 1      | `intret1` | Return for Priority 1 Interrupts |
| 2      | `intret2` | Return for Priority 2 Interrupts |
| 3      | `intret3` | Return for Priority 3 Interrupts |
| 4      | `ints0`   | Scratch Register                 |
| 5      | `ints1`   | Scratch Register                 |
| 6      | `ints2`   | Scratch Register                 |
| 7      | `ints3`   | Scratch Register                 |
| 8      | `intd0`   | Misc Config Register             |
| 9      | `intd1`   | Misc Config Register             |
| 10     | `intd2`   | Misc Config Register             |
| 11     | `intd3`   | Misc Config Register             |
| 31     | `inttab`  | Interrupt Table Pointer          |

skyarch[register.interrupt.validate]
Reading or Writing an undefined register causes `EX[2]`. Writing an invalid value to a defined register causes `EX[4]`

#### Interrupt Control (Map 1, Register 0)

skyarch[register.interrupt.intctl]

```
+31-----------------------------0+
|r00000000000000000000000000000mm|
+--------------------------------+
```

(All bits indicated as 0 must be written with 0)

skyarch[register.interrupt.intctl.desc]

| Bits | Name            | Description                                    |
| ---- | --------------- | ---------------------------------------------- |
| `r`  | Abort Triggered | Set to 1 when an Abort (Ex[0]) occurs.         |
| `m`  | Priority Mask   | Interrupts with priority value > m are blocked |

skyarch[register.interrupt.intctl.startup]
Both fields are set to `0` on startup.

##### Interrupt Priority

skyarch[register.interrupt.intctl.priority]
Interrupt Priority is used to ensure that overlapping Interrupts do not interfere.
There are 4 Priority levels, numbered in descending order of priority (0 is the highest priority, 3 is the lowest priority)

- Priority 0: Abort (Ex[0])
- Priority 1: Synchronous Exceptions (Ex[1], Ex[2], Ex[3], Ex[4])
- Priority 2: Asynchronous High Priority Event (Ex[7], EX[8-15])
- Priority 3: IRQs

skyarch[register.interrupt.intctl.mask]
An interrupt/trap is blocked when the priority level is greater than `m`. The behaviour depends on the kind of exception:

- Synchronous Exceptions (other than Abort) Reset the processor if `r = 1`, else they set `r = 1` and raise `Ex[0]`
- Asynchronous Events are discarded
- IRQs are buffered (up to an implementation-specific capacity until an `intret` occurs that sets `m` to be `3`) or are discarded.

#### Interrupt Return Registers

skyarch[register.interrupt.return]
Each priority of interrupt (other than priority 0) has a distinct return register, labeled `intret`_`n`_ where _n_ is the priority value, which corresponds to Register _n_ in map 1. Aborts are not recoverable, so no return register is provided.

skyarch[register.interrupt.return.format]

```
+31-----------------------------0+
|aaaaaaaaaaaaaaaaaaaaaaaaaaaaaamm|
+--------------------------------+
```

skyarch[register.interrupt.return.desc]

| Bits | Name          | Description                                     |
| ---- | ------------- | ----------------------------------------------- |
| `a`  | Address       | Contains the high 30 bits of the return address |
| `m`  | Priority Mask | Stores the priority mask before the interrupt   |

#### Interrupt Scratch/Config Registers

skyarch[register.interrupt.misc]

Registers 4 through 11 in Map 1 are unused, freely writable registers, labeled `ints`_`n`_ for registers 4+n and `intd`_`n`_ for registers 8+n.
The register `ints`_`n`_ is intended for use as a scratch register for interrupts with priority `n` (used during the interrupt procedure) and `intd`_`n`_ i intended for use as a data/configuration register for such interrupts (written by the program and read during each interrupt invocation).

#### Interrupt/Exception Table (Map 1, Register 31)

skyarch[register.interrupt.table]

skyarch[register.interrupt.table.format]

```
+31-----------------------------0+
|aaaaaaaaaaaaaaaaaaaaaaaaaaaaa000|
+--------------------------------+
```

(All bits indicated as 0 must be written with 0)

skyarch[register.interrupt.table.addr]
Bits `a` contain the 29 most significant bits of an 8-byte aligned address which points to the interrupt table. 512 bytes starting from this address refer to 64 8-byte entries of the interrupt table, which use the following format, in LSB-first order using little-endian byte encoding:

skyarch[register.interrupt.table.entry]
```
+31-----------------------------0+
|tttttttttttttttttttttttttttttt0p|
+63----------------------------32+
|00000000000000000000000000000000|
+--------------------------------+
```

The `t` bits are the 30 most significant bits of the address to transfer control to when the specified interrupt occurs.

The `p` bit must be set for all interrupt vectors that are present and valid to execute. If the CPU tries to execute a not-present interrupt vector, `EX[4]` is raised.

All bits indicated as `0` must be `0` when the vector is read from memory, or `EX[4]` is raised.

##### Interrupts

skyarch[register.interrupt.table.exceptions]
The first 32 interrupt entries are reserved for hardware exceptions, these interrupts are allocated as follows (and the `n`th entry in this list is designated elsewise as `EX[n]`):

- Entry `0`: Exception Handling Fault - an exception is raised when the `t` flag is set.
- Entry `1`: Bus Fault - accessing memory in a particular manner causes an error, or attempts to access memory that doesn't exist.
- Entry `2`: Invalid Instruction - An instruction that is executed is an unknown opcode, reserved, malformed, or invalid
- Entry `3`: Unaligned Branch Target - an indirect branch is unaligned.
- Entry `4`: Consistency - An invalid system control structure was loaded from memory, or an invalid value was written to a system register.
- Entry `7`: PIRQ - May be raised in response to a priority signal external to the processor that requires immediate resolution. This is handled like an IRQ, but uses priority 2 instead of priority 3.
- Entries `8`-`15`: Co-processor Unit `n` Error - The corresponding Coprocessor unit `n` signals an error after a `CPIn` instruction (`n` is Exception number - 4).
- Entries `5`, `6`, and `16`-`31` are reserved.

skyarch[register.interrupt.table.irqs]
The remaining entries (32-63), may be allocated as IRQ vectors.

#### Interrupt Checking

Interrupts are performed as follows:

skyarch[register.interrupt.check]

```
subroutine InterruptProcessor(iv: u6, pri: u2):
    let intctl: u32 = ReadRegister(1, 0);
    if (intctl & 3) < pri:
        if pri == 1:
            if (intctl & 0x80000000) != 0:
                ResetProcessor();
            else:
                WriteRegister(1, 0, 0x80000000);
                InterruptProcessor(0, 0);
                return;
        else:
            return;

    let addr: u32 = ReadRegister(1, 31) + (iv << 3);
    let retreg = IP | intctl & 3;
    if pri > 0:
        WriteRegister(1, pri, retreg);
    let iaddr = ReadMemory(addr);
    let rest = ReadMemory(addr + 4);
    CheckAndRaise(EX[2]);
    if rest != 0 or (iaddr & 2) != 0:
        Raise(EX[4]);
    if (iaddr & 1) == 0:
        Raise(EX[4]);
    let addr = iaddr & ~3;
    IP = addr;
    IL.valid = false;
    return;

subroutine Raise(EX[n]: Except):
    CancelCurrentInstruction();
    InterruptProcessor(n, 1);
    return;

subroutine CheckAndRaise(EX[n]: Except):
    if AsynchronousExceptionPending(EX[n]):
        Raise(EX[n]);
        ClearPendingException(EX[n]);
    return;

subroutine CheckAsync():
    if AsynchronousExceptionPending(EX[7]):
        InterruptProcessor(7, 2);
        ClearPendingException(EX[7]);
    let cpe = ReadRegister(4, 30);
    for n in 0..8:
        if (cpe & (1 << n)) != 0 and AsynchronousExceptionPending(EX[8+n]):
            InterruptProcessor(8+n, 2);
        ClearPendingException(EX[8+n]);

    let irq, hasirq = PullPendingIrq();
    if hasirq:
        InterruptProcessor(32+irq, 3);
```

skyarch[register.interrupt.check.pending-irq]
The processor behaves as if `CheckAsync()` is called after each instruction finishes writing to all memory and all registers.

### Map 2: I/O Transfer Registers

skyarch[register.io.syntax]
Assembly Syntax: `io`_`n`_.

skyarch[register.io]
Map 2 defines a sequence of input and output shift registers for transfering data to external peripherals.

skyarch[register.io.defs]
All Registers are defined and have no implied meaning.

### Map 3: Information Registers

skyarch[register.info]
The Information Registers Map is a Read Only Map that contains information about the CPU. All Registers Presently Read 0. Writes are illegal and raise `EX[2]`

### Map 4: Coprocessor Control

skyarch[register.coprocessor.control]
Each Co-processor has a 32-bit control word, which is defined by the Coprocessor.

skyarch[register.coprocessor.syntax]
Assembly Syntax: `cp`_`n`_ or alias.

skyarch[register.coprocessor.def]
Register N (N < 8) in Map 4 is defined if Co-processor N is present and enabled. Additionally, Register 30 is the coprocessor enable (`cpe`) register, and Register 31 is the coprocessor present (`cpp`) register.

skyarch[register.coprocessor.undefined]
Reads and writes to an undefined register or a register corresponding to a not-present or disabled coprocessor results in `EX[2]`. Writing to `cpp` results in `EX[2]`.

skyarch[register.coprocessor.serialize]
Where a `MOV` instruction writes to Register N (N < 8) in this map, the following guarantees are made about the ordering of surrounding instructions:

- The `MOV` instruction will not begin executing until any Coprocessor Invocation instruction that references coprocessor N and occurs before it have been fully written-back,
- Any Coprocessor Invocation instruction that references coprocessor N and occurs after it will not begin executing until the `MOV` instruction has been fully written-back

skyarch[register.coprocessor.validity]
A coprocessor may enforce arbitrary validity requirements on writes to its corresponding control register. Violations of these constraints generates a coprocessor error.

#### Map 4, Register 30: Coprocessor Enable

skyarch[register.coprocessor.enable]
The Coprocessor Enable register allows the system software to control what coprocessors are operating and usable from the CPU.

skyarch[register.coprocessor.enable]
```
+31-----------------------------0+
|000000000000000000000000EEEEEEEE|
+--------------------------------+
```

skyarch[register.coprocessor.enable.desc]
The bits marked `E` may be set by the program when the corresponding bit of Register 31 is set.
Setting the nth bit to 1 enables the coprocessor and setting it to 0 disables it.


skyarch[register.coprocessor.enable.undefined]
A bit may only be written with 1 if the corresponding bit in `cpp` is 1. If a write violates this rule, `EX[4]` is raised. Bits marked as `0` can never be written with a `1`.

skyarch[register.coprocessor.enable.serialize]
Where a `MOV` instruction writes to this register the following guarantees are made about the ordering of surrounding instructions:

- The `MOV` instruction will not begin executing until all off the following instructions that occur before it have been fully written-back:
  - Any `MOV` instruction that references a register in this map, other than an access to this register, `cpp`, or any undefined register,
  - Any `MOV` instruction that reads from a register in Maps 8-16 that does not refer to a not-present,
  - Any coprocessor invocation instruction that does not refer to a not-present coprocessor,
- Any of the following instructions that occur after the `MOV` instruction will not begin executing until the `MOV` instruction has been fully written-back:
  - Any `MOV` instruction that references a register in this map, other than this register, `cpp`, or any undefined register,
  - Any `MOV` instruction that accesses a register in Maps 8-16 that does not refer to a not-present coprocessor,
  - Any coprocessor invocation instruction that does not refer to a not-present coprocessor.

#### Map 4, Register 31: Coprocessor Present

skyarch[register.coprocessor.present]
The Coprocessor Enable register allows the system software to determine what coprocessors are connected to the CPU. This register is read-only and cannot be written from the CPU.

skyarch[register.coprocessor.present.format]

```
+31-----------------------------0+
|000000000000000000000000PPPPPPPP|
+--------------------------------+
```

skyarch[register.coprocessor.present.def]
The nth bit is set to 1 if the nth coprocessor is present. Note that it is not guaranteed that the set of enabled coprocessors is contiguous or that the set of enabled coprocessors begins at 0.

### Map 8-15: Co-processor Maps

skyarch[register.coprocessor-maps]

Co-processors connected to the system may expose up to 32 registers each. Registers in map `N` are only defined if the coprocessor co-processor (Co-processor N-8) is enabled.

skyarch[register.coprocessor-maps.undefined]
Writing to a register in map `8+N` with `cpe[bit N]` clear raises `Ex[2]`. Violating a validity constraint enforced by the coprocessor may raise an asynchronous coprocessor error.

### Reset State

skyarch[register.init]

On Reset (either hardware initiated, or initiated by an exception raised in an abort status), the CPU is initialized to the following state:

- It is executing (Status = 0)
- `IP` is initialized to 0xFF00.
- `cpe` is set to `0`.
- `ictl` is set to `m=0, a=0`
- `r0` is `0`.

skyarch[register.undefined]
All other registers, including `flags`, have undefined values. An undefined value means that any independent read from the register may return an unpredictable result. Undefined values are cleared when the register is written to

## Instructions

### Undefined Instructions

skarch[instr.und]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| UND      | `00000000` | -                          |
| UND      | `11111111` | -                          |

(The Payload bits are ignored by both instructions)

skarch[instr.und.except]

- `EX[2]` (decode): Unconditionally

skyarch[instr.und.behaviour]
Unconditionally raises Invalid Instruction errors

skyarch[instr.und.code]
```
instruction UND():
    Raise(EX[2])
```

### Pause

skyarch[instr.pause]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| PAUSE    | `00000001` | `000000000000000000kkkkkk` |

skyarch[instr.pause.payload]

- `k`: Total pause time

skyarch[instr.pause.behaviour]
Delays execution for `k` clock cycles, 0-63,

skyarch[instr.pause.serialize]
Any instruction that follows a `PAUSE` instruction will not begin executing until the `k` cycles have elapsed since the `PAUSE` instruction completed execution.

skyarch[instr.pause.code]
```
instruction PAUSE(k: u6):
    SuspendForClockTicks(k)
```

### Move

skyarch[instr.mov]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| MOV      | `00000010` | `00mmmmrl0sssss0ccccddddd` |

skyarch[instr.mov.payload]

- `m`: Map
- `r`: Direction
- `l`: Latency Control
- `s`: Source Register
- `c`: Condition Code (See Jump)
- `d`: Destination Register


skyarch[instr.mov.behaviour]
Copies data between general purpose registers and to/from general purpose registers into other registers.

skyarch[instr.mov.code]

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
        if md == 1 or m > 3:
            ValidateConfigurationRegisterValue(d, val);
        WriteRegister(md, d, val);
```

### LD/ST

skyarch[instr.ldst]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `ST`     | `00000011` | `rrmm00000000wwsssssddddd` |
| `LD`     | `00000100` | `rrmm00000000wwsssssddddd` |
| `LDI`    | `00000101` | `iiiiiiiiiiiiiiii00xddddd` |
| `LRA`    | `00000110` | `oooooooooooooooo00xddddd` |

skyarch[instr.ldst.payload]

- `r`: Ordering
- `m`: Update mode
- `w`: Width
- `s`: Source Register
- `x`: Set sign bits (Higher half)
- `i`: Immediate Value
- `o`: Offset
- `d`: Destination Register


skyarch[instr.ldst.behaviour]

- `ST`: Stores `1 << w` bytes from `d` to `[s]`
- `LD`: Loads `1 << w` bytes from `[s]` into `d`
- `LDI`: Loads an immediate `i` (sign or zero exteneded) into the first (h=0) 16 bits of `d`
- `LRA`: Loads the address `IP + o` (`o` is a signed immediate if `x` is true, and an unsigned immediate otherwise) into `d`. `IP` is taken from the beginning of the next instruction

skyarch[instr.ldst.well-ordered]
Every `ST`, `STIC`, or `STICW` instruction that modifies a given memory region will become visible on every core of the system in the same order.

skyarch[instr.ldst.code]

```

enum Ordering:
    Relaxed = 0,
    Acquire = 1,
    Release = 2,
    SeqCst = 3,

enum UpdateMode:
    None = 0,
    PostInc = 1,
    /* Illegal = 2 */,
    PreDec = 3,

instruction ST(s: u5, d: u5, w: u2 r: Ordering, m: UpdateMode):
    if r==1:
        Raise(EX[2])
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
        Raise(EX[1])

    SynchronizeMemoryAccordingToStore(r, addr);
    WriteAlignedMemoryTruncate(addr, val, width);
    CheckAndRaisePending(EX[1]);
    let new_addr: u32;
    if m == 1:
        new_addr = addr + width;
    if m != 0:
        WriteRegister(0, d, new_addr);


instruction LD(s: u5, d: u5,w: u2, p: u2):
    if r==2:
        Raise(EX[2])
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
    CheckAndRaisePending(EX[1]);
    SynchronizeMemoryAccordingToLoad(r, addr);
    WriteRegister(0,d,val);
    let new_addr: u32;
    if m == 1:
        new_addr = addr + width;
    if m != 0:
        WriteRegister(0, s, new_addr);

instruction LRA(d: u5, x: bool, i: u16):
    let val = SignExtendOrZeroExtend(i, x) + IP;
    WriteRegister(0,d,val);
```

### Immediate Arithmetic

skyarch[instr.add-imm]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `ADDI`   | `00001000` | `iiiiiiiiiiiiiiiihfxddddd` |


skyarch[instr.add-imm.payload]

- `i`: Immediate
- `h`: High half
- `f`: Enable Flags Modification
- `x`: Set sign bits (upper bits)
- `d`: Destination Register

skyarch[instr.add-imm.flags]
Sets `P`, `N`, and `Z` according to the result. Sets `V` and `C` according to the computation (signed overflow and carry)

skyarch[instr.add-imm.behaviour]
Adds a 16-bit zero or sign-extended immediate to `d`.

skyarch[instr.add-imm.code]
```
instruction ADDI(d: u5, x: bool, f: bool, h: bool, i: u16):
    let imm: u32;

    if h and x:
        Raise(Ex[2]);

    if h:
        imm = ZeroExtend(i, 32) << 16;
    else if x:
        imm = SignExtend(i, 32);
    else:
        imm = ZeroExtend(i, 32);
    let r = ReadRegister(0,d);

    let result, flags_val = r + imm;

    WriteRegister(0, d, result);
    if f:
        flags = flags_val;

```

### ALU Instructions

skyarch[instr.alu]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `ADD`    | `00001001` | `c0psssssfbbbbbaaaaaddddd` |
| `SUB`    | `00001010` | `c0psssssfbbbbbaaaaaddddd` |
| `AND`    | `00001011` | `jipsssssfbbbbbaaaaaddddd` |
| `OR`     | `00001100` | `jipsssssfbbbbbaaaaaddddd` |
| `XOR`    | `00001101` | `jipsssssfbbbbbaaaaaddddd` |

skyarch[instr.alu.payload]

- `c`: Carry in
- `j`: Invert op 2
- `i`: Invert op 1
- `p`: Shift Polarity
- `s`: Shift Quantity
- `f`: Enable Flags Modification
- `b`: Source Register 2
- `a`: Source Register 1
- `d`: Destination Register

skyarch[instr.alu.flags]

- `ADD`/`SUB`: Sets `P`, `N`, and `Z` according to the result. Sets `V` and `C` according to the computation (signed overflow and carry)
- `AND`/`OR`/`XOR`: Sets `P`, `N`, and `Z` according to the result. `V` and `C` are set to unspecified values.

skyarch[instr.alu.behaviour]
Computes the ALU corresponding ALU operation between the values in GPRs `b` and `a`, writing the result to GPR `d`. The operand corresponding to `p` is first shifted left by `s` (`p=1` shifts `a`, `p=0` shifts `b`)

- `ADD`: The result is `a + b`. If `c` is set, the carry flag is also added in.
- `SUB`: The result is `a - b`. If `c` is set, the carry flag is borrowed by the subtraction.
- `AND`: The result is `(i)a & (j)b`, `a` is inverted before being shifted if `i` is set, and `b` is inverted before being shifted if `j` is set.
- `OR`: The result is `(i)a | (j)b`, `a` is inverted before being shifted if `i` is set, and `b` is inverted before being shifted if `j` is set.
- `XOR`: The result is `(i)a ^ (j)b`, `a` is inverted before being shifted if `i` is set, and `b` is inverted before being shifted if `j` is set.

skyarch[instr.alu.code]

```
instruction {ADD, SUB}(a: u5, b: u5, d: u5, f: bool, s: u5, p: bool, c: bool):
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
            dest, flags_val = src1 + src2 + (flags.c & c);
            flags_mask = 0xF;
        case SUB:
            dest, flags_val = src1 - src2 + (~flags.c & c);
            flags_mask = 0xF;
    if f:
        flags = flags_val & flags_mask | nondeterministic() & ~flags_mask;

instruction {AND, OR, XOR}(a: u5, b: u5, d: u5, f: bool, s: u5, p: bool, i: bool, j: bool):
    let src1, src2: u32;
    if p:
        src1 = ReadRegister(0, a) << s;
        src2 = ReadRegister(0,b);
    else:
        src1 = ReadRegister(0, a);
        src2 = ReadRegister(0,b) << s;

    let val1, val2: u32;

    if i:
        val1 = ~src1;
    else:
        val1 = src1;

    if j:
        val2 = ~src2;
    else:
        val2 = src2;

    let dest: u32;
    let flags_val, flags_mask: u5;
    switch (instruction):
        case AND:
            dest = val1 & val2;
            flags_val = LogicCondition(dest);
            flags_mask = 0x3;
        case OR:
            dest = val1 | val2;
            flags_val = LogicCondition(dest);
            flags_mask = 0x3;
        case XOR:
            dest = val1 ^ val2;
            flags_val = LogicCondition(dest);
            flags_mask = 0x3;
    if f:
        flags = flags_val & flags_mask | nondeterministic() & ~flags_mask;
```

### Funnel Shifts

skyarch[instr.shift]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `FSL`    | `00001110` | `rrrrrw0xfqqqqqvvvvvddddd` |
| `FSR`    | `00001111` | `rrrrrw0xfqqqqqvvvvvddddd` |

skyarch[instr.shift.payload]

- `r`: Shift Remainder (Input value)
- `w`: Wrap Quantity
- `x`: Invert by Sign
- `f`: Enable Flags Modification
- `q`: Shift Quantity
- `v`: Input Value
- `d`: Destination Register

skyarch[instr.shift.flags]
Sets `P`, `Z`, and `N` according to the result. Sets `C` if any 1 bit was shifted out of `v`. Sets `V` if `q` is greater than 32 (regardless of `w`)

skyarch[instr.shift.behaviour]
Shifts `v` by `q` and places the value in `d`, filling the shifted in bits with bits taken from the corresponding high bits of `r`. `q` wraps at 32 if `w` is set. If `w` is clear, excess shift quanities shift `r` in fully first.

- `FSL`: `v` is shifted left by `q`
- `FSR`: `v` is shifted right by `q`

skyarch[instr.shift.code]
```
instruction FSL(d: u5, v: u5, q: u5, f: bool, x: bool, w: bool, r: u5):
    let val = ReadRegister(0, v);
    let quantity = ReadRegister(0, q);
    let remainder = ReadRegister(0, r);
    if x & SignBitOf(val):
        remainder = ~remainder;

    let overflow: u5;
    if quantity >= 32:
        overflow = 2;
    else:
        overflow = 0;

    if w:
        quantity = quantity & 31;


    let result, out = ShiftInLeft(val, remainder, quantity);
    WriteRegister(0, d);
    let carry: u5;

    if out != 0:
        carry = 1;
    else
        carry = 0;

    if quantity
    let flags_val = LogicCondition(result) | carry | overflow;
    if f:
        flags = flags_val;


instruction FSR(d: u5, v: u5, q: u5, c: bool, x: bool, w: bool, r: u5):
    let val = ReadRegister(0, v);
    let quantity = ReadRegister(0, q);
    let remainder = ReadRegister(0, r);
    if x & SignBitOf(val):
        remainder = ~remainder;

    let overflow: u5;
    if quantity >= 32:
        overflow = 2;
    else:
        overflow = 0;

    if w:
        quantity = quantity & 31;


    let result, out = ShiftInRight(val, remainder, quantity);
    WriteRegister(0, d);
    let carry: u5;

    if out != 0:
        carry = 1;
    else
        carry = 0;

    if quantity
    let flags_val = LogicCondition(result) | carry | overflow;
    if f:
        flags = flags_val;
```

### Branches

skyarch[instr.branch]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `JMP`    | `00010000` | `ooooooooooooooocccclllll` |
| `JMPR`   | `00010001` | `000000000rrrrr0cccclllll` |
| `IRET`   | `00010010` | `0000000000000000000000pp` |

skyarch[instr.branch.payload]

- `o`: Destination Offset (Bits 2..17)
- `r`: Destination Register
- `c`: Condition Code
- `l`: Link Register
- `p`: Target Interrupt Priority

skyarch[instr.branch.behaviour]
Jumps to the destination, if the condition is satisfied, saving the return address in `l` if taken:

- `JMP`: The offset is `IP + o * 4` where `o` is a signed offset. `IP` is the same as the return address and points to the beginning of the next instruction
- `JMPR`: The offset is read from `r`
- `IRET`: The offset it read from register `p` (p!=0) in Map 1. `intctl.m` is also loaded from `p.m` and `intctl.a` is cleared

skyarch[instr.branch.code]
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

instruction IRET(p: u2):
    if p == 0:
        Raise(Ex[2]);
    let reg = p as u5;
    let val = ReadRegister(1, reg);
    let addr = val & !3;
    IP = addr;
    IL.valid = false;
    WriteRegister(1, 0, val & 3);
```

#### Condition Code

skyarch[instr.branch.condition]

`JMP`, `JMPR`, and `MOV` all use a 4-bit condition code to encode the branch condition. This includes conditions for "Always" and "Never".

skyarch[instr.branch.condition.def]

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

skyarch[instr.io]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `IN`     | `00010100` | `wwwww000000ppppppppddddd` |
| `OUT`    | `00010101` | `wwwww000000ppppppppsssss` |

skyarch[instr.io.payload]

- w: Transfer Bit Width
- p: Port Number
- d: Destination Transfer Register
- s: Source Transfer Register

skyarch[instr.io.behaviour]

Shift `w` (in `1..=32`, mod 32) bits in an io transfer register in or out to an I/O Port. w=0 = 32

- `IN` : Shifts bits into the high bits of the transfer register
- `OUT`: Shifts bits out of the low bits of the transfer register

skyarch[instr.io.serialize]
Any `IN` or `OUT` instruction will not begin executing on a CPU until all `IN` or `OUT` instructions that occur before it have been fully written back. Additionally

- Any `OUT` instruction does not begin executing until all `LD` instructions that occur before it have been fully written back, and all `ST` instructions that occur before it have fully written to main memory
- Any `LD` or `ST` instruction does not begin executing until any `IN` instruction that occurs before it has been fully written back

```
instruction IN(s: u5, p: u8, w: u5):
    let val = RotateRight(ReadBitsFromPort(p,w),ExtendWidth(w));
    let regval = ReadRegister(2, s);
    let resval, bitsout = ShiftRightInOut(regval, val, ExtendWidth(w));
    WriteRegister(2,s, resval);

instruction OUT(s: u5, p: u8, w: u5):
    let regval = ReadRegister(2, s);
    let resval, bitsout = ShiftRightInOut(regval, 0, ExtendWidth(w));
    WriteRegister(2,s, resval);
    WriteBitsToPort(p, ExtendWidth(w), bitsout);

function ExtendWidth(w: u5) is u6:
    if w==0:
        return 0x20;
    else:
        w;
```

### Flags Manipulation

skyarch[instr.flags]

| Mnemonic  | Opcode     | Payload                    |
| --------- | ---------- | -------------------------- |
|           | `7------0` | `31---------------------8` |
| `LDFLAGS` | `00011000` | `00000000000000fffffddddd` |
| `STFLAGS` | `00011001` | `00000000000000fffffsssss` |
| `XVP`     | `00011010` | `000000000000000000000000` |

skyarch[instr.flags.payload]

- f: Flag modification mask
- d: Destination Register
- s: Source Register

skyarch[instr.flags.behaviour]

- `LDFLAGS` loads the flags bits into the lower 5 bits of `d` (zero extended)
- `STFLAGS` stores the lower 5 bits of `s` into the flags bits, overwriting only flags set to 1 in `f`
- `XVP` exchanges the v and p flags

skyarch[instr.flags.register]

The `flags` bits are, in order

| `4---0` |
| ------- |
| `pznvc` |

- `p`: Parity
- `z`: Zero
- `n`: Negative
- `v`: Signed Overflow
- `c`: Carry

skyarch[instr.flags.code]

```
instruction LDFL(d: u5, f: u5):
    let val = ZeroExtend(flags & f);
    WriteRegister(0,d, val);

instruction STFL(s: u5, f: u5)
    let val = ReadRegister(0, s);
    flags = (val & f) | (flags & ~f);

instruct XVP():
    let temp = flags.p;
    flags.p = flags.v;
    flags.v = temp;
```

### Exchange Register Contents

skyarch[instr.xchg]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `XCHG`   | `00011100` | `000000000bbbbblccccaaaaa` |

skyarch[instr.xchg.payload]

- `b`: Register 2
- `l`: Latency Control
- `c`: Condition Code (See Jump)
- `a`: Register 1

skyarch[instr.xchg.behaviour]
Exchanges GPR values `a` and `b`, if the condition check succeeds.

skyarch[instr.xchg.code]
```
instruction XCHG(a: u5, b: u5, l: bool, c: ConditionCode):
    let val1 = ReadRegister(0, a);
    let val2 = ReadRegister(0, b);
    if CheckCondtion(flags, c):
        WriteRegister(0, a, val2);
        WriteRegister(0, b, val1);
```

### Extend Register Contents

skyarch[instr.ext]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `EXT`    | `00011101` | `wwwww00000000xsssssddddd` |

skyarch[instr.ext.payload]

- `w`: Value width
- `x`: Extend Kind (sign/zero)
- `s`: Source
- `d`: Destination

skyarch[instr.ext.behaviour]
Masks only the lower `w` bits of a register, and extends it according to `x`

skyarch[instr.ext.code]

```
enum ExtKind:
    Sign = 0,
    Zero = 1

instruction EXT(dest: u5, src: u5, x: ExtKind, w: u5):
    let val = ReadRegister(0, src) & (1 << w)-1;
    let res: u32;
    switch(x):
        case Sign:
            res = SignExtend(val, w);
        case Zero:
            res = val;
    WriteRegister(0, dest, res);
```

### Swap Byte order

skyarch[instr.bswap]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `BSWAP`  | `00011110` | `000000000000000sssssddddd`|

skyarch[instr.bswap.payload]

- `s`: Source operand
- `d`: Destination Operand

skyarch[instr.bswap.behaviour]
Swaps the order of bytes from the source value.

skyarch[instr.bswap.code]
```
instruction BSWAP(s: u5, d: u5):
    let sval: u32 = ReadRegister(0, s);
    let rval: u32;
    rval[8:0] = sval[32:24];
    rval[16:8] = sval[24:16];
    rval[24:16] = sval[16:8];
    rval[32:24] = sval[8:0];
    WriteRegister(0, d);
```

### Random Bits

skyarch[instr.rand]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `RBGEN`  | `00011111` | `wwwww000000000eeeeeddddd` |

skyarch[instr.rand.payload]

- `w`: Poll width
- `e`: Status Destination
- `d`: Destination

skyarch[instr.rand.behaviour]
Polls a hardware random bit generator. If successful, writes `w` (in `1..=32`, mod 32) random bits to `d` and clears `flags.z`. If unsuccesful, writes `0` to `d` and sets `flags.z`. In all cases, the current status of the RBG is stored to `e`. (TODO: Write out status format). Note that `flags.z` is only set depending on success/failure. In particular, a successful poll that results in all `0s` (Approximately a 2^-(w+1) chance) will still clear `flags.z`.

skyarch[instr.rand.requirements]
The Random Bit Generator polled by the instruction shall have at least the following properties:

- Each complete output from the instruction is independant from other outputs on any core
- Each output from the instruction is distinct from all other outputs, with `2^-((w)/2)` probability of collision.
- If this instruction is used to generate at least 128 bits of randomness, which is then processed by a Cryptographic Hash Function, the resulting output shall have at least 64 bits of enthropy.

skyarch[instr.rand.code]

```
instruction RBGEN(d: u5, e: u5, w: u5):
    let valid, result, status = PollRand(ExtendWidth(w));
    WriteRegister(0, e, status);
    if valid:
        WriteRegister(0, d, result);
        flags.z = 0;
    else:
        WriteRegister(0, d, 0);
        flags.z = 1;
```

Status format:

```
+31-----------------------------0+
|r0000000000000sseeeeeeeeeeeeeeee|
+--------------------------------+
```

| Bit | Name               | Description                                       |
| --- | ------------------ | ------------------------------------------------- |
| `r` | Repeatable         | If set to 1, operation may be retried immediately |
| `s` | Status Code        | Status code (See Below)                           |
| `e` | Enthropy Available | Total ratio of enthropy available (\*2^16)        |

The following status code values are used

| Status Code | Name      | Description                            |
| ----------- | --------- | -------------------------------------- |
| 0           | `NORMAL`  | Normal status/spurious failure         |
| 1           | `UNAVAIL` | Required minimum enthropy unavailable  |
| 2           | `PAUSE`   | Generator Paused/Errored (Recoverable) |
| 3           | `FAULT`   | Unrecoverable Generator Error          |

The CPU shall ensure that it automatically attempts a reset of the Random Bit Generator after reporting a PAUSE status in finite time. In the case of a FAULT status, the Generator is only reset after a RESET.

### Invoke Coprocessor Unit

skyarch[instr.coprocessor]

| Mnemonic  | Opcode     | Payload                    |
| --------- | ---------- | -------------------------- |
|           | `7------0` | `31---------------------8` |
| `CPIx`    | `00100xxx` | `ppppppppppppppppppppffff` |
| `NCPIx`   | `00101xxx` | `ppppppppppppppppppppffff` |
| `CPIxEF`  | `00110xxx` | `ppppppppppppppppppffffff` |
| `NCPIxEF` | `00111xxx` | `ppppppppppppppppppffffff` |

(`x` is a value from `0` to `7`, representing the co-processor number to invoke, for example, `CPI0` has opcode 0x20 and `NCPI7` has opcode 0x2F)

skyarch[instr.coprocessor.payload]

- `p`: Co-processor instruction payload
- `f`: Co-processor function

skyarch[instr.coprocessor.behaviour]
Executes the specified Coprocessor function with the specified payload

- `CPIx`/`CPIxEF`: Waits for the Co-processor to finish all operations, and raises the appropriate unit error if the Coprocessor reports it,
- `NCPIx`/`NCPIxEF`: Finishes immediately.
- `CPIx`/`NCPIx`: Allows specifying up to 16 functions with a 20-bit payload
- `CPIxEF`/`NCPIxEF`: Allows specifying up to 64 functions with a 18-bit payload (bottom 18-bits of the 20-bit payload)

skyarch[instr.coprocessor.serialize]
If a `CPIx` or `CPIxEF` instruction begins execution on a core, the following guarantees are made:

* Any `CPIx`, `CPIxEF`, `NCPIx`, or `NCPIxEF` instruction (for the same `x`) that occurs after will not begin executing until the `CPIx` or `CPIxEF` and all `NCPIx` and `NCPIxEF` instructions that occur before the `CPIx` or `CPIxEF` have been fully written back, and
* Any `MOV` instruction that loads from a register in map `8+x` will not begin executing until the `CPIx` or `CPIxEF` and all `NCPIx` and `NCPIxEF` instructions that occur before the `CPIx` or `CPIxEF` have been fully written back.

skyarch[instr.coprocessor.code]

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
    CheckAndRaisePending(EX[8+coproc]);

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
    CheckAndRaisePending(EX[8+coproc]);

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

skyarch[instr.halt]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `HALT`   | `01000000` | `0000000000000000000000mm` |


skyarch[instr.halt.behaviour]
Places the CPU in a low-power state and stops executing.
The CPU responds to interrupts as though `ictl.m` was set temporarily to `m`. The CPU resumes execution after receiving an interrupt that is valid at priority `m` (if `m=0` then the CPU will never resume execution).

skyarch[instr.halt.serialize]
The `HALT` instruction will not begin executing until instructions that occur before it have been fully written back, and no instructions that occur after it will begin executing until the `HALT` instruction is fully written back.

```
instruction HALT(m: u2) {
    let saved_ictl: u32 = ReadRegister(1, 0);
    WriteRegister(1, 0, ZeroExtend(m) | (saved_ictl & (1 << 31)));

    if m != 0:
        SetStatus(2);
        WaitForInterrupt();
        WriteRegister(1, 0, saved_ictl);
    else:
        SetStatus(3);
        ShutdownCpu();
}
```

### Interlocked instructions

skyarch[instr.interlocked]

| Mnemonic | Opcode     | Payload                    |
| -------- | ---------- | -------------------------- |
|          | `7------0` | `31---------------------8` |
| `FENCE`  | `01001000` | `rr0000000000000000000000` |
| `STIC`   | `01001011` | `rr0000000000wwsssssddddd` |
| `LDIL`   | `01001100` | `rr0000000000wwsssssddddd` |
| `STICW`  | `01001101` | `rr0000000bbbbbsssssddddd` |
| `LDILW`  | `01001110` | `rr0000000bbbbbsssssddddd` |

skyarch[instr.interlocked.payload]

- `r`: Atomic Ordering
- `w`: Width
- `b`: Second source/destination register
- `s`: Source Register
- `d`: Destination Register

skyarch[instr.interlocked.exceptions]

- `EX[1]`: If `d` is unaligned
- `EX[1]`: If a bus fault occurs
- `EX[2]`: If `w = 3`
- `FENCE`: `EX[2]`: if `r = 0`
- `LDIL`, `LDILW`: `EX[2]`: if `r = 2`
- `STIC`, `STICW`: `EX[2]`: if `r = 1`

skyarch[instr.interlocked.flags]

- `STIC` and `STICW` set `z` if the validation check fails. In this case, no memory write or synchronization occurs.

skyarch[instr.interlocked.behaviour]

- `FENCE`: Serializes memory between processors according to `r`.
- `STIC`: Store `s` completing interlocked sequence on `d`. On success, the `z` flag is clear, and the store is guaranteed to be visible to any LDIL or LDILW instruction that is completed by a successful STIC instruction. It is also guaranteed that any ST instruction on any thread that was not observed by the LDIL instruction will not be overwritten by the STIC instruction.
- `LDIL`: Load from `s` into `d`, starting an interlocked sequence on `s` with `IL.width = w`
- `STICW`: Stores the 8-byte value in `b:s` into `d`, completing interlocked sequence on `d`. On success, the `z` flag is clear, and the store is guaranteed to be visible to any LDIL or LDILW instruction that is completed by a successful STIC instruction. It is also guaranteed that any ST instruction on any thread that was not observed by the LDILW instruction will not be overwritten by the STICW instruction.
- `LDILW`: Loads from `s` into an 8-byte value in `b:d`, starting an interlocked sequence on `s` with `IL.width = 3`

skyarch[instr.interlocked.state]
Each a core acts as though it stores the following additional state:

- Whether or not a valid interlocked transaction is occur (`IL.valid`)
- The address of the most recently begun interlocked transaction (`IL.addr`) if `IL.valid` is true
- The width of the most recently begun interlocked transaction (`IL.width`)

skyarch[instr.interlocked.invalidate]
The validity of an interlocked transaction on a core is reset when any of the following occurs:

- An `STIC` or `STICW` instruction completes execution, whether or not it was successful or failed
- An interrupt or exception occurs
- The `iret` instruction is issued
- If transaction is invalidated by a memory write on any core that violates the guarantees of any subsequent `STIC` or `STICW` instruction.

skyarch[instr.interlocked.constraint-stic]

An `STIC` instruction fails (sets `z = 1` and does not modify any memory) if:

- `IL.valid` is false
- `IL.addr` refers to a different address than the `STIC` instruction
- `IL.width` refers to a different width than the `STIC` instruction

skyarch[instr.interlocked.constraint-sticw]

An `STICW` instruction fails (sets `z = 1` and does not modify any memory) if:

- `IL.valid` is false
- `IL.addr` refers to a different address than the `STICW` instruction
- `IL.width` is not 3.

skyarch[instr.interlocked.acquire]
On a multicore system, any instruction that synchronizes memory according to the `Acquire` or `SeqCst` order guarantees that the instruction will not write back its result until, for each value loaded by any of the following instructions, all memory accesses visible to the corresponding store instruction will be observed by any instruction that occurs after that instruction:

- For `FENCE`: Any `LD`, `LDIL`, or `LDILW` instruction that preceeds it
- For `LD`, `LDIL`, or `LDILW`: That instruction.

skyarch[instr.interlocked.release]
On a multicore system, any instruction that synchronizes memory according to the `Release` or `SeqCst` order guarantees that the write will not be observed by any core until all memory operations that occur before it have completed and become visible to the instruction:

- For `FENCE`; Any `ST`, `STIC`, or `STICW` instruction that occurs after it
- For `ST`, `STIC`, or `STICW`: That insttruction.

skyarch[instr.interlocked.seqcst]
On a multicore system, any instruction that synchronizes memory according to the `SeqCst` order guarantees that all cores will observe the same order of memory effects caused by all such instructions.

skyarch[instr.interlocked.code]
```
instruction FENCE(r: Ordering):
    if r == Relaxed:
        Raise(EX[2]);
    SynchronizeMemoryAccordingToFence(r);

instruction STIC(d: u5, s: u5, w: u2, r: Ordering):
    if r == Acquire:
        Raise(EX[2]);
    if w == 3:
        Raise(EX[2]);
    let dest = ReadRegister(0, d);
    if dest & (1 << w)-1 != 0:
        Raise(EX[1]);

    let failed: u5;
    let value = ReadRegister(0, s);
    if not IL.valid or IL.addr != dest or IL.width != w:
        failed = 8;
    else:
        failed = TryInterlockedMemoryWrite(dest, w, value);
        CheckAndRaisePending(EX[1]);

    IL.valid = false;

    flags = failed | nondeterministic() & ~8;

instruction STICW(d: u5, s: u5, b: u5, r: Ordering):
    if r == Acquire:
        Raise(EX[2]);
    let dest = ReadRegister(0, d);
    if dest & 7 != 0:
        Raise(EX[1]);

    let failed: u5;
    let value = ReadRegister(0, s);
    let value_hi = ReadRegister(0, b);
    SynchronizeMemoryAccordingToWrites(r);
    if not IL.valid or IL.addr != dest or IL.width != 3:
        failed = 8;
    else:
        failed = TryInterlockedMemoryWriteWide(dest, value, value_hi);
        CheckAndRaisePending(EX[1]);

    IL.valid = false;

    flags = failed | nondeterministic() & ~8;

instruction LDIL(d: u5, s: u5, w: u2, r: Ordering):
    if r == Release:
        Raise(EX[2]);
    if w == 3:
        Raise(EX[2]);

    let src = ReadRegister(0, s);
    if src & (1 << w)-1 != 0:
        Raise(EX[1]);
    let value = InterlockedMemoryRead(src, w);
    CheckAndRaisePending(EX[1]);
    IL.valid = true;
    IL.addr = src;
    IL.width = w;
    WriteRegister(0, d, value);

instruction LDILW(d: u5, s: u5, b: u5, r: Ordering):
    if r == Release:
        Raise(EX[2]);
    if w == 3:
        Raise(EX[2]);

    let src = ReadRegister(0, s);
    if src & (1 << w)-1 != 0:
        Raise(EX[1]);
    let value, value_hi = InterlockedMemoryReadWide(src);
    CheckAndRaisePending(EX[1]);
    IL.valid = true;
    IL.addr = src;
    IL.width = 3;
    WriteRegister(0, d, value);
    WriteRegister(0, b, value_hi);
```
