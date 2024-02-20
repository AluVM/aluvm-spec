# AluVM Specifications

Specification of AluVM (pronounced as "alluvium", [ɑlˈluːvium]) instruction set 
architecture and assembly language (AluVM Assembly or AluAsm).

AluVM is a pure functional register-based highly deterministic & exception-less
instruction set architecture (ISA) and virtual machine (VM) without random
memory access, capable of performing arithmetic operations, including operations 
on elliptic curves. The AluVM ISA can be extended by the environment
running the virtual machine (*host environment*), providing ability to load
data to the VM registers and support application-specific instructions (like
SIMD).

The main purpose for ALuVM is to be used in distributed systems whether 
robustness, platform-independent determinism are more important than the speed
of computation. The main area of AluVM applications (using appropriate ISA
extensions) is blockchain environments, consensus-critical computations,
multiparty computing (including deterministic machine learning),
client-side-validation, sandboxed Internet2 computing and genetic algorithms.


## Runtime environments and ISA extensions

The current list of AluVM runtime environments (providing corresponding ISA
extensions) include:
- AluRE (ALU Runtime Environment, multiparty computations, machine learning and 
  sandboxed computing for Internet2/Web4 applications running on client-side). 
  Supports following ISA Extensions:
  * ALURE (User I/O),
  * WEB4 (Networking I/O),
  * SIMD.
- RGB Core (client-side-validated smart contracts);
  * RGB: access to RGB contract data,
  * BTC: access to bitcoin blockchain data,
- LNP Core (lightning network).
  * BTC: access to bitcoin blockchain data,
  * LNC: access to lightning network channel data
  * WEB4 (Internet2 & Web4 I/O),
  * SIMD.
- REBICA (runtime environment for biologically-inspired computing architectures)
  providing ISA extension with the same name.

## Design

VM is purely functional, meaning that it treats all computation as the
evaluation of mathematical functions and does not have any side effects.

To make VM exception-less and robust the following design decisions were
implemented:
- no side effects of any instruction outside of the well defined VM state, and
  thus:
  * absence of stack (however it may be emulated using registers);
  * absence of random memory access;
- two distinct byte encodings of the full VM state always represent two distinct
  states (this requires special handling of `NaN` and `±0` values for float 
  registers);
- VM state is represented by a finite set of register values with predefined
  bit dimensionality;
- all registers have a special "uninitialized/no value" state;
- all registers at the start of the program are always initialized into;
  "no value" state, after which some registers are filled with input values;
- impossible arithmetic operations (like division on zero) instead of generating
  exception bring destination register into "no value" state;
- all instructions accessing the register data must check for "no value" state 
  in the source registers and if any of them has it, put the destination 
  register into "no value" (`None`) state;
- each instruction modifies only those non-control flow registers which are
  explicitly provided as the instruction arguments;
- the set of the control-flow register modification which may happen during
  operation are part of this formal specification;
- there are no byte sequences which can't be interpreted as a valid VM
  instruction, so arbitrary code jumps can't lead to an exception;
- any two distinct byte strings always represent strictly distinct programs.

### Registers

AluVM operates five sets of registers:
- Arithmetic integer registers, or `A`-registers (prefixed with `a` 
  in mnemonics);
- Arithmetic float registers, or `F`-registers (prefixed with `f` in mnemonics);
- General R-registers, used in cryptography (prefixed with `r` in mnemonics);
- Bytestring registers, or `S`-registers (prefixed with `s` in mnemonics);
- Control flow registers, named individually

There are 8 types of integer and float arithmetic and general registers, which 
differ in their bit size. For each type of these, as for the bytestring 
registers, there are 32 individual registers, indexed with 1..=32 numbers put in
square brackets in mnemonics.

**Arithmetic integer register types:**
- 8-bit (`a8`)
- 16-bit (`a16`)
- 32-bit (`a32`)
- 64-bit (`a64`)
- 128-bit (`a128`)
- 256-bit (`a256`)
- 512-bit (`a512`)
- 1024-bit (`a1024`)

**Arithmetic float register types:**
- 16-bit bfloat16 format used in machine learning (`f16b`)
- 16-bit IEEE-754 binary16 half-precision (`f16`)
- 32-bit IEEE-754 binary32 single-precision (`f32`)
- 64-bit IEEE-754 binary64 double-precision (`f64`)
- 80-bit IEEE-754 extended precision (`f80`)
- 128-bit IEEE-754 binary128 quadruple precision (`f128`)
- 256-bit IEEE-754 binary256 octuple precision (`f256`)
- 512-bit tapered floating point (`f512`), described below

**General register types:**
- 128-bit (`r128`)
- 160-bit (`r160`), most useful for RIPEMD-160 hashing
- 256-bit (`r256`)
- 512-bit (`r512`)
- 1024-bit (`r1024`), most useful for RSA key handling
- 2048-bit (`r2048`), most useful for RSA key handling
- 4098-bit (`r4096`), most useful for RSA key handling
- 8192-bit (`r8192`), most useful for RSA key handling

**Bytestring registers** is a single family of 256 registers each of which is
a combination of 16-bit string length (not accessible directly by the
instructions) and 2^16-byte long value.

**Control flow registers:**
- 1-bit status register `st0`
- 16-bit jump counting register `cy0`
- 16-bit call stack pointer register `cp0`
- Block of 2^16 call stack registers `cs0[0..2^16]`


### Arithmetics

Arithmetic operations has the following degrees of freedom:

1. **Number encoding**: the per-bit layout of the register mapping to the actual 
   arithmetical number value:
   - Unsigned integer: little-endian byte-encoded integer data occupying full
     length of the register
   - Signed integer: the same as unsigned integer with the highest bit (the
     highest bit of the last byte) reserved for the sign: `0` value indicates
     positive sign, `1` indicates negative sign: `(-1)^sign_bit`.
   - IEEE-754 floating point matching the bit length of the registry, with
     special extension for tapered floating point encoding in 512-bit floating 
     registers and machine-learning popular format of `bfloat16`.
2. **Exceptions**
   - Impossible arithmetic operation (0/0, Inf/inf) always sets the destination 
     register into `None` state (corresponding to `NaN` value of IEEE-754, i.e.
     all registers with `NaN` must report `None` state instead).
   - Operation resulting in the value which can't fit the bit dimensions under
     a used encoding, including representation of infinity for integer 
     encodings (`x/0 if x != 0`) results in:
     * for float underflows, subnormally encoded number,
     * for `x/0 if x != 0` on float numbers, `±Inf` float value,
     * for overflows in integer `checked` operations and floats (NB: float 
       operations are always `checked`), `None` value, setting `st0` to `false`,
     * for overflows in integer `wrapped` operations, modulo division on the
       maximum register value;

As a result, most of the arithmetic operations has to be provided with flags
specifying which of the encoding and exception handling should be used:
- **Integer encodings** has two flags:
  * one for signed/unsigned variant of the encoding
  * one for checked or wrapped variant of exception handling
- **Float encoding** has 4 variants of rounding, matching IEEE-754 options.

Thus, many arithmetic instructions have 8 variants, indicating the used encoding
(unsigned, signed integer or float) and operation behaviour in situation when
resulting value does not fit into the register (overflow or wrap for integers
and one of four rounding options for floats). These operations include
- addition
- subtraction
- multiplication

Division operation can't overflow in case of integers, however in this case it 
requires specification of the rounding algorithm. So the same bit used to 
specify overflow or wrapping behaviour in this case is used to detect rounding
algorithm: euclidean or non-euclidean rounding.

These operations are performed by *flagged arithmetic instructions*.

Each of the flagged arithmetic operations must set the value of `st0` register
to `false` under the following conditions:

- if overflow has happened for `checked` integer arithmetics;
- if a float operation resulted in overflow, setting destination register to
  `±Inf` (but not if the `±Inf` value resulted from a non-overflowing operation,
  like `0 + Inf = Inf`);
- if a float operation resulted in `NaN` value, i.e. for impossible arithmetic
  operations (see above);

In all other cases, including overflows with wrapped flag set, the flagged
operations must set `st0` to `true`.

Some number encodings allows additional arithmetic operations:
- Modulo division (division reminder) for unsigned integers
- Negation for floats and signed integers
- Absolute value / sign detection for singed integers and floats

These operations never overflow/underflow and modulo division on zero will 
always result in `None` value in a destination. Instructions performing these
operations are named *unflagged arithmetic instructions* and they do not modify
the value of `st0` register.

**Except all aforementioned rules, the implementation of the float arithmetics 
must strictly follow IEEE-754 standard.**


## Instruction set

### 1. Arithmetics

Flagged arithmetic instructions operate two source registers and one destination
register and are represented by a 40-bit value with the following structure:

Bits   | Bit count | Meaning     | Possible values
------:| ---------:| ----------- | -------------------
   0-7 |         8 | Operation   | add, sub, mul, div
  8-15 |         8 | Flags       | unsigned_checked, unsigned_wrapped, signed_checked, signed_wrapped, float_near, float_ceil, float_floor, float_to0
 16-23 |         8 | Source 1    | number of the first source register
 24-31 |         8 | Source 2    | number of the second source register
 32-39 |         8 | Destination | number of the destination register

For assembly representation the core arithmetic operations are written with
the flags provided as the operation suffix

```aluasm
          add:uc a8[0],a8[1],a16[2] ; will perform unchecked unsigned addition
                                    ; of the values in the first and second
                                    ; 8-bit integer registers a8[0] and a8[1]
                                    ; putting result of operation into a8[2]

          add:n f32[0],f32[1],f32[2]  ; will perform addition with rounding
                                      ; to the nearest value for the first and 
                                      ; second 32-bit float register (f32[0] 
                                      ; and f32[1]) putting the result of the 
                                      ; operation into f32[2]
```

This will have a bytcode representation of `0x6400000102` and `0x6404202122`, 
where first bytes (`0x64`) are the id of `add` instruction, the second byte is
the flags for number encoding (indicating which set of arithmetic registers is
used, integer or float) and operation variant and the last three bytes are
indexes of the registers.
