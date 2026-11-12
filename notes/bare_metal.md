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
| Set bit 5 to 1 | `x |= (1 << 5);` | OR with a mask with only bit 5 on |
| Clear bit 5 to 0 | `x &= ~(1 << 5);` | AND with a mask with everything on EXCEPT bit 5 |
| Toggle bit 5 | `x ^= (1 << 5 );` | XOR flips on that bit | 

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

