# 4
Project 4 is a bit smaller in implementaion size due to amount of reading I needed to do. Nevertheless, it consists of:
- Adding instructions to support actual function calls,
- Reading (in-progress) the [ARM Procedure Call Standard](https://github.com/ARM-software/abi-aa/blob/main/aapcs32/aapcs32.rst#preamble).
- Started using [Compiler Explorer](https://godbolt.org/) for code generation comparisons
- Getting an emulator setup going via QEMU for toolchain-related tasks that aren't dependent on real hardware.

## Work Completed
- Added support for BL/BX instructions to enable function calls
  and returns to/from a given function (which is just a label):
  ```
  BL &function
  ; [...]

  @function
    ; [...]
    BX LR
  ```
- Added `PUSH {regs...}` and `POP {regs...}` for stack management
- Added `define!(<identifier>, <hex_literal>)` pseudo-instruction for definitions
  visible during assembly (but the `define` itself doesn't appear in the binary):
  ```
  define!(some-value, x11)
  ; [...]
  MOVS R0 $some-value
  ```
- Added `mov!(<register>, <hex_literal>)` pseudo-instruction for a little boilerplate
  reduction when generating plain MOVs


## Next Steps
I _think_ this is enough foundation for creating a primitive-but-useful module system that
isn't strictly "text-replace" based. That, combined with a small linker, will let me start
writing actually reusable code. Onward!