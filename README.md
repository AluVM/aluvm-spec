# AluVM Specifications

Specification of AluVM (virtual machine for Internet2 projects) and its assembly.

AluVM is a pure functional register-based highly deterministic & exception=less
virtual machine without random memory access, capable of performing arithmetic
operations, including operations on elliptic curves.

The main purpose for ALuVM is to be used in distributed systems whether 
robustness, platform-independent determinism are more important than the speed
of computation. Such system include blockchain environments, consensus-critical
computations, client-side-validation etc.

## Design

VM is purely functional, meaning that it treats all computation as the
evaluation of mathematical functions and does not have any side effects.

To make VM exception-less and robust the following design decisions were
implemented:
- absence of stack (however it may be emulated using registers)
- absence of random memory access
- all registers have a special "uninitialized/no value" state
- all registers at the start of the program are always initialized into
  "no value" state, after which some registers are filled with input values
- impossible arithmetic operations (like division on zero) instead of generating
  exception bring destination register into "no value" state
- all instructions accessing the register data must check for "no value" state 
  in the source registers and if any of them has it, put the destination 
  register into "no value" (`None`) state
- there are no byte sequences which can't be interpreted as a valid VM
  instruction, so arbitrary code jumps can't lead to an exception

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
     special extensions tampered floating point encoding in 512, 1024 and 
     2048-bit floating registers.
2. **Exceptions**
   - Impossible arithmetic operation (0/0, Inf/inf) always sets the destination 
     register into `None` state (corresponding to `NaN` value of IEEE-754, i.e.
     all registers with `NaN` must report `None` state instead).
   - Operation resulting in the value which can't fit the bit dimensions under
     a used encoding, including representation of infinity for integer 
     encodings (`x/0 if x != 0`) results in:
     * for float underflows, subnormally encoded number,
     * for `x/0 if x != 0` on float numbers, `±Inf` float value,
     * for overflows in integer `checked` operations and floats, `None` value,
     * for overflows in integer `wrapped` operations, modulo division on the
       maximum register value

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

Some of the used number encodings allows additional arithmetic operations:
- Modulo division (division reminder) for unsigned integers
- Negation for floats and signed integers
- Absolute value for singed integers and floats
- Detection of the sign for signed integers and floats

These operations never overflow/underflow and modulo division on zero will 
always result in `None` value in a destination. Instructions performing these
operations are named *unflagged arithmetic instructions*.

**In all the rest, the implementation of the float arithmetics must strictly 
follow IEEE-754 standard.**


## Instruction set

### 1. Arithmetics

Flagged arithmetic instructions operate two source registers and one destination
register and are represented by a 40-bit value with the following structure:

Bits   | Bit coint | Meaning     | Possible values
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
                                    ; 8-bit integer registers i8[0] and i8[1]
                                    ; putting result of operation into i8[2]

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
