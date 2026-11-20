# SysTick

## What is a SysTick?
Every ARM-Cortex M4 (like every STM32 i think) has a SysTick, a 24-bit down counter. When we load a number to it, it counts down by 1 every clock cycle. When it hits 0, it reloads, and fires an interrupt. 
If it ticks once a milisecond, you can have a handler to add 1 to a counter variable each tick, and this is how you can make a non-blocking program, while still keeping track of time.

`HAL_Delay()` waits on this. `HAL_Init()` starts SysTick at 1 ms per tick, then the handler adds to a counter. So if we delete `HAL_Init()`, then `HAL_Delay` would freeze, since theres nothing adding to the counter.

The maximum value you can load onto SysTick is 16,777,215 which is 2^24-1, the maximum number for a 24-bit unsigned integer. 

## SysTick Registers

SysTick has 3 registers

| Register | Task |
|---|---|---|
|`SysTick->LOAD`| The number set to countdown |
|`SysTick->VAL`| The current count |
|`SysTick->CTRL`| Control switches |


`CTRL` can be configured on these bits (switch them to 1)
- Bit 0 (ENABLE): Starts the counter
- Bit 1 (TICKINT): Fire an interrupt when timer reaches 0
- Bit 2 (CLKSOURCE): 1 counts the processor clock, 0 means the processor clock divided by 8
- Bit 16 (COUNTFLAG): read 1 if counted to 0 since last read

## Example: A non-blocking timer with SysTick, blinking LED on PA5
There is no `HAL_Init()`, so the SysTick should be started manually. Also assume that I already configured the GPIO somewhere else in the code.

        SysTick->LOAD = (16000U - 1U);
        SysTick->VAL = 0; // reset the count so no errors and stuff
        SysTick->CTRL = ( (1<<2) | (1 << 1) | (1<<0));

        volatile uint32_t tick = 0;

        void  SysTick_Handler(void) {
           tick++;
        }

        int main() {
             uint32_t last = 0;
             while(1) {
                   if (tick - last >= 500) {
                       GPIOA->ODR ^= (1<<5);
                       last = tick;
                    }
            }
        }

### Some Notes on this
- Volatile on `tick`
`tick` must have `volatile` as external things can change its value while the compiler is busy. `volatile` forces the compiler to read its value on every single cycle.

On the other hand, `last` doesn't need volatile because nothing outside of `main()` changes it.
- Duplicate handler
CubeIDE already has a handler on `stm32f4xx_it.c`, it calls `HAL_IncTick()`
You can either change the handler on `stm32f4xx_it.c` by declaring a global tick variable on top, and changing `HAL_IncTick();` with `tick++;`

OR you can just delete it and use the one in `main.c`.
