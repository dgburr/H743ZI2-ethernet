# What is this?

This repository contains a functional Ethernet driver for the NUCLEO H743ZI2 development board from ST.  It is based on [V2 of the improved driver provided by Alister Fisher on the ST Community Forums](https://community.st.com/s/question/0D50X0000C6eNNSSQ2/bug-fixes-stm32h7-ethernet).

# Why is it needed?

This driver is needed because the ethernet driver provided by ST in STM32CubeIDE/STM32CubeMX is extremely buggy and unstable.

# Setup instructions

## 1. Add new files to your project

* Copy the modified `stm32h7xx_hal_eth.h` file into the same path as `stm32h7xx_hal_conf.h` (`Core/Inc/` by default).  This should be enough to prevent the original version from being used.
* Copy the modified `stm32h7xx_hal_eth.c` and `ethernetif.c` files somewhere into your source tree.
* Exclude the previous versions of these files (i.e. `Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_eth.c` and `LWIP/Target/ethernetif.c` from your build.
* The original versions of `stm32h7xx_hal_eth.h`, `stm32h7xx_hal_eth.c` and `ethernetif.c` can then be safely deleted from your source tree (not mandatory, but may prevent confusion in the future).

## 2. Configure up memory map

I located all of the necessary memory in RAM_D2 as follows:

* 0x30000000: 128KB region (uncached) for Ethernet RX buffers
* 0x30020000: 16KB region (cached) for Ethernet TX buffers
* 0x30024000: 1KB region (uncached, strongly ordered) for Ethernet descriptors

The remainder of these instructions assume that this memory map is being used.  If a different memory map is used, then pay particular attention to the following:

* `EthRxBlock` should be aligned on a 128K boundary
* `EthDescriptorsBlock` should be aligned on a 1K boundary

Assuming that the memory map is as I described above, then add the following to `STM32H743ZITX_FLASH.ld`:

```
  .EthRxBlock (NOLOAD) :
  {
    /* MPU region for the ETH rx buffers: 128KB at start of D2 region.
     * Rationale: To conserve memory we want the rx buffers to be uncached.
     * If they were cached, the starts and ends of each rx buffer must be
     * cache-line aligned to avoid corrupting the pbuf part when the
     * buffer's invalidated following DMA receive. */
    . = ABSOLUTE(0x30000000);
    *(.RxArraySection)
  } >RAM_D2 AT> FLASH

  .EthTxBlock (NOLOAD) :
  {
    /* Normal, cached memory for the LwIP heap: 16KB following EthRxBlock.
     * Rationale: LwIP allocates ETH tx buffers on its heap, and so to position
     * the tx buffers in the D2 domain we must relocate the entire heap. */
    . = ABSOLUTE(0x30020000);
    *(.TxArraySection)
  } >RAM_D2 AT> FLASH

  .EthDescriptorsBlock (NOLOAD) :
  {
    /* MPU region for the ETH rx and tx descriptors: 1KB following EthTxBlock.
     * Rationale: to guarantee memory accesses complete in program order so
     * memory barriers aren't necessary. */
    . = ABSOLUTE(0x30024000);
    *(.RxDecripSection)
    *(.TxDecripSection)
    . = ALIGN(1K);
  } >RAM_D2 AT> FLASH
```

## 3. Adjust settings in STM32CubeIDE

### Ethernet configuration

* Navigate to *Connectivity* -> *ETH* -> *Parameter Settings*
* Set *Rx Buffers Length* to 1024
* The other values (*Tx Descriptor Length*, *First Tx Descriptor Address*, *Rx Descriptor Length*, *First Rx Descriptor Address*, *Rx Buffers Address*) are not used and can safely be set to 0
* Make sure that *Ethernet global interrupt* is enabled in *NVIC Settings*

### LWIP configuration

* Navigate to *Middleware* -> *LWIP* -> *Key Options*
* Enable *Show Advanced Parameters*
* *Infrastructure - Core Locking and MPU Option* -> *LWIP_MPU_COMPATIBLE (Special Memory Management)* -> Set to *Enabled*
* *Infrastructure - Heap and Memory Pools Options -> MEM_SIZE (Heap Memory Size)* -> Set to *16000 Byte(s)*
* *Infrastructure - Heap and Memory Pools Options -> LWIP_RAM_HEAP_POINTER (RAM Heap Pointer)* -> Set to 0x30020000

### MPU settings

* Nativate to *System Core* -> *CORETEX_M7* -> *Parameter Settings*
* Configure settings as shown below:

![MPU Configuration](screenshots/mpu_config.png?raw=true "MPU Configuration")

The resulting code should be as follows:

```
void MPU_Config(void)
{
  MPU_Region_InitTypeDef MPU_InitStruct = {0};

  /* Disables the MPU */
  HAL_MPU_Disable();
  /** Initializes and configures the Region and the memory to be protected
  */
  MPU_InitStruct.Enable = MPU_REGION_ENABLE;
  MPU_InitStruct.Number = MPU_REGION_NUMBER0;
  MPU_InitStruct.BaseAddress = 0x30000000;
  MPU_InitStruct.Size = MPU_REGION_SIZE_128KB;
  MPU_InitStruct.SubRegionDisable = 0x0;
  MPU_InitStruct.TypeExtField = MPU_TEX_LEVEL1;
  MPU_InitStruct.AccessPermission = MPU_REGION_FULL_ACCESS;
  MPU_InitStruct.DisableExec = MPU_INSTRUCTION_ACCESS_DISABLE;
  MPU_InitStruct.IsShareable = MPU_ACCESS_NOT_SHAREABLE;
  MPU_InitStruct.IsCacheable = MPU_ACCESS_NOT_CACHEABLE;
  MPU_InitStruct.IsBufferable = MPU_ACCESS_NOT_BUFFERABLE;

  HAL_MPU_ConfigRegion(&MPU_InitStruct);
  /** Initializes and configures the Region and the memory to be protected
  */
  MPU_InitStruct.Enable = MPU_REGION_ENABLE;
  MPU_InitStruct.Number = MPU_REGION_NUMBER1;
  MPU_InitStruct.BaseAddress = 0x30024000;
  MPU_InitStruct.Size = MPU_REGION_SIZE_1KB;
  MPU_InitStruct.SubRegionDisable = 0x0;
  MPU_InitStruct.TypeExtField = MPU_TEX_LEVEL0;
  MPU_InitStruct.AccessPermission = MPU_REGION_FULL_ACCESS;
  MPU_InitStruct.DisableExec = MPU_INSTRUCTION_ACCESS_DISABLE;
  MPU_InitStruct.IsShareable = MPU_ACCESS_NOT_SHAREABLE;
  MPU_InitStruct.IsCacheable = MPU_ACCESS_NOT_CACHEABLE;
  MPU_InitStruct.IsBufferable = MPU_ACCESS_BUFFERABLE;

  HAL_MPU_ConfigRegion(&MPU_InitStruct);
  /* Enables the MPU */
  HAL_MPU_Enable(MPU_PRIVILEGED_DEFAULT);

}
```

## 4. Regenerate code

*Project* -> *Generate Code*

## 5. Modify lwipopts.h

Add the following to the `USER CODE 1` section in `lwipopts.h`:

```
#define LWIP_DEBUG 0
```

## 6. Compile

*Project* -> *Build All*

## 7. Enjoy

# Acknowledgements
Thumbs up to Alister Fisher for his great work on improving the original driver from ST.  
Thumbs down to ST for providing such a broken driver in the first place.
