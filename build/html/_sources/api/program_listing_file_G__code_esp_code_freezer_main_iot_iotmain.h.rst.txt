
.. _program_listing_file_G__code_esp_code_freezer_main_iot_iotmain.h:

Program Listing for File iotmain.h
==================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_iot_iotmain.h>` (``G:\code\esp\code\freezer\main\iot\iotmain.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __IOT_MAIN__
   #define __IOT_MAIN__
   
   
   //at
   void atCmdPro(void);
   int atCmdInit(void);
   
   void iotMainInit(void);
   void iotMainPro(void);
   
   void keyDisplayInit(void);
   void keyDisplayPro(void);
   
   
   
   #endif
   
