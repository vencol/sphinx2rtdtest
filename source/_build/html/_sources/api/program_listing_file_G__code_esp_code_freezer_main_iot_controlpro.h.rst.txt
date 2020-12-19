
.. _program_listing_file_G__code_esp_code_freezer_main_iot_controlpro.h:

Program Listing for File controlpro.h
=====================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_iot_controlpro.h>` (``G:\code\esp\code\freezer\main\iot\controlpro.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __TEMP_PROCESS__
   #define __TEMP_PROCESS__
   
   typedef enum {
       SET_FIRST_TSG,
       SET_WORK_TIMEOUT,
       SET_REMOTE_OFF=7,
   
       RESET_FIRST_TSG,
       RESET_WORK_TIMEOUT,
       RESET_REMOTE_OFF=15,
       SET_ERROR_ON,
   } control_flagbit_opt_t;
   
   void controlInit(void);
   void controlProcess(void);
   int isCompressorDelay(void);
   void powerOnControl(char status);
   #endif
   
