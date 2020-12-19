
.. _program_listing_file_main_freezerdrv_drv_loopuart.c:

Program Listing for File drv_loopuart.c
=======================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_freezerdrv_drv_loopuart.c>` (``main\freezerdrv\drv_loopuart.c``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /* UART asynchronous example, that uses separate RX and TX tasks
   
      This example code is in the Public Domain (or CC0 licensed, at your option.)
   
      Unless required by applicable law or agreed to in writing, this
      software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
      CONDITIONS OF ANY KIND, either express or implied.
   */
   #include "esp_system.h"
   #include "esp_log.h"
   #include "string.h"
   #include "drv_loopuart.h"
   
   static const char *LOOPTAG = "drvuart";
   
   
   int drvLoop_bufferInit(stLoopBuffer *loopbuf, char *buf, int maxlen)
   {
       if(loopbuf == NULL || buf == NULL)
           return -1;
       loopbuf->Buffer                 = buf;
       loopbuf->MaxLen                 = maxlen;
       loopbuf->BufferReadptr  = 0;
       loopbuf->BufferWriteptr = 0;
       return 0;
   }
   int drvLoop_CanWriteData(stLoopBuffer *loopbuf, int len)
   {
       // ESP_LOGI(LOOPTAG, "%s MaxLen: %d  len: %d BufferReadptr: %d BufferReadptr: %d", __FUNCTION__ , loopbuf->MaxLen, len, loopbuf->BufferWriteptr, loopbuf->BufferReadptr);
       if( (loopbuf->BufferReadptr <= loopbuf->BufferWriteptr && loopbuf->BufferReadptr + loopbuf->MaxLen - loopbuf->BufferWriteptr < len)
            || (loopbuf->BufferReadptr > loopbuf->BufferWriteptr && loopbuf->BufferReadptr - loopbuf->BufferWriteptr < len)
            || len < 1 )
           return -1; 
       // ESP_LOGI(LOOPTAG, "%s len %d from %d ", __FUNCTION__ , len, loopbuf->BufferWriteptr);
       return 0; 
   }
   int drvLoop_WriteData(stLoopBuffer *loopbuf, char *data, int len)
   {
       int i;
       if( drvLoop_CanWriteData(loopbuf, len) == 0 )
       {
           if( (loopbuf->BufferWriteptr >= loopbuf->BufferReadptr && loopbuf->BufferWriteptr + len <= loopbuf->MaxLen) 
               || (loopbuf->BufferWriteptr < loopbuf->BufferReadptr && loopbuf->BufferWriteptr + len <= loopbuf->BufferReadptr) )
           {
               for(i=0; i<len; i++)
                   *(loopbuf->Buffer + loopbuf->BufferWriteptr++) = *(data++);
               // ESP_LOGI(LOOPTAG, "%s BufferWriteptr  %d len:%d", __FUNCTION__ , loopbuf->BufferWriteptr, len);
           }
           else
           {
               len = len + loopbuf->BufferWriteptr - loopbuf->MaxLen;
               for(i=loopbuf->BufferWriteptr; i<loopbuf->MaxLen; i++)
                   *(loopbuf->Buffer + i) = *(data++);
               for(i=0; i<len; i++)
                   *(loopbuf->Buffer + i) = *(data++);
               loopbuf->BufferWriteptr = len;
               // ESP_LOGI(LOOPTAG, "%s BufferWriteptr change %d len:%d", __FUNCTION__ , loopbuf->BufferWriteptr, len);
           }
       }
       return 0;
   }
   int drvLoop_CanReadData(stLoopBuffer *loopbuf, int start, int len)
   {
       if( (start > loopbuf->BufferWriteptr && loopbuf->BufferWriteptr + loopbuf->MaxLen - start < len)
            || (start <= loopbuf->BufferWriteptr && loopbuf->BufferWriteptr - start < len) 
            || len < 1 )
           return -1;
       return 0; 
   }
   int drvLoop_ReadData(stLoopBuffer *loopbuf, char * data, int start, int len)
   {
       int i;
       //maybe nouse data between start and loopbuf->BufferReadptr
       if( drvLoop_CanReadData(loopbuf, start, len) == 0 )
       {
           loopbuf->BufferReadptr = start;
           if(loopbuf->BufferReadptr <= loopbuf->BufferWriteptr)
           {
               for(i=0; i<len; i++)
                   *(data++) = *(loopbuf->Buffer + loopbuf->BufferReadptr++);
               // ESP_LOGI(LOOPTAG, "%s BufferReadptr %d ", __FUNCTION__ , loopbuf->BufferReadptr);
           }
           else
           {
               len = len + loopbuf->BufferReadptr - loopbuf->MaxLen;
               for(i=loopbuf->BufferReadptr; i<loopbuf->MaxLen; i++)
                   *(data++) = *(loopbuf->Buffer + i);
               for(i=0; i<len; i++)
                   *(data++) = *(loopbuf->Buffer + i);
               loopbuf->BufferReadptr = len;
               // ESP_LOGI(LOOPTAG, "%s BufferReadptr change %d ", __FUNCTION__ , loopbuf->BufferReadptr);
           }
       }
       return 0;
   }
