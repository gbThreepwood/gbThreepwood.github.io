+++
title = "tinyUSB CDC UART interface"
author = ["Eirik Haustveit"]
date = 2025-09-27T21:42:00+02:00
tags = ["tag1"]
categories = ["category1"]
draft = false
+++

In this article I will cover how to program a STM32f103C6 (<https://www.st.com/en/microcontrollers-microprocessors/stm32f103c6.html>) Bluepill board for use as a USB to serial port (UART) interface. The USB terminology for this kind of device is a communications device class (or USB CDC) device. These kind of interfaces have a lot of use cases, and by having native USB support in the microcontroller we can create multiple such interfaces, and even mix them with other things such as audio and have everything over a single USB cable.

The STM32f103C6 might not be powerfull enough for audio (time will show), but I intend to experiment with a combination of CDC and audio (UAC2) in the future, as I am working on several projects where this would be useful.

I will use the tinyUSB (<https://github.com/hathach/tinyusb>) library, which is a USB host and device stack for microcontrollers and other embedded systems. The documentation for the library is a bit deficient in my opinion, but there are a lot of good project examples in the repository.

It is possible to run the example projects directly from within the repository. Although this can be nice if one wants to get up and running as quickly as possible, it obscures the details required to configure tinyUSB for a custom project.


## Project configuration and initial tests {#project-configuration-and-initial-tests}

I generally prefer a custom CMake configuration for larger projects, while PlatformIO is my preferred solution for smaller project to get started quickly. My first step was to make sure that I could blink the onboard LED on the bluepill using the following program. As always I avoid the STM32Cube IDE and HAL library at all costs, here I have used the Low Layer (LL) library.

```c
#include <stm32f1xx_ll_gpio.h>

#include <stm32f1xx_ll_cortex.h>
#include <stm32f1xx_ll_rcc.h>
#include "stm32f1xx_ll_bus.h"

#define BUILTIN_LED_PORT          GPIOC
#define BUILTIN_LED_PIN           LL_GPIO_PIN_13

void SysTick_Handler(void)
{
    static uint_fast32_t counter = 0;
    counter++;

    // 4 Hz blinking
    if ((counter % 500) == 0)
        LL_GPIO_TogglePin(BUILTIN_LED_PORT, BUILTIN_LED_PIN);

}

void init_gpio() {

    LL_APB2_GRP1_EnableClock(LL_APB2_GRP1_PERIPH_GPIOC);
    LL_GPIO_SetPinMode(BUILTIN_LED_PORT, BUILTIN_LED_PIN, LL_GPIO_MODE_OUTPUT);
    LL_GPIO_SetPinOutputType(BUILTIN_LED_PORT, BUILTIN_LED_PIN, LL_GPIO_OUTPUT_PUSHPULL);
}

int main(void) {
    init_gpio();

    // 1kHz ticks
    SystemCoreClockUpdate();
    SysTick_Config(SystemCoreClock / 1000);

    while(1);

    return 0;
}
```

The _platformio.ini_ file looks like this:

```ini
[env:bluepill_f103c6]
platform = ststm32
board = bluepill_f103c6
framework = stm32cube
```

To flash the program to the microcontroller I used the stlink onboard a random Nucleo board I had lying around.

TODO: Add picture


## Configure the project for tinyUSB {#configure-the-project-for-tinyusb}

Start by cloning or git submoduling the tinyUSB library to the _lib_ folder.

In addition to our _main.c_ file any tinyUSB project should typically have a few more files.

-   tusb_config.h
-   usb_descriptors.c

I added the headers to the _include_ directory, and source files to the _src_ directory as is the convention in platformIO.

Furthermore I added:

-   helper_functions.h
-   helper_functions.c

It is of course possible to merge all the code in to a single file, but I will not do that.

The full project tree looks like this:

```nil
.
├── include
│   ├── helper_functions.h
│   ├── README
│   └── tusb_config.h
├── lib
│   ├── README
│   └── tinyusb
│       ├── CODE_OF_CONDUCT.rst
│       ├── CONTRIBUTORS.rst
│       ├── docs
│       ├── examples
│       ├── hw
│       ├── lib
│       ├── library.json
│       ├── LICENSE
│       ├── pkg.yml
│       ├── README.rst
│       ├── repository.yml
│       ├── SConscript
│       ├── src
│       ├── test
│       ├── tools
│       └── version.yml
├── platformio.ini
├── src
│   ├── helper_functions.c
│   ├── main.c
│   └── usb_descriptors.c
└── test
    └── README
```


### Helper functions {#helper-functions}

For now the only helper function is related to the generation of a unique serial number for our USB device. These functions are to be found inside the tinyUSB library but I extracted them since I want to limit the use of platform specific functions from within the USB library.

The header file is:

```c
#ifndef _HELPER_FUNCTIONS_H_
#define _HELPER_FUNCTIONS_H_

#ifdef __cplusplus
 extern "C" {
#endif

#include <string.h>
#include <stdint.h>

size_t stm32f103_get_unique_id(uint8_t id[], size_t max_len);

// Get USB Serial number string from unique ID if available. Return number of character.
// Input is string descriptor from index 1 (index 0 is type + len)
static inline size_t stm32f103_usb_get_serial(uint16_t desc_str1[], size_t max_chars) {
  uint8_t uid[16] TU_ATTR_ALIGNED(4);
  size_t uid_len;

  uid_len = stm32f103_get_unique_id(uid, sizeof(uid));

  if ( uid_len > max_chars / 2 ) uid_len = max_chars / 2;

  for ( size_t i = 0; i < uid_len; i++ ) {
    for ( size_t j = 0; j < 2; j++ ) {
      const char nibble_to_hex[16] = {
          '0', '1', '2', '3', '4', '5', '6', '7',
          '8', '9', 'A', 'B', 'C', 'D', 'E', 'F'
      };
      uint8_t const nibble = (uid[i] >> (j * 4)) & 0xf;
      desc_str1[i * 2 + (1 - j)] = nibble_to_hex[nibble]; // UTF-16-LE
    }
  }

  return 2 * uid_len;
}

#ifdef __cplusplus
 }
#endif

#endif /* _HELPER_FUNCTIONS_H_ */
```

The source file is:

```c
#include "tusb.h"
#include "helper_functions.h"
#include <string.h>
#include <stdint.h>

#include "stm32f103x6.h"

size_t stm32f103_get_unique_id(uint8_t id[], size_t max_len) {
  (void) max_len;
  volatile uint32_t * stm32_uuid = (volatile uint32_t *) UID_BASE;
  uint32_t* id32 = (uint32_t*) (uintptr_t) id;
  uint8_t const len = 12;

  id32[0] = stm32_uuid[0];
  id32[1] = stm32_uuid[1];
  id32[2] = stm32_uuid[2];

  return len;
}
```


### tusb_config.h {#tusb-config-dot-h}

This file holds the project specific configuration of the tinyUSB library. The only change I made was to define the microcontroller I use, and that I wanted to use CDC class.

```c
#ifndef _TUSB_CONFIG_H_
#define _TUSB_CONFIG_H_

#ifdef __cplusplus
 extern "C" {
#endif

//--------------------------------------------------------------------+
// Board Specific Configuration
//--------------------------------------------------------------------+

// RHPort number used for device can be defined by board.mk, default to port 0
#ifndef BOARD_TUD_RHPORT
#define BOARD_TUD_RHPORT      0
#endif

// RHPort max operational speed can defined by board.mk
#ifndef BOARD_TUD_MAX_SPEED
#define BOARD_TUD_MAX_SPEED   OPT_MODE_DEFAULT_SPEED
#endif

//--------------------------------------------------------------------
// Common Configuration
//--------------------------------------------------------------------

#define CFG_TUSB_MCU OPT_MCU_STM32F1

// defined by compiler flags for flexibility
#ifndef CFG_TUSB_MCU
#error CFG_TUSB_MCU must be defined
#endif

#ifndef CFG_TUSB_OS
#define CFG_TUSB_OS           OPT_OS_NONE
#endif

#ifndef CFG_TUSB_DEBUG
#define CFG_TUSB_DEBUG        0
#endif

// Enable Device stack
#define CFG_TUD_ENABLED       1

// Default is max speed that hardware controller could support with on-chip PHY
#define CFG_TUD_MAX_SPEED     BOARD_TUD_MAX_SPEED

/* USB DMA on some MCUs can only access a specific SRAM region with restriction on alignment.
 * Tinyusb use follows macros to declare transferring memory so that they can be put
 * into those specific section.
 * e.g
 * - CFG_TUSB_MEM SECTION : __attribute__ (( section(".usb_ram") ))
 * - CFG_TUSB_MEM_ALIGN   : __attribute__ ((aligned(4)))
 */
#ifndef CFG_TUSB_MEM_SECTION
#define CFG_TUSB_MEM_SECTION
#endif

#ifndef CFG_TUSB_MEM_ALIGN
#define CFG_TUSB_MEM_ALIGN    __attribute__ ((aligned(4)))
#endif

//--------------------------------------------------------------------
// DEVICE CONFIGURATION
//--------------------------------------------------------------------

#ifndef CFG_TUD_ENDPOINT0_SIZE
#define CFG_TUD_ENDPOINT0_SIZE   64
#endif

//------------- CLASS -------------//
#define CFG_TUD_CDC              1
#define CFG_TUD_MSC              0
#define CFG_TUD_HID              0
#define CFG_TUD_MIDI             0
#define CFG_TUD_VENDOR           0

#define CFG_TUD_CDC_NOTIFY        1 // Enable use of notification endpoint

// CDC FIFO size of TX and RX
#define CFG_TUD_CDC_RX_BUFSIZE   (TUD_OPT_HIGH_SPEED ? 512 : 64)
#define CFG_TUD_CDC_TX_BUFSIZE   (TUD_OPT_HIGH_SPEED ? 512 : 64)

// CDC Endpoint transfer buffer size, more is faster
#define CFG_TUD_CDC_EP_BUFSIZE   (TUD_OPT_HIGH_SPEED ? 512 : 64)

#ifdef __cplusplus
 }
#endif

#endif /* _TUSB_CONFIG_H_ */
```


### USB decriptors {#usb-decriptors}

The file _usb_decriptors.c_ configures the required USB descriptors that are sent to the host when the device is connected to the USB port. There is a lot of details to cover in this file.

TODO: Write a step by step walkthrough of this file

```c
#include "tusb.h"
#include "helper_functions.h"

/* A combination of interfaces must have a unique product id, since PC will save device driver after the first plug.
 * Same VID/PID with different interface e.g MSC (first), then CDC (later) will possibly cause system error on PC.
 *
 * Auto ProductID layout's Bitmap:
 *   [MSB]         HID | MSC | CDC          [LSB]
 */
#define _PID_MAP(itf, n)  ( (CFG_TUD_##itf) << (n) )

#define USB_PID           (0x4000 | _PID_MAP(CDC, 0) | _PID_MAP(MSC, 1) | _PID_MAP(HID, 2) | \
                           _PID_MAP(MIDI, 3) | _PID_MAP(VENDOR, 4) )

#define USB_VID   0xCafe

// USB release number in BCD (binary coded decimal)
#define USB_BCD   0x0200

//--------------------------------------------------------------------+
// Device Descriptors
//--------------------------------------------------------------------+
tusb_desc_device_t const desc_device = {
    .bLength            = sizeof(tusb_desc_device_t),
    .bDescriptorType    = TUSB_DESC_DEVICE,
    .bcdUSB             = USB_BCD,

    // Use Interface Association Descriptor (IAD) for CDC
    // As required by USB Specs IAD's subclass must be common class (2) and protocol must be IAD (1)
    .bDeviceClass       = TUSB_CLASS_MISC,
    .bDeviceSubClass    = MISC_SUBCLASS_COMMON,
    .bDeviceProtocol    = MISC_PROTOCOL_IAD,
    .bMaxPacketSize0    = CFG_TUD_ENDPOINT0_SIZE,

    .idVendor           = USB_VID,
    .idProduct          = USB_PID,
    .bcdDevice          = 0x0100,

    .iManufacturer      = 0x01,
    .iProduct           = 0x02,
    .iSerialNumber      = 0x03,

    .bNumConfigurations = 0x01
};

// Invoked when received GET DEVICE DESCRIPTOR
// Application return pointer to descriptor
uint8_t const *tud_descriptor_device_cb(void) {
  return (uint8_t const *) &desc_device;
}

//--------------------------------------------------------------------+
// Configuration Descriptor
//--------------------------------------------------------------------+

enum {
  ITF_NUM_CDC = 0,
  ITF_NUM_CDC_DATA,
  ITF_NUM_MSC,
  ITF_NUM_TOTAL
};

#if CFG_TUSB_MCU == OPT_MCU_LPC175X_6X || CFG_TUSB_MCU == OPT_MCU_LPC177X_8X || CFG_TUSB_MCU == OPT_MCU_LPC40XX
  // LPC 17xx and 40xx endpoint type (bulk/interrupt/iso) are fixed by its number
  // 0 control, 1 In, 2 Bulk, 3 Iso, 4 In, 5 Bulk etc ...
  #define EPNUM_CDC_NOTIF   0x81
  #define EPNUM_CDC_OUT     0x02
  #define EPNUM_CDC_IN      0x82

  #define EPNUM_MSC_OUT     0x05
  #define EPNUM_MSC_IN      0x85

#elif CFG_TUSB_MCU == OPT_MCU_CXD56
  // CXD56 USB driver has fixed endpoint type (bulk/interrupt/iso) and direction (IN/OUT) by its number
  // 0 control (IN/OUT), 1 Bulk (IN), 2 Bulk (OUT), 3 In (IN), 4 Bulk (IN), 5 Bulk (OUT), 6 In (IN)
  #define EPNUM_CDC_NOTIF   0x83
  #define EPNUM_CDC_OUT     0x02
  #define EPNUM_CDC_IN      0x81

  #define EPNUM_MSC_OUT     0x05
  #define EPNUM_MSC_IN      0x84

#elif defined(TUD_ENDPOINT_ONE_DIRECTION_ONLY)
  // MCUs that don't support a same endpoint number with different direction IN and OUT defined in tusb_mcu.h
  //    e.g EP1 OUT & EP1 IN cannot exist together
  #define EPNUM_CDC_NOTIF   0x81
  #define EPNUM_CDC_OUT     0x02
  #define EPNUM_CDC_IN      0x83

  #define EPNUM_MSC_OUT     0x04
  #define EPNUM_MSC_IN      0x85

#else
  #define EPNUM_CDC_NOTIF   0x81
  #define EPNUM_CDC_OUT     0x02
  #define EPNUM_CDC_IN      0x82

  #define EPNUM_MSC_OUT     0x03
  #define EPNUM_MSC_IN      0x83

#endif

#define CONFIG_TOTAL_LEN    (TUD_CONFIG_DESC_LEN + TUD_CDC_DESC_LEN) // + TUD_MSC_DESC_LEN)

// full speed configuration
uint8_t const desc_fs_configuration[] = {
    // Config number, interface count, string index, total length, attribute, power in mA
    TUD_CONFIG_DESCRIPTOR(1, ITF_NUM_TOTAL, 0, CONFIG_TOTAL_LEN, 0x00, 100),

    // Interface number, string index, EP notification address and size, EP data address (out, in) and size.
    TUD_CDC_DESCRIPTOR(ITF_NUM_CDC, 4, EPNUM_CDC_NOTIF, 16, EPNUM_CDC_OUT, EPNUM_CDC_IN, 64),
};

#if TUD_OPT_HIGH_SPEED
// Per USB specs: high speed capable device must report device_qualifier and other_speed_configuration

// high speed configuration
uint8_t const desc_hs_configuration[] = {
    // Config number, interface count, string index, total length, attribute, power in mA
    TUD_CONFIG_DESCRIPTOR(1, ITF_NUM_TOTAL, 0, CONFIG_TOTAL_LEN, 0x00, 100),

    // Interface number, string index, EP notification address and size, EP data address (out, in) and size.
    TUD_CDC_DESCRIPTOR(ITF_NUM_CDC, 4, EPNUM_CDC_NOTIF, 16, EPNUM_CDC_OUT, EPNUM_CDC_IN, 512),
};

// other speed configuration
uint8_t desc_other_speed_config[CONFIG_TOTAL_LEN];

// device qualifier is mostly similar to device descriptor since we don't change configuration based on speed
tusb_desc_device_qualifier_t const desc_device_qualifier = {
    .bLength            = sizeof(tusb_desc_device_qualifier_t),
    .bDescriptorType    = TUSB_DESC_DEVICE_QUALIFIER,
    .bcdUSB             = USB_BCD,

    .bDeviceClass       = TUSB_CLASS_MISC,
    .bDeviceSubClass    = MISC_SUBCLASS_COMMON,
    .bDeviceProtocol    = MISC_PROTOCOL_IAD,

    .bMaxPacketSize0    = CFG_TUD_ENDPOINT0_SIZE,
    .bNumConfigurations = 0x01,
    .bReserved          = 0x00
};

// Invoked when received GET DEVICE QUALIFIER DESCRIPTOR request
// Application return pointer to descriptor, whose contents must exist long enough for transfer to complete.
// device_qualifier descriptor describes information about a high-speed capable device that would
// change if the device were operating at the other speed. If not highspeed capable stall this request.
uint8_t const *tud_descriptor_device_qualifier_cb(void) {
  return (uint8_t const *) &desc_device_qualifier;
}

// Invoked when received GET OTHER SEED CONFIGURATION DESCRIPTOR request
// Application return pointer to descriptor, whose contents must exist long enough for transfer to complete
// Configuration descriptor in the other speed e.g if high speed then this is for full speed and vice versa
uint8_t const *tud_descriptor_other_speed_configuration_cb(uint8_t index) {
  (void) index; // for multiple configurations

  // if link speed is high return fullspeed config, and vice versa
  // Note: the descriptor type is OHER_SPEED_CONFIG instead of CONFIG
  memcpy(desc_other_speed_config,
         (tud_speed_get() == TUSB_SPEED_HIGH) ? desc_fs_configuration : desc_hs_configuration,
         CONFIG_TOTAL_LEN);

  desc_other_speed_config[1] = TUSB_DESC_OTHER_SPEED_CONFIG;

  return desc_other_speed_config;
}

#endif // highspeed

// Invoked when received GET CONFIGURATION DESCRIPTOR
// Application return pointer to descriptor
// Descriptor contents must exist long enough for transfer to complete
uint8_t const *tud_descriptor_configuration_cb(uint8_t index) {
  (void) index; // for multiple configurations

#if TUD_OPT_HIGH_SPEED
  // Although we are highspeed, host may be fullspeed.
  return (tud_speed_get() == TUSB_SPEED_HIGH) ? desc_hs_configuration : desc_fs_configuration;
#else
  return desc_fs_configuration;
#endif
}

//--------------------------------------------------------------------+
// String Descriptors
//--------------------------------------------------------------------+

// String Descriptor Index
enum {
  STRID_LANGID = 0,
  STRID_MANUFACTURER,
  STRID_PRODUCT,
  STRID_SERIAL,
};

// array of pointer to string descriptors
char const *string_desc_arr[] = {
    (const char[]) { 0x09, 0x04 }, // 0: is supported language is English (0x0409)
    "Eirik Haustveit",                       // 1: Manufacturer
    "USB dingselur",              // 2: Product
    NULL,                          // 3: Serials will use unique ID if possible
    "dingsUSB CDC",                 // 4: CDC Interface
    "dingsUSB MSC",                 // 5: MSC Interface
};

static uint16_t _desc_str[32 + 1];

// Invoked when received GET STRING DESCRIPTOR request
// Application return pointer to descriptor, whose contents must exist long enough for transfer to complete
uint16_t const *tud_descriptor_string_cb(uint8_t index, uint16_t langid) {
  (void) langid;
  size_t chr_count;

  switch ( index ) {
    case STRID_LANGID:
      memcpy(&_desc_str[1], string_desc_arr[0], 2);
      chr_count = 1;
      break;

    case STRID_SERIAL:
      chr_count = stm32f103_usb_get_serial(_desc_str + 1, 32);
      break;

    default:
      // Note: the 0xEE index string is a Microsoft OS 1.0 Descriptors.
      // https://docs.microsoft.com/en-us/windows-hardware/drivers/usbcon/microsoft-defined-usb-descriptors

      if ( !(index < sizeof(string_desc_arr) / sizeof(string_desc_arr[0])) ) { return NULL; }

      const char *str = string_desc_arr[index];

      // Cap at max char
      chr_count = strlen(str);
      size_t const max_count = sizeof(_desc_str) / sizeof(_desc_str[0]) - 1; // -1 for string type
      if ( chr_count > max_count ) { chr_count = max_count; }

      // Convert ASCII string into UTF-16
      for ( size_t i = 0; i < chr_count; i++ ) {
        _desc_str[1 + i] = str[i];
      }
      break;
  }

  // first byte is length (including header), second byte is string type
  _desc_str[0] = (uint16_t) ((TUSB_DESC_STRING << 8) | (2 * chr_count + 2));
  return _desc_str;
}
```


### main.c {#main-dot-c}

_main.c_ source file. Here I have adapted the blink example from before to include the required I/O configuration, callbacks, and interrupt service routines for tinyUSB.

The most interesting things happens in the _cdc_task()_ function where the data received on the USB serial port interface is echoed back, but also included in the echo is the number of bytes received.

```c
#include <stm32f1xx_ll_gpio.h>

#include <stm32f1xx_ll_cortex.h>
#include <stm32f1xx_ll_rcc.h>
#include <stm32f1xx_ll_bus.h>
#include <stm32f1xx_ll_usb.h>

#include <tusb.h>
#include "tusb_config.h"

#include <string.h>
#include <stdio.h>

#define BUILTIN_LED_PORT          GPIOC
#define BUILTIN_LED_PIN           LL_GPIO_PIN_13

void cdc_task(void);


static uint_fast32_t counter = 0;
void SysTick_Handler(void)
{
    counter++;

    // blinking
    if ((counter % 1500) == 0)
        LL_GPIO_TogglePin(BUILTIN_LED_PORT, BUILTIN_LED_PIN);

}

void init_clock(){
    RCC_ClkInitTypeDef clkinitstruct = {0};
    RCC_OscInitTypeDef oscinitstruct = {0};
    RCC_PeriphCLKInitTypeDef rccperiphclkinit = {0};

    // TODO: Rewrite this to use LL library

    /* Enable HSE Oscillator and activate PLL with HSE as source */
    oscinitstruct.OscillatorType  = RCC_OSCILLATORTYPE_HSE;
    oscinitstruct.HSEState        = RCC_HSE_ON;
    oscinitstruct.HSEPredivValue  = RCC_HSE_PREDIV_DIV1;
    oscinitstruct.PLL.PLLMUL      = RCC_PLL_MUL9;
    oscinitstruct.PLL.PLLState    = RCC_PLL_ON;
    oscinitstruct.PLL.PLLSource   = RCC_PLLSOURCE_HSE;
    HAL_RCC_OscConfig(&oscinitstruct);

    /* USB clock selection */
    rccperiphclkinit.PeriphClockSelection = RCC_PERIPHCLK_USB;
    rccperiphclkinit.UsbClockSelection = RCC_USBCLKSOURCE_PLL_DIV1_5;
    HAL_RCCEx_PeriphCLKConfig(&rccperiphclkinit);

    /* Select PLL as system clock source and configure the HCLK, PCLK1 and PCLK2 clocks dividers */
    clkinitstruct.ClockType = (RCC_CLOCKTYPE_SYSCLK | RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2);
    clkinitstruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    clkinitstruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
    clkinitstruct.APB1CLKDivider = RCC_HCLK_DIV2;
    clkinitstruct.APB2CLKDivider = RCC_HCLK_DIV1;
    HAL_RCC_ClockConfig(&clkinitstruct, FLASH_LATENCY_2);
}

void init_gpio() {

    //LL_AHB2_GRP1_EnableClock(LL_AHB2_GRP1_PERIPH_GPIOA);

    //LL_AHB2_GRP1_EnableClock(LL_AHB2_GRP1_PERIPH_GPIOB);
    LL_APB2_GRP1_EnableClock(LL_APB2_GRP1_PERIPH_GPIOC);

    LL_GPIO_SetPinMode(BUILTIN_LED_PORT, BUILTIN_LED_PIN, LL_GPIO_MODE_OUTPUT);

    LL_GPIO_SetPinOutputType(BUILTIN_LED_PORT, BUILTIN_LED_PIN, LL_GPIO_OUTPUT_PUSHPULL);

    LL_GPIO_SetOutputPin(BUILTIN_LED_PORT, BUILTIN_LED_PIN);
}

void init_usb(){

    LL_APB2_GRP1_EnableClock(LL_APB2_GRP1_PERIPH_GPIOA);

    LL_GPIO_SetPinMode(GPIOA, GPIO_PIN_11, LL_GPIO_MODE_ALTERNATE);
    LL_GPIO_SetPinMode(GPIOA, GPIO_PIN_12, LL_GPIO_MODE_ALTERNATE);

    LL_GPIO_SetPinOutputType(GPIOA, GPIO_PIN_11, LL_GPIO_OUTPUT_PUSHPULL);
    LL_GPIO_SetPinOutputType(GPIOA, GPIO_PIN_12, LL_GPIO_OUTPUT_PUSHPULL);

    LL_GPIO_SetPinSpeed(GPIOA, GPIO_PIN_11, LL_GPIO_SPEED_FREQ_HIGH);
    LL_GPIO_SetPinSpeed(GPIOA, GPIO_PIN_12, LL_GPIO_SPEED_FREQ_HIGH);


    LL_APB1_GRP1_EnableClock(LL_APB1_GRP1_PERIPH_USB);

}

int main(void) {

    init_clock();

    init_gpio();

    // 1kHz ticks
    SystemCoreClockUpdate();
    SysTick_Config(SystemCoreClock / 1000);

    init_usb();

    // init device stack on configured roothub port
    tusb_rhport_init_t dev_init = {
      .role = TUSB_ROLE_DEVICE,
      .speed = TUSB_SPEED_AUTO
    };
    tusb_init(BOARD_TUD_RHPORT, &dev_init);


    while (1) {
        tud_task(); // tinyusb device task

        cdc_task();
    }

    return 0;
}


//--------------------------------------------------------------------+
// Device callbacks
//--------------------------------------------------------------------+

// Invoked when device is mounted
void tud_mount_cb(void) {
}

// Invoked when device is unmounted
void tud_umount_cb(void) {
}

// Invoked when usb bus is suspended
// remote_wakeup_en : if host allow us  to perform remote wakeup
// Within 7ms, device must draw an average of current less than 2.5 mA from bus
void tud_suspend_cb(bool remote_wakeup_en) {
  (void) remote_wakeup_en;
}

// Invoked when usb bus is resumed
void tud_resume_cb(void) {
}


//--------------------------------------------------------------------+
// USB CDC
//--------------------------------------------------------------------+
void cdc_task(void) {
  // connected() check for DTR bit
  // Most but not all terminal client set this when making connection
  // if ( tud_cdc_connected() )
  {
    // connected and there are data available
    if (tud_cdc_available()) {
      // read data
      char buf[64];
      uint32_t count = tud_cdc_read(buf, sizeof(buf));
      //(void) count;

      // Echo back
      // Note: Skip echo by commenting out write() and write_flush()
      // for throughput test e.g
      //    $ dd if=/dev/zero of=/dev/ttyACM0 count=10000

      char message_buf[22];

      // Send the length of the received data back to the sender
      snprintf(message_buf, sizeof(message_buf), "Length: %li\r\n", count);
      tud_cdc_write(message_buf, strlen(message_buf));

      tud_cdc_write(buf, count);
      tud_cdc_write_flush();
    }

    // Press on-board button to send Uart status notification
    static uint32_t btn_prev = 0;
    static cdc_notify_uart_state_t uart_state = { .value = 0 };
    const uint32_t btn = 0; //board_button_read();
    if (!btn_prev && btn) {
      uart_state.dsr ^= 1;
      tud_cdc_notify_uart_state(&uart_state);
    }
    btn_prev = btn;
  }
}

// Invoked when cdc when line state changed e.g connected/disconnected
void tud_cdc_line_state_cb(uint8_t itf, bool dtr, bool rts) {
  (void) itf;
  (void) rts;

  // TODO set some indicator
  if (dtr) {
    // Terminal connected
  } else {
    // Terminal disconnected
  }
}

// Invoked when CDC interface received data from host
void tud_cdc_rx_cb(uint8_t itf) {
  (void) itf;
}

//--------------------------------------------------------------------+
// Forward USB interrupt events to TinyUSB IRQ Handler
//--------------------------------------------------------------------+
void USB_HP_IRQHandler(void) {
  tud_int_handler(0);
}

void USB_LP_IRQHandler(void) {
  tud_int_handler(0);
}

void USBWakeUp_IRQHandler(void) {
  tud_int_handler(0);
}
```


### Testing {#testing}

After flashing of the microcontroller _dmesg_ gives the following output:

```console
$ sudo dmesg | tail -n 8
[sudo] password for eirik:
[24151.861921] usb 5-2.3.3: new full-speed USB device number 65 using xhci_hcd
[24151.953663] usb 5-2.3.3: config 1 has 2 interfaces, different from the descriptor's value: 3
[24151.954623] usb 5-2.3.3: New USB device found, idVendor=cafe, idProduct=4001, bcdDevice= 1.00
[24151.954633] usb 5-2.3.3: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[24151.954638] usb 5-2.3.3: Product: USB dingselur
[24151.954643] usb 5-2.3.3: Manufacturer: Eirik Haustveit
[24151.954647] usb 5-2.3.3: SerialNumber: 32FFDC054D58313642631943
[24151.970098] cdc_acm 5-2.3.3:1.0: ttyACM1: USB ACM device
```

_lsusb_ also provides some information:

```console
$ lsusb
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
...
Bus 005 Device 065: ID cafe:4001 Eirik Haustveit USB dingselur
...
```

My slightly silly echo application with length indication yields the following when typing "test" in to the console:

```console
$ picocom -b 115200 /dev/ttyACM1
picocom v3.1

port is        : /dev/ttyACM1
flowcontrol    : none
baudrate is    : 115200
parity is      : none
databits are   : 8
stopbits are   : 1
escape is      : C-a
local echo is  : no
noinit is      : no
noreset is     : no
hangup is      : no
nolock is      : no
send_cmd is    : sz -vv
receive_cmd is : rz -vv -E
imap is        :
omap is        :
emap is        : crcrlf,delbs,
logfile is     : none
initstring     : none
exit_after is  : not set
exit is        : no

Type [C-a] [C-h] to see available commands
Terminal ready
Length: 1
tLength: 1
eLength: 1
sLength: 1
t
```
