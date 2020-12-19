
.. _program_listing_file_G__code_esp_code_freezer_main_iot_iotmain.c:

Program Listing for File iotmain.c
==================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_iot_iotmain.c>` (``G:\code\esp\code\freezer\main\iot\iotmain.c``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /* WiFi station Example
   
      This example code is in the Public Domain (or CC0 licensed, at your option.)
   
      Unless required by applicable law or agreed to in writing, this
      software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
      CONDITIONS OF ANY KIND, either express or implied.
   */
   #include "freertos/FreeRTOS.h"
   #include "freertos/task.h"
   #include "iotmain.h"
   #include "controlpro.h"
   #include "keydisplay.h"
   
   void iotMainInit(void)
   {
     controlInit();
     displayInit();
   }
   static void iotMainTask(void *arg) 
   {
     while (1) 
     {
       vTaskDelay(10 / portTICK_PERIOD_MS);
       controlProcess();
       displayProcess();
     }
     vTaskDelete(NULL);
   }
   
   void iotMainPro(void)
   {
     xTaskCreate(iotMainTask, "iotMainTask", 1024*3, NULL, configMAX_PRIORITIES, NULL);
   }
