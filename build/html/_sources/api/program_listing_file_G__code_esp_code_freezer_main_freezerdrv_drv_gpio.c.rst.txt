
.. _program_listing_file_G__code_esp_code_freezer_main_freezerdrv_drv_gpio.c:

Program Listing for File drv_gpio.c
===================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_freezerdrv_drv_gpio.c>` (``G:\code\esp\code\freezer\main\freezerdrv\drv_gpio.c``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /* ADC1 Example
   
      This example code is in the Public Domain (or CC0 licensed, at your option.)
   
      Unless required by applicable law or agreed to in writing, this
      software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
      CONDITIONS OF ANY KIND, either express or implied.
   */
   #include <stdio.h>
   #include <string.h>
   #include "freertos/FreeRTOS.h"
   #include "freertos/task.h"
   #include "esp_attr.h"
   #include "drv_gpio.h"
   
   #define ESP_INTR_FLAG_DEFAULT 0
   #define MODULE_4G_CLOSE_TIME           (1)
   #define MODULE_4G_OPEN_TIME            (MODULE_4G_CLOSE_TIME + 5)
   #define MODULE_4G_PWRKEY_OFF_TIME      (MODULE_4G_OPEN_TIME + 5)
   #define MODULE_4G_PWRKEY_REON_TIME     (MODULE_4G_PWRKEY_OFF_TIME + 10)
   #define MODULE_4G_OPEN_MAX_TIME        (MODULE_4G_PWRKEY_REON_TIME + 100)
   // Forces code into IRAM instead of flash
   // #define IRAM_ATTR _SECTION_ATTR_IMPL(".iram1", __COUNTER__)
       // static unsigned short abc=0;
       // if(++abc == 1)
       //   SET_4GRESET_OUTPUT(1);
       // else if(abc == 500)
       //   SET_4GRESET_OUTPUT(0);
       // else if(abc == 1000)
       //   abc = 0;
   
   static bool on=true;
   static unsigned short powerOn4gTime = 0;
   
   
   static void IRAM_ATTR gpio_isr_handler(void* arg)
   {
      on = !on;
   }
   static void moduleGpioPro(void)//10ms
   {
      if(powerOn4gTime) {
         if(powerOn4gTime == MODULE_4G_CLOSE_TIME)   {
            SET_4GPOWER_OUTPUT(OUTPUT_LOW);
            SET_4GPWRKEY_OUTPUT(OUTPUT_LOW);
            SET_4GRESET_OUTPUT(OUTPUT_LOW);
         }
         else if(powerOn4gTime == MODULE_4G_OPEN_TIME)   {
            SET_4GPOWER_OUTPUT(OUTPUT_HIGH);
            SET_4GPWRKEY_OUTPUT(OUTPUT_HIGH);
         }
         else if (powerOn4gTime == MODULE_4G_PWRKEY_OFF_TIME)  
            SET_4GPWRKEY_OUTPUT(OUTPUT_LOW);
         else if (powerOn4gTime == MODULE_4G_PWRKEY_REON_TIME)  
            SET_4GPWRKEY_OUTPUT(OUTPUT_HIGH);
         else if (powerOn4gTime == MODULE_4G_OPEN_MAX_TIME)  
            powerOn4gTime = MODULE_4G_OPEN_MAX_TIME - 1;
         powerOn4gTime++;
      }
   }
   void setModuleStatus(char type)
   {
      if(type) 
         powerOn4gTime = MODULE_4G_CLOSE_TIME;
      else  {
         SET_4GPOWER_OUTPUT(OUTPUT_LOW);
         SET_4GPWRKEY_OUTPUT(OUTPUT_LOW);
         SET_4GRESET_OUTPUT(OUTPUT_LOW);
         powerOn4gTime = 0;
      }
   }
   
   void drvOutputGpioInit(void)
   {
       gpio_config_t io_conf;
       //disable interrupt
       io_conf.intr_type = GPIO_PIN_INTR_DISABLE;
       //set as output mode
       io_conf.mode = GPIO_MODE_OUTPUT;
       //bit mask of the pins that you want to set,e.g.GPIO18/19
       io_conf.pin_bit_mask = FREEZER_OUTPUT_GPIO();
       //disable pull-down mode
       io_conf.pull_down_en = 0;
       //disable pull-up mode
       io_conf.pull_up_en = 0;
       //configure GPIO with the given settings
       gpio_config(&io_conf);
   
      SET_COMPRESSOR_OUTPUT(OUTPUT_LOW);
      SET_FAN_OUTPUT(OUTPUT_LOW);
      SET_4GPOWER_OUTPUT(OUTPUT_LOW);
      SET_4GPWRKEY_OUTPUT(OUTPUT_LOW);
      SET_4GRESET_OUTPUT(OUTPUT_LOW);
   
      //  interrupt of rising edge
       io_conf.intr_type = GPIO_PIN_INTR_NEGEDGE;
       //bit mask of the pins, use GPIO4/5 here
       io_conf.pin_bit_mask = FREEZER_INTERRUPT_GPIO();
       //set as input mode    
       io_conf.mode = GPIO_MODE_INPUT;
       //enable pull-up mode
       io_conf.pull_up_en = 1;
       gpio_config(&io_conf);
       //install gpio isr service
       gpio_install_isr_service(ESP_INTR_FLAG_DEFAULT);
       //change gpio intrrupt type for one pin
       gpio_set_intr_type(INTERRUPT_ZERO_NUMBER, GPIO_PIN_INTR_NEGEDGE);
       //hook isr handler for specific gpio pin
       gpio_isr_handler_add(INTERRUPT_ZERO_NUMBER, gpio_isr_handler, (void*) INTERRUPT_ZERO_NUMBER);
      //  //remove isr handler for gpio number.
      //  gpio_isr_handler_remove(GPIO_INPUT_IO_0);
   
   }
   void drvGpioPro(void)//10ms
   {
      moduleGpioPro();
   }
