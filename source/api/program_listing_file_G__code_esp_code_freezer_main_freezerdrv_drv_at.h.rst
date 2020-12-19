
.. _program_listing_file_G__code_esp_code_freezer_main_freezerdrv_drv_at.h:

Program Listing for File drv_at.h
=================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_freezerdrv_drv_at.h>` (``G:\code\esp\code\freezer\main\freezerdrv\drv_at.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __DRV_AT__
   #define __DRV_AT__
   
   // #define AT_UART_NUM (UART_NUM_1)
   // #define AT_TXD_PIN (4)//UART_NUM_2 GPIO_NUM_17
   // #define AT_RXD_PIN (5)
   #define AT_UART_NUM (UART_NUM_2)
   #define AT_TXD_PIN (17)//UART_NUM_2 GPIO_NUM_17
   #define AT_RXD_PIN (16)
   #define AT_RX_BUF_SIZE          (2048/2*3)
   #define AT_DATA_QUEUE_SIZE      (20)
   #define AT_DATA_RECEIVE_TIMEOUT (3)
   
   typedef void * QueueHandle_t;
   typedef void (*uartReceiveCallback)( char* recbuf, int len);
   int drvAtInit(uartReceiveCallback recCb, QueueHandle_t* queue) ;
   void drvAtProcess(void);
   int drvSetAtBaudrate(uint32_t baudrate) ;
   int drvAtSendData(const char* data, int len);
   //use in queue
   int drvAtReadData(char * data, int start, int len);
   int drvAtCanReadData(int start, int len);
   
   typedef struct   
   { 
       int start;
       int len;
   }stQueueMsg;
   
   #endif
   
