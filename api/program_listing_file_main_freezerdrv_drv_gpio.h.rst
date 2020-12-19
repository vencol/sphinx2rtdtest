
.. _program_listing_file_main_freezerdrv_drv_gpio.h:

Program Listing for File drv_gpio.h
===================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_freezerdrv_drv_gpio.h>` (``main\freezerdrv\drv_gpio.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __DRV_GPIO__
   #define __DRV_GPIO__
   
   #include "driver/gpio.h"
   
   void drvOutputGpioInit(void);
   void drvGpioPro(void);
   void setModuleStatus(char type);
   
   #define OUTPUT_HIGH                     (1)
   #define OUTPUT_LOW                      (0)
   
   #define OUTPUT_COMPRESSOR_NUMBER        (21)
   #define OUTPUT_FAN_NUMBER               (19)
   #define OUTPUT_4GPOWER_NUMBER           (27)//4g on off
   #define OUTPUT_4GPWRKEY_NUMBER          (26)//4g pwrkey
   #define OUTPUT_4GRESET_NUMBER           (25)//4g reset
   #define INTERRUPT_ZERO_NUMBER           (32)
   
   #define GPIO_OUTPUT_NUM(pin)                ( 1ULL<<(pin) )
   #define OUTPUT_COMPRESSOR                   ( GPIO_OUTPUT_NUM(OUTPUT_COMPRESSOR_NUMBER) )
   #define OUTPUT_FAN                          ( GPIO_OUTPUT_NUM(OUTPUT_FAN_NUMBER) )
   #define OUTPUT_4GPOWER                      ( GPIO_OUTPUT_NUM(OUTPUT_4GPOWER_NUMBER) )
   #define OUTPUT_4GPWRKEY                     ( GPIO_OUTPUT_NUM(OUTPUT_4GPWRKEY_NUMBER) )
   #define OUTPUT_4GRESET                      ( GPIO_OUTPUT_NUM(OUTPUT_4GRESET_NUMBER) )
   #define FREEZER_OUTPUT_GPIO()               (OUTPUT_COMPRESSOR | OUTPUT_FAN | OUTPUT_4GPOWER | OUTPUT_4GPWRKEY | OUTPUT_4GRESET)//
   
   #define SET_COMPRESSOR_OUTPUT(x)            do{ gpio_set_level(OUTPUT_COMPRESSOR_NUMBER, (x)); gpio_set_level(OUTPUT_FAN_NUMBER, (x));}while(0)
   #define SET_FAN_OUTPUT(x)                   do{ gpio_set_level(OUTPUT_FAN_NUMBER, (x)); }while(0)
   #define SET_4GPOWER_OUTPUT(x)               do{ gpio_set_level(OUTPUT_4GPOWER_NUMBER, (x)); }while(0)
   #define SET_4GPWRKEY_OUTPUT(x)              do{ gpio_set_level(OUTPUT_4GPWRKEY_NUMBER, (x)); }while(0)
   #define SET_4GRESET_OUTPUT(x)               do{ gpio_set_level(OUTPUT_4GRESET_NUMBER, (x)); }while(0)
   
   #define INTERRUPT_ZERO                      ( GPIO_OUTPUT_NUM(INTERRUPT_ZERO_NUMBER) )
   #define FREEZER_INTERRUPT_GPIO()            (INTERRUPT_ZERO )//
   #endif
   
