# BARE METAL PROGRAMMING

A 32 bit register is like a row of switches that are 1 and 0. The LED on the Nucleo is on pin PA5, on for GPIOA, every output register (ODR) has a switch on every pin. Turning on the LED on PA5 means flipping the bit on the ODR register, but ONLY on that single bit, leave the other bits alone.

Registers are written with a prefix, like `0xABCDEFGH`

A`0b` prefix shows that it is a binary number (example: `0b11001010`

A `0x` prefix shows that it is a hexadecimal number 

IMPORTANT: THEY ARE READ FROM RIGHT TO LEFT AND ARE INDEX 0!!

## Prerequisites

Since we want to only switch one bit at a time WITHOUT messing up the others in the register. We can do it with these 3 operations


| Task | Code | What it does |
| --- | --- | --- |
| Set bit 5 to 1 | <code>x &#124;= (1 << 5);</code> | OR with a mask with only bit 5 on |
| Clear bit 5 to 0 | `x &= ~(1 << 5);` | AND with a mask with everything on EXCEPT bit 5 |
| Toggle bit 5 | `x ^= (1 << 5);` | XOR flips on that bit |

`1 << 5` means (start with one and shift it left 5 places). So the result would be `0b0010_0000`

### Worked Examples
Consider `x = 0b0000_1010`
Now, to set bit 5 to 1, we do `x |= (1 << 5);`. Remember that it is OR

          0000_1010   (x)
        | 0010_0000   (mask)
        = 0010_1010   (only bit 5 changed)

This uses OR, so the value of a bit in x and in the mask must either be 1, or both 1, resulting in 1.

Consider `x = 0b0011_0010`
To clear bit 5, use `x &= ~(1 << 5);`. Notice the `~`, this inverts the mask. 

          0011_0010   (x)
        & 1101_1111   (mask)
        = 0001_0010   (only bit 5 changed)
        
This uses AND, so the value of a bit in x and in the mask must both be 1, resulting in 1.

Consider `x = 0000_1010`
To toggle bit 5 from 0 to 1, we use the `x ^= (1 << 5 );`. This uses bitwise XOR.

          0000_1010   (x = 10)
        ^ 0010_0000   (mask)
        = 0010_1010   (bit 5 flipped from 0 to 1)

XOR flips bit 0 to 1, or 1 to 0.


## GPIO REGISTERS
GPIO can be configured by changing the bits in the registers
To configure a GPIO, use the mode register (GPIOx_MODER)
Since a register is 8 bytes or 32 bits, a GPIO can be configured like
### Approach A
        // Configuring PA5 as General Purpose Output Mode 
        RCC->AHB1ENR |= (1 << 0) // Enable GPIOA port
        GPIOA->MODER |= (1 << 10) 
        GPIOA->MODER &= ~(1 << 10)

For a GPIO pin, its bits are 2y:2y+1 in the register. The code sets bit 10 as 1 and 11 as 0 in the mode register, so PA5 is 01: General Purpose Output Mode.



However, there are other approaches to configuring registers
### Approach B
        GPIOA->MODER = (GPIOA->MODER & ~(3U << 10)) | (1U << 10);

Using Approach A is fine (and more explicit), however it can be improved for more reliability.

Approach A uses `1 << 11`, which is a signed `int`. While harmless, but in something like `1 << 31` (which is pin 15) it is undefined behavior in C. Use unsigned or unsigned long constants such as `1U` or `1UL`.

Approach A also does 2 seperate read-modify-writes. The pin enters a intermediate state briefly through betweeen the operation, this can be avoided by the one liner in Approach B.

The "intermediate state" is the moment between the two lines. For a pin just being configured out of reset is fine, but it matters when the pin is already configured.

For a few clock cycles, the pin is flaoting, then drives the level `ODR` holds.

This can cause a glitch on the pin output, through a false edge or a stale `ODR` value.

### Approach C
Or, you can use the CMSIS, to avoid counting the bits, which is honestly tedious. The F4 headers, handle them.
        // General purpose output mode (01) 
        GPIOA->MODER = (GPIOA->MODER & ~GPIO_MODER_MODER5) | GPIO_MODER_MODER5_0;
        
`GPIO_MODER_MODER5`is the mask of the whole 2-bit field (bits 11:10)

`GPIO_MODER_MODER5_0` is the low bit of the field (bit 10)

`GPIO_MODER_MODER5_1` is the high bit (bit 11)

This can also cover other modes

        // AF mode (10):
        GPIOA->MODER = (GPIOA->MODER & ~GPIO_MODER_MODER5) | GPIO_MODER_MODER5_1;

        // Analog (11): both bits
        GPIOA->MODER |= GPIO_MODER_MODER5;

        // Input (00): just clear
        GPIOA->MODER &= ~GPIO_MODER_MODER5;

RCC register values can be accessed in section 6.3.28 in the RM0390: Reference Manual

GPIO register values can be accessed in 7.4 in the RM0390: Reference Manual
