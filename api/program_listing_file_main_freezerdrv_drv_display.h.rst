
.. _program_listing_file_main_freezerdrv_drv_display.h:

Program Listing for File drv_display.h
======================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_freezerdrv_drv_display.h>` (``main\freezerdrv\drv_display.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __DRV_DISPLAY__
   #define __DRV_DISPLAY__
   
   
   #define DISPLAY_DRIVER_1650         (1)
   #define DISPLAY_DRIVER_UART         (2)
   #define DISPLAY_DRIVER_WAY          (DISPLAY_DRIVER_UART)
   
   #ifndef DISPLAY_DRIVER_WAY
       #error "must have to defined display driver way, 1650 or uart"
   #else
       #if (DISPLAY_DRIVER_WAY != DISPLAY_DRIVER_UART && DISPLAY_DRIVER_WAY != DISPLAY_DRIVER_1650)
           #error "display driver is not allow, pls select 1650 or uart"
       #endif
   #endif
   
   #if (DISPLAY_DRIVER_WAY == DISPLAY_DRIVER_UART)
   #define DISPLAY_UART_NUM (UART_NUM_1)
   #define DISPLAY_TXD_PIN (4)//UART_NUM_2 GPIO_NUM_17
   #define DISPLAY_RXD_PIN (5)
   // #define DISPLAY_UART_NUM (UART_NUM_2)
   // #define DISPLAY_TXD_PIN (17)//UART_NUM_2 GPIO_NUM_17
   // #define DISPLAY_RXD_PIN (16)
   #define DISPLAY_RX_BUF_SIZE             (130)//larger than 128
   #define DISPLAY_DATA_RECEIVE_TIMEOUT    (1)
   
   #elif (DISPLAY_DRIVER_WAY == DISPLAY_DRIVER_1650)
   #define DISPLAY_RX_BUF_SIZE             (32)//
   #endif
   
   #define DISPLAY_DATA_QUEUE_SIZE         (10)
   
   typedef void * QueueHandle_t;
   typedef void (*uartReceiveCallback)( char* recbuf, int len);
   int drvDisplayInit(uartReceiveCallback recCb, QueueHandle_t* queue) ;
   void drvDisplayProcess(void);
   int drvDisplaySendData(const char* data, int len);
   //use in queue
   int drvDisplayReadData(char * data, int start, int len);
   int drvDisplayCanReadData(int start, int len);
   
   typedef struct   
   { 
       int start;
       int len;
   }stQueueMsg;
   #endif
   
