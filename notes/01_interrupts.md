# Interrupts
Think of interrupts like a fire alarm. It follows this chain.

The pin is the physical button. On the NUCLEO-F446RE, the big blue button is on PC13, when pressed, it is read 0.

SYSCFG is the one that manages the external interrupt connection to the GPIOs

EXTI is the sensor. It has 16 input lines (0-15), one line for every pin number. So Line 4 can be fed by PA4, PB4, PC4. The `EXTICR` decides which port is connected to each line. EXTI watches its line for an edge (when the signal switches from HIGH to LOW or vice versa). It gives a pending flag when it detects one.

NVIC is the dispatcher. It decides if that interrupt is enabled and its priority

The CPU then looks up the handler's address in the vector table and runs the ISR.

## Some Definitions

### System Configuration Controller (SYSCFG)
The system configuration controller is mainly used to remap the memory accessible in the code area and to manage the external interrupt line connection to the GPIO.

It has registers like `SYSCFG_EXTICRx` which selects the source input for the EXTI interrupt. Each of these registers are 32 bits, however Bits[31:16] are reserved. Bits[15:0] are the ones used by the software to configure it.

`SYSCFG` has 4 `EXTICR` registers from `SYSCFG_EXTICR1` to `SYSCFG_EXTICR2`.

It uses 4 bits per EXTI line. So each register covers 4 lines

- `EXTICR1`: `EXTI0`-`EXTI3`
- `EXTICR2`: `EXTI4`-`EXTI7`
- `EXTICR3`: `EXTI8`-`EXTI11`
- `EXTICR4`: `EXTI12`-`EXTI15`

SYSCFG has its own RCC clock enable. It is on bit 14 in `RCC_APB2ENR` 

See 6.3.14 on the reference manual

See the reference manual RM0390 on 8.2.3


### External Interrupt/Event Controller (EXTI)
A hardware peripheral that detects changes in input lines. It can be configured
to detect the rising edge, the falling edge, or both.
```
        HIGH (1)            +-------------------+

                            |                   |
                            |                   |
                  Rising    |                   |    Falling
                   Edge     |                   |     Edge
                    (↑)     |                   |      (↓)

                            |                   |
        LOW (0)  -----------+                   +-----------
                           t1                  t2
```
It triggers an interrupt request sent to the NVIC to run an ISR, or generate a wakeup signal without a full ISR
 
### Nested Vectored Interrupt Controller (NVIC)
A hardware block that manages, prioritizes and dispatches interrupts and exceptions. It assigns priority levels (lower numbers are higher priority). It uses the vector table of ISRs so the processor can jump straight to it. Also trakcs the state of interrupts (inactive, pending, active).

### Interrupt Service Routine (ISR)
A function that runs automatically when an interrupt triggers, pauses the main program to respond to the event. ISRs should be short, like simply adding a number. Avoid long operations (like long delays or heavy math), so other interrupts are not blocked and so it can respond if another interrupt happens within that timeframe.

Example in the main program

        void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
        {
            if (GPIO_Pin == GPIO_PIN_0)
            {
                button_pressed = 1;
            }
        }


