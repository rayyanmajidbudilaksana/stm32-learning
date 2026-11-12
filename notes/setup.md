# Reset, Flash and the Vector Table

# **Flash**
Flash is the chip's non-volatile storage, so its contents are the same when powering on and off. The NUCLEO-F446RE has 512 KB of Flash and 128 KB of SRAM

## **Memory Mapped**
Flash is starts at the address space `0x0800 0000`. CPU reads it with ordinary load instructions, no special execution. CPU runs code directly from Flash

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

# Memory Map

Program memory, data memory, registers and I/O ports are organized within the same linear 4-Gbyte address space. The lowest numbered byte in a word is considered the word's least significant byte, and the highest numbered byte the most significant

The address space is divided to eight blocks, 512 Mbytes each.

Flash - 0x0800 0000 (part of FMC)
SRAM - 0x2000 0000
Peripherals - 0x4000 0000
Registers such as `GPIOx->ODR` are just addresses in this region.

See 2.2.2 in the RM0390 to see the addresses of everything

# Boot Config
The CPU will ALWAYS fetch the reset vector from address `0x0000 0000`. BOOT0/BOOT1 pins (latched on the 4th rising edge of SYSCLK) chooses which memory is aliased in the address.
BOOT0 = 0; Boots from main Flash. Normal case
BOOT0 = 1; BOOT1 = 0: Boots from system memory (built in bootloader, flash over UART/USB)
BOOT0 = 1; BOOT1 = 1: boots from embedded SRAM
See 2.3 in RM0390

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
      .word  0 /* RESERVED */

NOTE: `.word 0` is a reserved slot
It's a an array of 32-bit numbers that sits on the start of Flash, it's like an array of memory addresses for the CPU to find. On reset, word 0 `_estack` is loaded to SP (stack pointer) and word 1 `Reset_Handler` is loaded to PC (program counter). 


The rest of the table is read later when the program actually runs. The CPU computes the memory address from the vector table `0x0800 0000 + 4 x exception number` and jumps to it. Exceptions 0-15 are the core's own. IRQ N from a peripheral  is exception N+16. E.g. SysTick is at `0x0800 003c` so it is `0x0800 0000 + 4 × 15`

# Running the debugger and seeing what happens when the MCU resets
Launch the debugger, and run

    break Reset_Handler
    monitor reset
    continue

Using the command `info registers sp pc`, this will appear

    (gdb) info registers sp pc
    sp             0x20020000          0x20020000
    pc             0x08000ac8          0x08000ac8 <Reset_Handler>

`sp = 0x20020000` is at the top of SRAM, where the stack starts and grows downward, and PC is at `<Reset_Handler>`. No instructions have been executed yet


    (gdb) x/2wx 0x08000000
    0x8000000 <g_pfnVectors>:       0x20020000      0x08000ac9

Using `print SystemCoreClock` to show the system's clock at both states
gdb says 
    (gdb) print SystemCoreClock
    $2 = 16000000
This shows that the copy loop works, and the STM32F4 is running at the default 16 MHz internal RC oscillator.


# `.data` and `.bss`
The `.data` section holds global and static variables that have been initialized with non-zero values.

`.data` counts toward both Flash usage (initial values) and SRAM usage (active variables)

`.bss`
`.bss` holds global and static variables that have no initial value. Upon reset, the startup code clears the entire memory region to 0 during boot before `main()` runs. When compiling, variables that are not given an explicit  initial value or zero are put into the `.bss` section. .bss does not take any space in Flash, as it has no values to store, only a size.


# **The Vector Table and .data**
Upon turning on or reset, the CPU does not run main() immediately. The CPU loads word 0 to SP and word 1 to PC. Then `startup_stm32` executes, copying `.data` values from Flash to SRAM.

Example Case
A global variable `int x =5;` needs both properties, it has to survive power off (stored in Flash), and has to be modifiable during runtime (must be in RAM too). `Reset_Handler` does this by copying the initial values from Flash to RAM before `main()` runs

# Linker Symbols
Special variables generated that represent the memory addresses or sizes of specific sections within the memory map. Found in STM32F446RETX_FLASH

    /* Used by the startup to initialize data */
      _sidata = LOADADDR(.data);

      /* Initialized data sections into "RAM" Ram type memory */
      .data :
      {
        . = ALIGN(4);
        _sdata = .;        /* create a global symbol at data start */
        *(.data)           /* .data sections */
        *(.data*)          /* .data* sections */
        *(.RamFunc)        /* .RamFunc sections */
        *(.RamFunc*)       /* .RamFunc* sections */

        . = ALIGN(4);
        _edata = .;        /* define a global symbol at data end */

      } >RAM AT> FLASH
      
      
`.data`'s initial values are stored in Flash at `_sidata`, and copied to RAM between `_sdata` and `_edata`

    .bss :
      {
        /* This is used by the startup in order to initialize the .bss section */
        _sbss = .;         /* define a global symbol at bss start */
        __bss_start__ = _sbss;
        *(.bss)
        *(.bss*)
        *(COMMON)

        . = ALIGN(4);
        _ebss = .;         /* define a global symbol at bss end */
        __bss_end__ = _ebss;
      } >RAM
Here is the linker symbol for bss, it defines it between `_sbss` and `_ebss`. It shows how it collects zero-initiliazed global variables from the compiled program and place them into SRAM.

## Running at GDB 
    print (void *)&_sdata
    $2 = (void *) 0x20000000 <SystemCoreClock>
    (gdb) print (void *)&_edata
    $3 = (void *) 0x20000070 <completed>
    (gdb) print (void *)&_sidata
    $4 = (void *) 0x800361c
    (gdb) 

`_sidata` is `0x0800361c` as gdb printed
Looking at `_sdata` and `_edata`, we can see that the size of `.data` is `0x70`, so it is 112 bytes from `_sdata` to `_edata`

    print (void *)&_sbss
    $2 = (void *) 0x20000070 <completed>
    (gdb) print (void *)&_ebss
    $3 = (void *) 0x20000228

`_sbss` is at `0x20000070`, so it is right after `_edata`
Size: `0x228` - `0x70` = `0x1B8` = 440 bytes
Takes no Flash space, just a size.
    
    
# SystemInit
A standard function. Runs right after reset and before jumping to `main`.

      /* Configure the Vector Table location -------------------------------------*/
    #if defined(USER_VECT_TAB_ADDRESS)
      SCB->VTOR = VECT_TAB_BASE_ADDRESS | VECT_TAB_OFFSET; /* Vector Table Relocation in Internal SRAM */
    #endif /* USER_VECT_TAB_ADDRESS */
    
Here the SystemInit function finds sets the vector table location upon initialization.


# What happens between reset and `main()`. 
CPU loads SP and PC from the vector table. `Reset_Handler` copies `.data` from Flash to RAM, zeroes `.bss`*, calls `SystemInit`, runs the static initializers and branches to main. 

SRAM is uninitialized at power-up, something like x = 5 cannot exist on its own. Which is why RAM needs this process


Sources:
https://community.st.com/stm32-mcus-products-25/what-is-a-vector-table-and-what-information-it-contained-while-flashed-with-a-ota-user-application-34677

RM0390:
https://www.st.com/resource/en/reference_manual/rm0390-stm32f446xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf
