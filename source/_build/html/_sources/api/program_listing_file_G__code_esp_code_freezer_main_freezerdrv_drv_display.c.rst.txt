
.. _program_listing_file_G__code_esp_code_freezer_main_freezerdrv_drv_display.c:

Program Listing for File drv_display.c
======================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_freezerdrv_drv_display.c>` (``G:\code\esp\code\freezer\main\freezerdrv\drv_display.c``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /* UART asynchronous example, that uses separate RX and TX tasks
   
      This example code is in the Public Domain (or CC0 licensed, at your option.)
   
      Unless required by applicable law or agreed to in writing, this
      software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
      CONDITIONS OF ANY KIND, either express or implied.
   */
   #include "freertos/FreeRTOS.h"
   #include "freertos/task.h"
   #include "freertos/queue.h"
   #include "esp_task_wdt.h"
   #include "esp_system.h"
   #include "esp_log.h"
   #include "string.h"
   #include "drv_loopuart.h"
   #include "drv_display.h"
   
   static const char *UARTTAG = "drvDisplay";
   static uartReceiveCallback displayReceiveCb=NULL;
   static QueueHandle_t displayUartQueue=NULL;
   static QueueHandle_t displayDataQueue=NULL;
   static stQueueMsg    displayQueuemsg;
   static char *displayReceiveBuf=NULL;
   static stLoopBuffer drvLoopDisplay={0};
   
   #if (DISPLAY_DRIVER_WAY == DISPLAY_DRIVER_UART)
   #include "driver/uart.h"
   #define PATTERN_CHR_NUM    (3)         
   int drvDisplayCanReadData(int start, int len)
   {
       return drvLoop_CanReadData(&drvLoopDisplay, start, len);
   }
   
   int drvDisplayReadData(char * data, int start, int len)
   {
       return drvLoop_ReadData(&drvLoopDisplay, data, start, len);
       // return 0;
   }
   
   int drvDisplaySendData(const char* data, int len)
   {
       const int txBytes = uart_write_bytes(DISPLAY_UART_NUM, (const char*)data, len);
       // ESP_LOGI(UARTTAG, "drvDisplaySendData Wrote %d bytes", txBytes);
       // ESP_LOG_BUFFER_HEXDUMP(UARTTAG, (const char*)data, len, ESP_LOG_INFO);
       return txBytes;
   }
   
   int drvDisplayInit(uartReceiveCallback recCb, QueueHandle_t* queue) 
   {
       if(recCb == NULL)
           return -1;
       const uart_config_t uart_config = {
           .baud_rate = 9600,
           .data_bits = UART_DATA_8_BITS,
           .parity = UART_PARITY_DISABLE,
           .stop_bits = UART_STOP_BITS_1,
           .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
           .source_clk = UART_SCLK_APB,
       };
       displayReceiveCb = recCb;
       displayDataQueue  = xQueueCreate(DISPLAY_DATA_QUEUE_SIZE, sizeof(displayQueuemsg));
       if(displayDataQueue == NULL)
           return -1;
       *queue      = displayDataQueue;
       displayReceiveBuf = (char*) realloc(displayReceiveBuf, DISPLAY_RX_BUF_SIZE);
       if(displayReceiveBuf == NULL)
           return -1;
       drvLoop_bufferInit(&drvLoopDisplay, displayReceiveBuf, DISPLAY_RX_BUF_SIZE);
       // We won't use a buffer for sending data.
       uart_driver_install(DISPLAY_UART_NUM, DISPLAY_RX_BUF_SIZE, DISPLAY_RX_BUF_SIZE, 5, &displayUartQueue, 0);
       uart_param_config(DISPLAY_UART_NUM, &uart_config);
       uart_set_pin(DISPLAY_UART_NUM, DISPLAY_TXD_PIN, DISPLAY_RXD_PIN, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
       //Reset the pattern queue length to record at most 20 pattern positions.
       uart_pattern_queue_reset(DISPLAY_UART_NUM, 20);
   
       ESP_ERROR_CHECK(uart_set_rx_timeout(DISPLAY_UART_NUM, 3));
       uart_flush(DISPLAY_UART_NUM);
       return 0;
   }
   
   
   static void drvDisplayEventTask(void *pvParameters)
   {
       char* bufTemp=NULL;
       uart_event_t event;
       unsigned int receiveTimeout=0;
       bufTemp = (char*) realloc(bufTemp, 128 );//one pack 120
       if(bufTemp == NULL)
           return;
       while (1)
       {
           //Waiting for UART event.
           // millis()
           if(receiveTimeout)
           {
               if(++receiveTimeout >= DISPLAY_DATA_RECEIVE_TIMEOUT && displayDataQueue)
               {
                   xQueueSend(displayDataQueue, &displayQueuemsg, 10 / portTICK_PERIOD_MS);
                   displayQueuemsg.len    = 0;
                   receiveTimeout      = 0;
               }
           }
           if(xQueueReceive(displayUartQueue, (void * )&event, 200 / portTICK_PERIOD_MS)) 
           {
               // esp_task_wdt_reset();
               // ESP_LOGI(UARTTAG, "esp_get_free_heap_size[%d] event:", esp_get_free_heap_size());
               // ESP_LOGI(UARTTAG, "esp_get_minimum_free_heap_size()[%d] event:", esp_get_minimum_free_heap_size());
               // ESP_LOGI(UARTTAG, "uart[%d] event:", DISPLAY_UART_NUM);
               if(receiveTimeout)
                   receiveTimeout = 1;
               switch(event.type) 
               {
                   //Event of UART receving data
                   /*We'd better handler data event fast, there would be much more data events than
                   other types of events. If we take too much time on data event, the queue might
                   be full.*/
                   case UART_DATA:
                       // ESP_LOGI(UARTTAG, "[UART DATA]: %d", event.size);
                       // uart_read_bytes(DISPLAY_UART_NUM, (const unsigned char*)g_receiveBuf, event.size, portMAX_DELAY);
                       // ESP_LOGI(UARTTAG, "[DATA EVT]:");
                       // uart_write_bytes(DISPLAY_UART_NUM, g_receiveBuf, event.size);
                       
                       if( drvLoop_CanWriteData(&drvLoopDisplay, event.size) == 0 )
                       {
                           uart_read_bytes(DISPLAY_UART_NUM, (const unsigned char*)bufTemp, event.size, portMAX_DELAY);
                           // ESP_LOG_BUFFER_HEXDUMP(UARTTAG, bufTemp, event.size, ESP_LOG_INFO);
                           if(displayQueuemsg.len == 0)
                           {
                               receiveTimeout      = 1;
                               displayQueuemsg.start  = drvLoopDisplay.BufferWriteptr;
                           }
                           if(drvLoop_WriteData(&drvLoopDisplay, bufTemp, event.size) == 0)
                               displayQueuemsg.len += event.size;
                       }
                       else
                           uart_flush(DISPLAY_UART_NUM);
                       
   
                       if(displayQueuemsg.len && displayDataQueue)
                       {
                           if(event.size != 120)//UART_FULL_THRESH_DEFAULT
                           {
                               xQueueSend(displayDataQueue, &displayQueuemsg, 10 / portTICK_PERIOD_MS);
                               displayQueuemsg.len    = 0;
                               receiveTimeout      = 0;
                           }
                       }
                       break;
                   //Event of HW FIFO overflow detected
                   case UART_FIFO_OVF:
                       ESP_LOGI(UARTTAG, "hw fifo overflow");
                       // If fifo overflow happened, you should consider adding flow control for your application.
                       // The ISR has already reset the rx FIFO,
                       // As an example, we directly flush the rx buffer here in order to read more data.
                       uart_flush_input(DISPLAY_UART_NUM);
                       xQueueReset(displayUartQueue);
                       break;
                   //Event of UART ring buffer full
                   case UART_BUFFER_FULL:
                       ESP_LOGI(UARTTAG, "ring buffer full");
                       // If buffer full happened, you should consider encreasing your buffer size
                       // As an example, we directly flush the rx buffer here in order to read more data.
                       uart_flush_input(DISPLAY_UART_NUM);
                       xQueueReset(displayUartQueue);
                       break;
                   //Event of UART RX break detected
                   case UART_BREAK:
                       ESP_LOGI(UARTTAG, "uart rx break");
                       break;
                   //Event of UART parity check error
                   case UART_PARITY_ERR:
                       ESP_LOGI(UARTTAG, "uart parity error");
                       break;
                   //Event of UART frame error
                   case UART_FRAME_ERR:
                       ESP_LOGI(UARTTAG, "uart frame error");
                       break;
                   //UART_PATTERN_DET
                   case UART_PATTERN_DET:
                       // uart_get_buffered_data_len(DISPLAY_UART_NUM, &buffered_size);
                       // int pos = uart_pattern_pop_pos(DISPLAY_UART_NUM);
                       // ESP_LOGI(UARTTAG, "[UART PATTERN DETECTED] pos: %d, buffered size: %d", pos, buffered_size);
                       // if (pos == -1) {
                       //     // There used to be a UART_PATTERN_DET event, but the pattern position queue is full so that it can not
                       //     // record the position. We should set a larger queue size.
                       //     // As an example, we directly flush the rx buffer here.
                       //     uart_flush_input(DISPLAY_UART_NUM);
                       // } else {
                       //     uart_read_bytes(DISPLAY_UART_NUM, (const unsigned char*)g_receiveBuf, pos, 100 / portTICK_PERIOD_MS);
                       //     uint8_t pat[PATTERN_CHR_NUM + 1];
                       //     memset(pat, 0, sizeof(pat));
                       //     uart_read_bytes(DISPLAY_UART_NUM, pat, PATTERN_CHR_NUM, 100 / portTICK_PERIOD_MS);
                       //     ESP_LOGI(UARTTAG, "read data: %s", g_receiveBuf);
                       //     ESP_LOGI(UARTTAG, "read pat : %s", pat);
                       // }
                       break;
                   //Others
                   default:
                       ESP_LOGI(UARTTAG, "uart event type: %d", event.type);
                       break;
               }
           }
           // vTaskDelay(1 / portTICK_PERIOD_MS);
       }
       free(bufTemp);
       bufTemp = NULL;
       vTaskDelete(NULL);
   }
   
   void drvDisplayProcess(void)
   {
       xTaskCreate(drvDisplayEventTask, "drvDisplayEventTask", 1024*3, NULL, configMAX_PRIORITIES, NULL);
   }
   #elif (DISPLAY_DRIVER_WAY == DISPLAY_DRIVER_1650)
   #include "drv_1650.h"
   int drvDisplayCanReadData(int start, int len)
   {
       return drvLoop_CanReadData(&drvLoopDisplay, start, len);
   }
   
   int drvDisplayReadData(char * data, int start, int len)
   {
       return drvLoop_ReadData(&drvLoopDisplay, data, start, len);
       // return 0;
   }
   
   static int drvDisplayGetKey(void)
   {
       unsigned char key = 0;
       tm1650ReadKey(&key);
       if(key != 0x07)
           ESP_LOGI(UARTTAG, "drvDisplaySendData key 0x%d", key);
       return key;
   }
   int drvDisplaySendData(const char* data, int len)
   {
       drvDisplayGetKey();
       int txBytes = tm1650SetDispalyData((uint8_t*)data, len);
       // ESP_LOG_BUFFER_HEXDUMP(UARTTAG, (const char*)data, len, ESP_LOG_INFO);
       return txBytes;
   }
   int drvDisplayInit(uartReceiveCallback recCb, QueueHandle_t* queue) 
   {
       if(recCb == NULL)
           return -1;
       displayDataQueue  = xQueueCreate(DISPLAY_DATA_QUEUE_SIZE, sizeof(displayQueuemsg));
       if(displayDataQueue == NULL)
           return -1;
       *queue      = displayDataQueue;
       displayReceiveBuf = (char*) realloc(displayReceiveBuf, DISPLAY_RX_BUF_SIZE);
       if(displayReceiveBuf == NULL)
           return -1;
       drvLoop_bufferInit(&drvLoopDisplay, displayReceiveBuf, DISPLAY_RX_BUF_SIZE);
       drvTm1650Init();
       tm1650SetDispaly( TM1650_DISPLAY_VALUE(LEVEL_1, TM1650_SEGMENT_8, TM1650_DISPLAY_ON) );
       return 0;
   }
   // static void drvDisplayEventTask(void *pvParameters)
   // {
   //     char* bufTemp=NULL;
   //     // uart_event_t event;
   //     unsigned int receiveTimeout=0;
   //     while (1)
   //     {
   //         vTaskDelay(1000 / portTICK_PERIOD_MS);
   //     }
   //     vTaskDelete(NULL);
   // }
   void drvDisplayProcess(void)
   {
       // xTaskCreate(drvDisplayEventTask, "drvDisplayEventTask", 1024*3, NULL, configMAX_PRIORITIES, NULL);
   }
   #endif
   
