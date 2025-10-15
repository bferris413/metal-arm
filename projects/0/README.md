# 0
The purpose of Project 0 is to get my own, handwritten machine code running on my
practically unused [EK-TM4C123GXL](https://www.ti.com/tool/EK-TM4C123GXL) board.

## Goals
- Flash a minimal, handcrafted, working binary to the board
- No HALs or external software will run on the board apart from what I write
    - I get to use GDB for debugging
    - I get to use external software for flashing and whatever else is needed to get stuff _onto_ the board
- Maximize **learning**

## Work Completed
- Added [`hx`](https://github.com/bferris413/chasm-toolchain/tree/main/hx) utility for converting hex 
  text -> binary files, and vice versa
    - [`xxd`](https://linux.die.net/man/1/xxd) doesn't support comments, but I wanted to annotate
    whatever machine code I needed to write and didn't bother looking further
- Created minimal vector table and reset handler
- Initialized stack pointer
- Loop and increment a register forever
- Flashed with `openocd` and `telnet` (I think this could have just used `openocd` but I'm not sure)
- Single stepped and verified with `gdb`

Final program is in `program.hxs`.

## Commands Used

- text -> bin: 
```
hx program.hxs > program.bin
```
- start `openocd`: 
```
openocd -f board/ti_ek-tm4c123gxl.cfg
```
- flash binary:
```
telnet localhost 4444
flash write_image erase ./program.bin 0x00000000 bin
```
- step and inspect with gdb:
```
arm-none-eabi-gdb
(gdb) target extended-remote :3333
Remote debugging using :3333
# ...
(gdb) monitor reset halt
[tm4c123gh6pm.cpu] halted due to debug-request, current mode: Thread 
xPSR: 0x01000000 pc: 0x00000008 msp: 0x20008000
(gdb) stepi
halted: PC: 0x0000000a
(gdb) info registers
r0             0x1                 1
r1             0x0                 0
# ...
pc             0xa                 0xa
# ...

(gdb) stepi
halted: PC: 0x0000000c
(gdb) info registers
r0             0x2                 2
r1             0x0                 0
# ...
pc             0xc                 0xc
# ...
```