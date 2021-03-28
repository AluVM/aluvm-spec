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
