# AluVM Specifications

Specification of AluVM (virtual machine for Internet2 projects) and its assembly.

AluVM is a pure functional register-based highly deterministic & exceptionless
virtual machine without random memory access, capable of performing arithmetic
operations, including operations on elliptic curves.

## Design

VM is purely functional, meaning that it treats all computation as the
evaluation of mathematical functions and does not have any side-effects.

To make VM exceptionless and rubust the following design decisions were
implemented:
- absence of stack
- absence of random memory access
- all registers has a special "uninitialized/no value" state
- all registers at the start of the program are always initialized into
  "no value" state, after which some of registers are filled with input values
- impossible arithmetic operations (like division on zero) instead of generating
  exception bring destination register into "no value" state
- all instructions accessing register data must check for "no value" state in
  the source registers and if any of them has it put destination register into
  "no value" state
- there are no byte sequences which can't be interpreted as a valid VM
  instruction, so arbitrary code jumps can't lead to an exception

## Instructions

### 1. Arithmetics

Core arithmetic instructions operate two source registers and one destination
register and are represented by bit sequence of 16 bits which has the following
structure (in little-endian encoding):

Bit no | Bits | Meaning   | Possible values
------:| ----:| --------- | -------------------
   0-1 |    2 | Operation | add, sub, mul, div
   2-4 |    3 | Dimension | bits: 8, 16, 32, 64, 128, 256, 512, arbitrary precision (ap)
   5-7 |    3 | Type      | unsigned_checked, signed_checked, unsigned_unchecked, signed_unchecked, unsigned_ap, signed_ap, fiload, float_ap
  8-11 |    4 | Source 1  | number of the first source register
 12-15 |    4 | Source 2  | number of the second source register which is also destination

For assembly representation the core arithmetic operations are written with
prefix denoting the operation type, suffix denoting operation dimension

```aluasm
                uc:add8 [0] [1]         ; will perform unchecked unsigned addition
                                        ; of the values in the first and second
                                        ; 8-bit arithmetic register a8[0] and a8[1]
                                        ; putting result of operation into a8[1]
```

This will have a bytcode representation by `0x000201` instruction triplete,
where first byte represents class of the insturctions (core arithmetics).
