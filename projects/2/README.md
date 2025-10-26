# 2
Project 2 consists of:
- Expanding chasm-toolchain until we can turn on the board's red LED.
- Write the assembly program to actually do that.

## Work Completed
- Added the following instructions to `chasm assemble`:
  - MOVW
  - MOVT
  - LDR
  - ORRS
  - STR
  - ANDS
  - BEQ
  - (most only support the minimum needed for the given context, like 1 or 2 encodings,
    no "optional" parameters, no conditions, etc.)
- Continued reading a bunch of:
  - [TM4C123GH6PM Microcontroller data sheet](https://www.ti.com/lit/ds/symlink/tm4c123gh6pm.pdf?ts=1759278523305&ref_url=https%253A%252F%252Fwww.ti.com%252Fproduct%252FTM4C123GH6PM) - instructions and encodings mostly at this point.
  - [ARMv7-M Architecture Reference Manual](https://developer.arm.com/documentation/ddi0403/ee) - all board-specific info, like where GPIO ports are located, how to initialize them, etc.
- Flashed and tested on EK-TM4C123GXL, and we get a solid red LED 🎉 
  - Used `chasm assemble 2.cas` on commit `b601092026c8f0d11192b516a2ef74d5e528f92f`

The full assembly program is:
```
; vector table
x2000.8000         ; sp starting address
x0000.0009         ; reset handler address | 1 (thumb mode)

; Reset handler - toggle led

; GPIO Port F, (AHB) base - x4005.D000
; Data register pins at offset x01 (red), x02 (blue), x03 (green)

; 1. enable clock at RCGCGPIO - x400F.E608 
; set bit 5 to 1
MOVW R0 xE608
MOVT R0 x400F
LDR R1 R0
MOVS R2 x20             ; bits[5] = 1
ORRS R1 R2
STR R1 R0

; We need to use the AHB (vs. APB), AHB is the only option supported
; by the board but it's not set on reset
MOVW R0 xE06C 
MOVT R0 x400F
LDR R1 R0
MOVS R2 x20             ; bits[5] = 1
ORRS R1 R2
STR R1 R0

; wait for GPIO F to be ready for use via PRGPIO
MOVW R0 xEA08 
MOVT R0 x400F
MOVS R2 x20

@waitready
    LDR R1 R0
    ANDS R1 R2
    BEQ @waitready

; 2. Set direction of GPIO port F LED pins to output 
; GPIODIR register at offset x400, set pins[1..=3] to 1
MOVW R0 xD400
MOVT R0 x4005
LDR R1 R0
MOVS R2 x0E             ; bits[1..=3] = 1
ORRS R1 R2
STR R1 R0

; 3. Enable as digital I/O
; offset at x51C, pins 1, 2, 3 set to 1
MOVW R0 xD51C
MOVT R0 x4005
LDR R1 R0
MOVS R2 x0E             ; bits[1..=3] = 1
ORRS R1 R2
STR R1 R0

; Set port F pin 1 (red) to 1
; GPOI port F data register is at x4005.D000, and we need to write
; our mask for setting Pin 1 (0b00000010) at addr + (mask << 2):
; x4005.D000 + (0x02 << 2) = x4005.D008
MOVW R0 xD008
MOVT R0 x4005
MOVS R1 x02
STR R1 R0

; loop forever
@loop
    B @loop

```

## Next Steps
Partway through this I realized my assembler syntax doesn't exactly resemble what's in the ARM manual
(e.g. ARM would indicate a load like `LDR R1 [R0]`, but chasm syntax is `LDR R1 R0`). I don't know if
it's relevant in the practical sense, but I'll probably spend some time aligning at least that bit.

Also, the repetition of the sequence of load/store operations piqued my interest, so I might try to 
introduce function calls. If that goes smoothly, I still want to cycle through RGB colors, but to do that
I'll need to use one of the system timers, so that may happen, too.