# **startup_stm32f446retx.s**

# **Flash**
Flash is the chip's non-volatile storage, so its contents are the same when powering and off. The NUCLEO-F446RE has 512 KB of Flash and 128 KB of SRAM

## **Memory Mapped**
Flash is in the address space `0x0800 0000`. CPU reads it with ordinary load instructions, no special execution. CPU runs code directly from Flash

## **Erase before Write**
To write Flash, you need to erase a whole sector (a block of memory cells) of Flash (sets every bit to 1), the program then flips bits from 1 to 0. For the F446, the sectors are:
4 16 KB sectors
1 64 KB sector
3 128 KB sectors

This is why the debugger needs a special flash programming sequence to upload the code to the MCU. Also why Flash should not be used as a variable.

## **Endurance**	
A sector survives ~10,000 erase cycles. It is for code and rarely changing data

## **Slower than CPU**
At 180 MHz, Flash cannot deliver a word in every cycle, so the MCU inserts wait states (CPU pauses a few cycles per read). The MCU chip hides this with the "ART accelerator". If the clock is set too high without enough wait states, the CPU reads nonsense and crashes.


# **The Vector Table**

The Vector table is a block of data that you can find in the startup_stm32f446retx.s file


    g_pfnVectors:
      .word  _estack
      .word  Reset_Handler

      .word  NMI_Handler
      .word  HardFault_Handler
      .word  MemManage_Handler
      .word  BusFault_Handler
      .word  UsageFault_Handler
      .word  0 #RESERVED


It's a an array of 32-bit numbers that sits on the start of Flash, it's like an array of memory addresses for the CPU to find. On reset, word 0 `_estack` is loaded to SP (stack pointer) and word 1 `Reset_Handler` is loaded to PC (program counter) 


The rest of the table is read later when the program actually runs. E.g. SysTick is fired, the CPU computes the memory address from the vector table `0x0800 0000 + 4 x (N+16)` and jumps to it


`.data`
The `.data` section holds global and static variables that have been initialized with non-zero values.

`.data` counts toward both Flash usage (initial values) and SRAM usage (active variables)


# **The Vector Table and .data**
Upon turning on or reset, the CPU does not run main() immediately. The CPU loads word 0 to SP and word 1 to PC. Then `startup_stm32` executes, copying `.data` values from Flash to SRAM.

Example Case
A global variable `int x =5;` needs both properties, it has to survive power off (stored in Flash), and has to be modifiable during runtime (must be in RAM too). `Reset_Handler` does this by copying the initial values from Flash to RAM before `main()` runs

Sources:
https://community.st.com/stm32-mcus-products-25/what-is-a-vector-table-and-what-information-it-contained-while-flashed-with-a-ota-user-application-34677


