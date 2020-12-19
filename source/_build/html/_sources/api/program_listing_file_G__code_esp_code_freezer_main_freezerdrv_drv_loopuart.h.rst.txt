
.. _program_listing_file_G__code_esp_code_freezer_main_freezerdrv_drv_loopuart.h:

Program Listing for File drv_loopuart.h
=======================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_freezerdrv_drv_loopuart.h>` (``G:\code\esp\code\freezer\main\freezerdrv\drv_loopuart.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __DRV_LOOPUART__
   #define __DRV_LOOPUART__
   
   typedef struct   
   { 
       char *Buffer;    
       int BufferWriteptr;
       int BufferReadptr;
       int MaxLen;   
   }stLoopBuffer; 
   
   
   int drvLoop_bufferInit(stLoopBuffer *loopbuf, char *buf, int maxlen);
   int drvLoop_CanWriteData(stLoopBuffer *loopbuf, int len);
   int drvLoop_WriteData(stLoopBuffer *loopbuf, char *data, int len);
   int drvLoop_CanReadData(stLoopBuffer *loopbuf, int start, int len);
   int drvLoop_ReadData(stLoopBuffer *loopbuf, char * data, int start, int len);
   #endif
   
