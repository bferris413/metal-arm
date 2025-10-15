# 1
Project 1 consists of:
- Bootstrapping the `chasm` compiler driver
- Building enough of the assembler to compile an assembly version of Project 0

## Work Completed
- Created the [`chasm` binary application](https://github.com/bferris413/chasm-toolchain/tree/main/chasm)
    - The `chasm` binary is intended to be the driver for all build/test actions, similar to 
    `cargo` and (less so) `gcc`
    - Subcommands will enable access to different parts of the toolchain
        - `build` and `test` for application-level projects
        - `assemble` and `link` for manual assembly/linking
- Converted Project 0's binary to assembly
- Implemented enough of `chasm assemble` to recreate Project 0's binary
    - chasm commit `faf4d33d0a956e5c23c8be5308ee47148de6ef0b`
- Flashed and tested on EK-TM4C123GXL 

The full assembly program is:
```
; vector table
x2000.8000         ; sp starting address
x0000.0009         ; reset handler address | 1 (thumb mode)

; Reset handler - Initializes and continuously increments R0
MOVS R0 x01

@loop
    ADDS R0 x01
    B @loop
```

## Next Steps
Well for one, it's unlikely that I'll write any more handcrafted binaries. It was a good exercise
in Project 0 for getting a grip on encoding, but going forward I'll be adding whatever support is
needed to `chasm assemble` and let the machine handle it.

Implementation-wise, I'd like to interact more with the board itself, probably by cycling the RGB
spectrum on the built-in LED.