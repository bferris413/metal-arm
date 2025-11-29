# 3
Project 3 consists of:
- Responding to button presses, where each press toggles the red LED.

## Work Completed
- Updated syntax to use register dereferences like `LDR R1 [R0]` (added `[]`)
- Added support for label references like `&label`, which evaluates to the address of `@label`
- Extended branching to support not-yet-defined labels (like `B &label ... @label`)
- Added support for "pseudo" instructions. These are assembler-interpreted operations
  that simplify some redundant or common behavior that would be onerous to do manually
  - Examples are:
    - `pad-with-to!(<byte>, <target_address>)` - fill `[current_address, target_address)` with `byte`
    - `thumb-addr!(<address>)` - convert `address` to a thumb address (`address | 1`)
- Added the following instructions to `chasm assemble`:
  - EORS
  - BX
- Flashed and tested on EK-TM4C123GXL, and we can toggle the red LED with Button #2 on the board. 
  - Used `chasm assemble 3.cas` on chasm commit `8a2ef01e3f5ff169bd933a4e87d7742312c69899`
- Read a ton of the [board reference manual](https://www.ti.com/lit/ds/symlink/tm4c123gh6pm.pdf?ts=1759278523305&ref_url=https%253A%252F%252Fwww.ti.com%252Fproduct%252FTM4C123GH6PM) and the [Arm v7 Architecture Reference](https://developer.arm.com/documentation/ddi0403/ee). New information included:
  - Extending the vector table to include a GPIO port event handler (GPIO F in this case, for the button presses)
  - Setting pins on GPIO to be used as input
  - Enabling different activation triggers (pull-up, edge-detect, level-detect, etc.)
  - Enabling the nested vector interrupt controller (NVIC) for GPIO (port F, specifically)
  - Actually enabling interrupts on GPIO F
  - Implementing the event handler
    - The final logic works but doesn't seem idiomatic, specificaly around dealing with race 
      conditions and my toggle behavior. When pressing the button, sometimes I get a flicker/burst
      on the LED. I rubber-ducked the symptoms off ChatGPT and showed my assembly, and it suggested
      changing the way I clear the interrupt, turning off interrupts while in the handler, and adding
      a short busy-loop as a debounce. I never tested those ideas out since I figured I implemented enough for the current iteration and the light _does_ toggle =). 

The full assembly program is:
```
; vector table -------------------------------
x2000.8000                      ; sp starting address
thumb-addr!(&reset)             ; reset handler address | 1 (thumb mode)
pad-with-to!(x00, x0000.00B8)   ; skip to 0xB8, GPIO F interrupt handler address
thumb-addr!(&button-press)      ; GPIO F button press event handler | 1 (thumb mode)
pad-with-to!(x00, x0000.026C)   ; pad out the rest of the vtable
; --------------------------------------------

@reset
    ; enable clock at RCGCGPIO - x400F.E608 
    ; set bit 5 to 1
    MOVW R0 xE608
    MOVT R0 x400F
    LDR R1 [R0]
    MOVS R2 x20             ; bits[5] = 1
    ORRS R1 R2
    STR R1 [R0]

    ; We need to use the AHB (vs. APB), AHB is the only option supported
    ; by the board but it's not set on reset
    MOVW R0 xE06C 
    MOVT R0 x400F
    LDR R1 [R0]
    MOVS R2 x20             ; bits[5] = 1
    ORRS R1 R2
    STR R1 [R0]

    ; wait for GPIO F to be ready for use via PRGPIO
    MOVW R0 xEA08 
    MOVT R0 x400F
    MOVS R2 x20

    @waitready
        LDR R1 [R0]
        ANDS R1 R2
        BEQ &waitready


    ; Set direction of GPIO port F pins: LED -> output, button -> input 
    ; GPIODIR register at offset x400, set pins[1..=3] to 1
    ; bits[0] and bits[4] are implicitly 0 (inputs)
    MOVW R0 xD400
    MOVT R0 x4005
    LDR R1 [R0]
    MOVS R2 x0E             ; bits[1..=3] = 1, bits[0,4] = 0
    ORRS R1 R2
    STR R1 [R0]

    ; Enable pullup on GPIO F[4]
    ; GPIOPUR register at offset x510, set pins[4] to 1
    MOVW R0 xD510
    MOVT R0 x4005
    LDR R1 [R0]
    MOVS R2 x10             ; bits[4] = 1
    ORRS R1 R2
    STR R1 [R0]

    ; Enable as digital I/O
    ; offset at x51C, pins 1, 2, 3, 4 set to 1
    MOVW R0 xD51C
    MOVT R0 x4005
    LDR R1 [R0]
    MOVS R2 x1E             ; bits[1..=4] = 1
    ORRS R1 R2
    STR R1 [R0]

    ; Set GPIOIS (cleared is edge-detect, set is level-detect)
    ; Since it's cleared on reset, do nothing

    ; Set GPIOIEV (cleared is falling-edge, set is rising-edge)
    ; Since it's cleared on reset, do nothing

    ; Enable NVIC interrupt handling
    ; GPIO F is interrupt 30, corresponding to bits[30] at xE000.E000,
    ; offset x100
    MOVW R0 xE100
    MOVT R0 xE000
    LDR R1 [R0]
    MOVS R2 x00
    MOVT R2 x4000           ; bits[30] = 1
    ORRS R1 R2
    STR R1 [R0]

    ; Enable interrupt on GPIOF[4]
    ; GPIOIM register at offset x410, set pins[4] to 1
    MOVW R0 xD410
    MOVT R0 x4005
    LDR R1 [R0]
    MOVS R2 x10             ; bits[4] = 1
    ORRS R1 R2
    STR R1 [R0]

    @wait-for-event
        B &wait-for-event


; Toggle red LED, GPIO F pin 1
@button-press
    ; clear interrupt
    ; offset x41C, port F at x4005.D000, pins[4]
    MOVW R0 xD41C
    MOVT R0 x4005
    MOVS R5 x10
    LDR R1 [R0]     ; Lack of atomics is now glaring, need to figure that out
    ORRS R1 R5
    STR R1 [R0]     

    ; we'll read GPIO F data at offset x03FC, which is the masked addr
    ; for mask xFF. Also, realized this is a bit unsafe if we get interrupted
    ; since we're writing all bits. Baby steps
    MOVW R0 xD3FC
    MOVT R0 x4005
    LDR R1 [R0]
    MOVS R2 x02    ; bits[1] (pin 1)
    EORS R1 R2
    STR R1 [R0]

    ; return
    ; load EXC_RETURN (pre-loaded into LR) into PC
    BX LR    

```

## Next Steps
I mentioned in #2 that repeating the same setup code is kind of onerous, and I
felt the same in this project. In the next project I'd like to add chasm support
for function calls and some kind of `include` statement. This implies multiple
source files, and would be the first foray into a primitive linking step.

Depending on how that goes I'll either wrap it up _or_ try to cover an embedded
feature. Not having any debug printing has been kind of painful, so maybe
investigating and implementing that is next.