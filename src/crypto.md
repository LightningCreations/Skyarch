# Cryptographic Instructions

## Working Registers

The first 16 registers of the Co-processor register set are working registers. Working Registers are available to all operations.
Registers 16-31 are scratch registers. These can be accessed by `MOV`, `CRMOV`, `CRLD`, and `CRST`, and by `CRADD`, `CRSUB`, `CROR`, `CRXOR`, `CRAND`, and `CRANDN`. 

## Functions

### Copy/Load

| Mnemonic  | Opcode | `pppppppppppppppppppp` |
|-----------|--------|------------------------|
|           | `0--3` | `4-----------------20` |
| `CRMOV`   | `0x01` | `dddddsssss0000000000` |
| `CRLD`    | `0x02` | `dddddsssssww00000000` |
| `CRST`    | `0x03` | `dddddsssssww00000000` |
| `CRLDB`   | `0x04` | `dddd0ssssscccc000000` |
| `CRSTB`   | `0x05` | `dddd0ssssscccc000000` |


### Comparison

| Mnemonic  | Opcode | `pppppppppppppppppppp` |
|-----------|--------|------------------------|
|           | `0--3` | `4-----------------20` |
| `CRCMP`   | `0x0F` | `dddddaaaa0bbbb0r00tt` |

Bits:
* `d`: Destination Register
* `a`: First working register to compare
* `b`: Second working register to compare
* `r`: Overwrite Condition Mask
* `t`: Test (0: Equals, 1: Similarity, 2: Difference, 3: Not Equal)

Behaviour: Tests `a` and `b` according to `t`, modifying `d` accordingly. If `r` is set, `d `is set to the result. If `r` is clear, `d` is set to the result anded with the current value of `d`. For test `0`, the result is all 1s if they are equal, and all 0s if they are different. For Test 1, sets the nth bit to 1 if and only if that bit is the same between a and b. For Test 2, sets the nth bit to 1 if and only if that bit is different between a and b. For test 3, the result is all 1s if the values are different.

This instruction is guaranteed to have consistent timing regardless of the input values of `d`, `a`, or `b`.

###


| Mnemonic  | Opcode   | `pppppppppppppppppppp` |
|-----------|----------|------------------------|
|           | `0----5` | `4---------------18` |
| `SHA32SIG`| `0o0020` | `ssssrrrrwwwwvvvv00` |
| `SHA32SUM`| `0o0021` | `ssssrrrraaaaeeee00` |
| `SHA32CHM`| `0o0022` | `ddddaaaabbbbcccc0m` |
| `SHA2IV`  | `0o0023` | `ddddiiii0000000l00` |
| `SHA2RC`  | `0o0024` | `ddddkkkkk000000l00` |