
.. _program_listing_file_G__code_esp_code_freezer_main_freezerdrv_drv_at.c:

Program Listing for File drv_at.c
=================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_freezerdrv_drv_at.c>` (``G:\code\esp\code\freezer\main\freezerdrv\drv_at.c``)

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
   #include "driver/uart.h"
   #include "drv_loopuart.h"
   #include "drv_at.h"
   
   static const char *UARTTAG = "drvAt";
   static uartReceiveCallback atReceiveCb=NULL;
   static QueueHandle_t atUartQueue=NULL;
   static QueueHandle_t atDataQueue=NULL;
   static stQueueMsg    atQueuemsg;
   static char *atReceiveBuf=NULL;
   #define PATTERN_CHR_NUM    (3)         
   static stLoopBuffer drvLoopAt={0};
   int drvAtCanReadData(int start, int len)
   {
       return drvLoop_CanReadData(&drvLoopAt, start, len);
   }
   
   int drvAtReadData(char * data, int start, int len)
   {
       return drvLoop_ReadData(&drvLoopAt, data, start, len);
       // return 0;
   }
   
   int drvAtSendData(const char* data, int len)
   {
       static char *senddata = NULL;
       // senddata = realloc(senddata, len);
       // if(senddata == NULL)
       //     return -1;
       // memcpy(senddata, data, len);
   
       senddata = data;
       const int txBytes = uart_write_bytes(AT_UART_NUM, (const char*)senddata, len);
       // uart_wait_tx_done(AT_UART_NUM, 1 / portTICK_PERIOD_MS);
       // ESP_LOGI(UARTTAG, "drvAtSendData Wrote %d bytes [%s]", txBytes, senddata);
       // free(senddata);
       // senddata = NULL;
       return txBytes;
   }
   
   int drvSetAtBaudrate(uint32_t baudrate) 
   {
       if(baudrate > 921600)
           return -1;
       if (uart_set_baudrate(AT_UART_NUM, baudrate) != ESP_OK) {
           ESP_LOGI(UARTTAG, "drvSetAtBaudrate baudrate %d fail", baudrate);
           return -1;
       }
       return 0;
   }
   int drvAtInit(uartReceiveCallback recCb, QueueHandle_t* queue) 
   {
       if(recCb == NULL)
           return -1;
       const uart_config_t uart_config = {
           .baud_rate = 115200,
           .data_bits = UART_DATA_8_BITS,
           .parity = UART_PARITY_DISABLE,
           .stop_bits = UART_STOP_BITS_1,
           .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
           .source_clk = UART_SCLK_APB,
       };
       atReceiveCb = recCb;
       atDataQueue  = xQueueCreate(AT_DATA_QUEUE_SIZE, sizeof(atQueuemsg));
       if(atDataQueue == NULL)
           return -1;
       *queue      = atDataQueue;
       atReceiveBuf = (char*) realloc(atReceiveBuf, AT_RX_BUF_SIZE);
       if(atReceiveBuf == NULL)
           return -1;
       drvLoop_bufferInit(&drvLoopAt, atReceiveBuf, AT_RX_BUF_SIZE);
       // We won't use a buffer for sending data.
       // uart_driver_install(AT_UART_NUM, AT_RX_BUF_SIZE * 2, 0, 0, NULL, 0);
       uart_driver_install(AT_UART_NUM, AT_RX_BUF_SIZE / 2 * 3, AT_RX_BUF_SIZE / 2, AT_DATA_QUEUE_SIZE, &atUartQueue, 0);
       uart_param_config(AT_UART_NUM, &uart_config);
       uart_set_pin(AT_UART_NUM, AT_TXD_PIN, AT_RXD_PIN, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
       
       //Reset the pattern queue length to record at most 20 pattern positions.
       uart_pattern_queue_reset(AT_UART_NUM, 20);
   
       ESP_ERROR_CHECK(uart_set_rx_timeout(AT_UART_NUM, 3));
       uart_flush(AT_UART_NUM);
       return 0;
   }
   
   
   static void drvAtEventTask(void *pvParameters)
   {
       char* bufTemp=NULL;
       uart_event_t event;
       unsigned int receiveTimeout=0;
       bufTemp = (char*) realloc(bufTemp, 128 );//one pack 120
       if(bufTemp == NULL)
           return ;
       while (1)
       {
           //Waiting for UART event.
           // millis()
           if(receiveTimeout)
           {
               if(atDataQueue && ++receiveTimeout > AT_DATA_RECEIVE_TIMEOUT)
               {
                   ESP_LOGW(UARTTAG, "esp_get_free_heap_size[%d] timeoutsend", esp_get_free_heap_size());
                   xQueueSend(atDataQueue, &atQueuemsg, 10 / portTICK_PERIOD_MS);
                   atQueuemsg.len    = 0;
                   receiveTimeout    = 0;
               }
           }
           if(xQueueReceive(atUartQueue, (void * )&event, 20 / portTICK_PERIOD_MS)) //115200 8.3s/120B //921600 1.0146s/120B
           {
               // esp_task_wdt_reset();
               // ESP_LOGI(UARTTAG, "esp_get_free_heap_size[%d] event:%d receiveTimeout:%d", esp_get_free_heap_size(), event.type, receiveTimeout);
               // ESP_LOGI(UARTTAG, "uart[%d] event:", AT_UART_NUM);
               if(receiveTimeout)
                   receiveTimeout = 1;
               switch(event.type) 
               {
                   //Event of UART receving data
                   /*We'd better handler data event fast, there would be much more data events than
                   other types of events. If we take too much time on data event, the queue might
                   be full.*/
                   case UART_DATA:
                       // ESP_LOGI(UARTTAG, "[UART DATA] size:%d", event.size);
                       // uart_read_bytes(AT_UART_NUM, (const unsigned char*)g_receiveBuf, event.size, portMAX_DELAY);
                       // ESP_LOGI(UARTTAG, "[DATA EVT]:");
                       // uart_write_bytes(AT_UART_NUM, g_receiveBuf, event.size);
                       
                       if( drvLoop_CanWriteData(&drvLoopAt, event.size) == 0 )
                       {
                           uart_read_bytes(AT_UART_NUM, (unsigned char*)bufTemp, event.size, portMAX_DELAY);
                           // ESP_LOG_BUFFER_HEXDUMP(UARTTAG, bufTemp, event.size, ESP_LOG_INFO);
                           if(atQueuemsg.len == 0)
                           {
                               atQueuemsg.start  = drvLoopAt.BufferWriteptr;
                           }
                           receiveTimeout      = 1;
                           if(drvLoop_WriteData(&drvLoopAt, bufTemp, event.size) == 0)
                               atQueuemsg.len += event.size;
                       }
                       else    {
                           ESP_LOGW(UARTTAG, "[DATA EVT]: flush");
                           uart_flush(AT_UART_NUM);
                       }
                       
   
                       if(atQueuemsg.len && atDataQueue)
                       {
                           if(event.size < 120)//UART_FULL_THRESH_DEFAULT
                           {
                               xQueueSend(atDataQueue, &atQueuemsg, 10 / portTICK_PERIOD_MS);
                               atQueuemsg.len    = 0;
                               receiveTimeout    = 0;
                           }
                       }
                       break;
                   //Event of HW FIFO overflow detected
                   case UART_FIFO_OVF:
                       ESP_LOGI(UARTTAG, "hw fifo overflow");
                       // If fifo overflow happened, you should consider adding flow control for your application.
                       // The ISR has already reset the rx FIFO,
                       // As an example, we directly flush the rx buffer here in order to read more data.
                       uart_flush_input(AT_UART_NUM);
                       xQueueReset(atUartQueue);
                       break;
                   //Event of UART ring buffer full
                   case UART_BUFFER_FULL:
                       ESP_LOGI(UARTTAG, "ring buffer full");
                       // If buffer full happened, you should consider encreasing your buffer size
                       // As an example, we directly flush the rx buffer here in order to read more data.
                       uart_flush_input(AT_UART_NUM);
                       xQueueReset(atUartQueue);
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
                       // uart_get_buffered_data_len(AT_UART_NUM, &buffered_size);
                       // int pos = uart_pattern_pop_pos(AT_UART_NUM);
                       // ESP_LOGI(UARTTAG, "[UART PATTERN DETECTED] pos: %d, buffered size: %d", pos, buffered_size);
                       // if (pos == -1) {
                       //     // There used to be a UART_PATTERN_DET event, but the pattern position queue is full so that it can not
                       //     // record the position. We should set a larger queue size.
                       //     // As an example, we directly flush the rx buffer here.
                       //     uart_flush_input(AT_UART_NUM);
                       // } else {
                       //     uart_read_bytes(AT_UART_NUM, (const unsigned char*)g_receiveBuf, pos, 100 / portTICK_PERIOD_MS);
                       //     uint8_t pat[PATTERN_CHR_NUM + 1];
                       //     memset(pat, 0, sizeof(pat));
                       //     uart_read_bytes(AT_UART_NUM, pat, PATTERN_CHR_NUM, 100 / portTICK_PERIOD_MS);
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
   
   void drvAtProcess(void)
   {
       xTaskCreate(drvAtEventTask, "drvAtEventTask", 1024*3, NULL, configMAX_PRIORITIES, NULL);
       // xTaskCreate(rx_task, "uart_rx_task", 1024*2, NULL, configMAX_PRIORITIES, NULL);
       // xTaskCreate(tx_task, "uart_tx_task", 1024*2, NULL, configMAX_PRIORITIES-1, NULL);
   }
