
.. _program_listing_file_G__code_esp_code_freezer_main_freezerdrv_drv_1650.h:

Program Listing for File drv_1650.h
===================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_freezerdrv_drv_1650.h>` (``G:\code\esp\code\freezer\main\freezerdrv\drv_1650.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __DRV_1650__
   #define __DRV_1650__
   
   
   typedef enum {
       LEVEL_8,
       LEVEL_1,
       LEVEL_2,
       LEVEL_3,
       LEVEL_4,
       LEVEL_5,
       LEVEL_6,
       LEVEL_7
   } display_light_level;
   
   #define TM1650_SEGMENT_8            0
   #define TM1650_SEGMENT_7            1
   #define TM1650_DISPLAY_ON           1
   #define TM1650_DISPLAY_OFF          0
   #define TM1650_DISPLAY_VALUE(level, seg, on)        ( (level)<<4 | (seg)<<3 | (on) )
   
   int drvTm1650Init(void);
   int tm1650ReadKey(uint8_t *key);
   int tm1650SetDispaly(uint8_t data);
   int tm1650SetDispalyData(uint8_t *data, uint8_t size);
   
   #endif
   
