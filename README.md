# Noobs Guide For Writing Bootloader ARM Cortex M

## What is a Bootloader and why is it important?

A bootloader is a small program that runs immediately after a microcontroller (MCU) resets or powers up. After the boot ROM's execution, the bootloader is executed and will do the update when required and  then execute the end-user application.

The STM32F103C8T6 (the common “Blue Pill” MCU) comes from ST with a factory-programmed bootloader in ROM (system memory).

This bootloader is permanent and stored in system memory (0x1FFFF000 on F1 series), not in Flash, so you can’t erase it.

It supports programming via different interfaces, depending on the specific device:

1. USART1 (PA9 = TX, PA10 = RX) → always available

2. USB DFU (on some STM32F103 with USB peripheral, like the “C8T6” has), but not always enabled in the default ST bootloader

3. CAN (on some variants, not all)

Now what we want is not to change the inbuilt factory bootloader, we are going to create a bootloader and flash it into the flash memory and make sure that it jumps to the right address fo the firmware when a certain condition meets. The user application doesn't necessarily need to know the existence of the bootloader.

## How to write a bootloader?

The bootloader is usually placed at the chips flash base address, so that it will be executed by the CPU after reset. The following figure demonstrates a typical code placement of the user application and the bootloader.

<img width="799" height="939" alt="3913_Bootloader_Flash2" src="https://github.com/user-attachments/assets/eacc9aaf-a907-4e53-8bd4-51ea389b2785" />

So if we flash the bootloader firmware at the base address of the flash ram then the bootloader will be the first firmware to start getting executed.

A few steps are required to jump correctly to the specific address. We will see all of them below.

This is the main function that we are going to use to set the MSP and VTOR so that the jump to application is successful.
```c
void jump_to_firmware(uint32_t * appAddress) {

	HAL_RCC_DeInit();
	HAL_DeInit();

    SysTick->CTRL = 0;
    SysTick->LOAD = 0;
    SysTick->VAL  = 0;

    if( CONTROL_SPSEL_Msk & __get_CONTROL( ) )
    {  /* MSP is not active */
      __set_MSP( __get_PSP( ) ) ;
      __set_CONTROL( __get_CONTROL( ) & ~CONTROL_SPSEL_Msk ) ;
    }

    SCB->VTOR = appAddress;

    BootJumpASM(appAddress[0], appAddress[1]);

}

```

1. We need to send the firmware address as a pointer. On ARM Cortex-M (like STM32), when you jump to an application:

	The first word at the application’s base address holds the initial Main Stack Pointer (MSP). 

	The second word holds the Reset Handler (entry point) address.

	So we need a pointer so you can dereference and fetch these values.
	```c
	void jump_to_firmware(uint32_t * appAddress) {}
	```

	### What if you don’t use a pointer?

	Say you write:

	```c
	void jump_to_firmware(uint32_t appAddress) {}
	```

	Now appAddress is just a number (an integer), not a pointer.

	You cannot do appAddress[0] or appAddress[1].

	You won’t be able to read the vector table at that memory location.

	The function won’t know how to fetch the stack pointer and reset vector → so it cannot properly jump to firmware.

	At best, you’d just have a raw number, and if you try to cast and jump directly, it’ll crash because you didn’t set up the MSP.


2. ```c
	HAL_RCC_DeInit();
	HAL_DeInit();

    SysTick->CTRL = 0;
    SysTick->LOAD = 0;
    SysTick->VAL  = 0;
	```
	Now we need to reset the clock and make it all back to the normal reset state so that the next application will be intialized with the reset state of MCU then the user 		application can reinitialze the the clock according to their use.

	You can also reset all the interrputs requests in NVIC but as I am not using any interrupts so it doesn't matter.


3. ```c
    if( CONTROL_SPSEL_Msk & __get_CONTROL( ) )
    {  /* MSP is not active */
      __set_MSP( __get_PSP( ) ) ;
      __set_CONTROL( __get_CONTROL( ) & ~CONTROL_SPSEL_Msk ) ;
    }
	```

   Activate the MSP, if the core is found to currently run with the PSP. As the compiler might still use the stack, the PSP needs to be copied to the MSP before this.

4. ```c
	SCB->VTOR = appAddress;
   ```

   Load the vector table address of the user application into SCB->VTOR register. Make sure the address meets the alignment requirements.

5. ```c
	BootJumpASM( appAddress[ 0 ], appAddress[ 1 ] ) ;
   ```
   	The final part is to set the MSP to the value found in the user application vector table and then load the PC with the reset vector value of the user application. This 		can't be done in C, as it is always possible, that the compiler uses the current SP. But that would be gone after setting the new MSP. So, a call to a small assembler 			function is done.

6. The BootJumpASM( ) helper function can also be implemented with the compiler. However, writing assembler is something compiler-specific. So the implementation for the 		    BootJumpASM( ) function looks different for each compiler.

	```c
	__attribute__( ( naked, noreturn ) ) void BootJumpASM( uint32_t SP, uint32_t RH )
	{
  	__asm("MSR      MSP,r0");
 	 __asm("BX       r1");
	}
 	```

 	I'm using this as my stm32CubeIde has ARM COMPILER-6




