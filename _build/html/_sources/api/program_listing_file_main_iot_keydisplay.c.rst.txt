
.. _program_listing_file_main_iot_keydisplay.c:

Program Listing for File keydisplay.c
=====================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_iot_keydisplay.c>` (``main\iot\keydisplay.c``)

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
   #include "freertos/queue.h"
   #include <stdio.h>
   #include <string.h>
   #include "esp_system.h"
   #include "esp_log.h"
   #include "drv_display.h"
   #include "iotinfo.h"
   #include "controlpro.h"
   #include "keydisplay.h"
   static const char *ATTAG = "keyDisplay";
   
   #define PROTOCOL_VERSION                            (0 << 6)
   #define PROTOCOL_VERSION_MASK                   (0xC0)
   #define PROTOCOL_HEADER                             (0xAA)
   
   static QueueHandle_t keyDisplayDataQueue=NULL;
   static char ProtocolSendBuf[32] = {PROTOCOL_HEADER, 0, 0};
   static char getDisplaykey(unsigned char key);
   static int keyDisplayUartPrase(char* recbuf, int len);
   
   typedef enum {
       KEY_NONE                      = 0,
       KEY_UP_SHORT                  = 0X02,
       KEY_DOWN_SHORT                = 0X07,
       KEY_SET_SHORT                 = 0X0C,
       KEY_SET_3S                    = 0X0D,
       KEY_UPSET_3S                  = 0X12,
       KEY_DOWNSET_3S                = 0X14,
       KEY_UPDOWN_8S                 = 0X16,
       KEY_UP_CONTINUE               = 0x21,
       KEY_DOWN_CONTINUE             = 0x22,
       KEY_HAVE_PRESS                = 0X23,
   } key_opt_t;
   
   #define FAST_COUNT_TIME          (1)
   
   #if (DISPLAY_DRIVER_WAY == DISPLAY_DRIVER_UART)
   static uint8_t ProtocolCrc(char *data, int len)
   {
       uint8_t crc=0,j;
       if(data == NULL)
           return 0;
       for(j=0; j<len; j++)
           crc += *(data + j);
       return crc;
   }
   uint8_t keyDisplayUartSend(uint8_t cmd, const char *data, uint8_t len)
   {
       uint8_t i=0,j;
       *(ProtocolSendBuf + i++) = PROTOCOL_HEADER;
       *(ProtocolSendBuf + i++) = PROTOCOL_VERSION | (3 + len);
       *(ProtocolSendBuf + i++) = cmd;
       if(data && len)
           for(j=0; j<len; j++)
               *(ProtocolSendBuf + i++) = *(data + j);
       *(ProtocolSendBuf + i++) = ProtocolCrc(ProtocolSendBuf+1, 2+len);
       return drvDisplaySendData(ProtocolSendBuf, i);
     return 0;
   }
   //AA 44 01 03 48 AA 44 01 03 48
   static void keyDisplayUartReceive(char* recbuf, int len)
   {
     int i=0, plen;
     // ESP_LOG_BUFFER_HEXDUMP(ATTAG, recbuf, len, ESP_LOG_INFO);
       for(i=0; i<len; i++)
       {
       if(*(recbuf+i) == PROTOCOL_HEADER)//
       {
             plen = (*(recbuf+i+1) & ~PROTOCOL_VERSION_MASK);
         if(i + plen + 1 > len || ProtocolCrc(recbuf+i+1, plen -1) != *(recbuf+i+plen))
           continue;
         // WashMode.DisplayVersion = (*(pdata+1) & KEY_PROTOCOL_VERSION_MASK) >> 6;
               keyDisplayUartPrase(recbuf+i + 2, plen - 2);
         i += plen;
       }
       else 
         continue;
       }
   }
   static int keyDisplayUartPrase(char* recbuf, int len)
   {
     // int i=0, j;
     // ESP_LOGI(ATTAG, "keyDisplayUartReceive Read %d bytes: '%s'  ", len, recbuf);
       if(recbuf == NULL || len == 0)
           return -1;
     // ESP_LOG_BUFFER_HEXDUMP(ATTAG, recbuf, len, ESP_LOG_INFO);
     // ESP_LOGI(ATTAG, "%s len [%d]", __FUNCTION__ , len);
     if(*recbuf == 0)  {  //respon 
       if(len == 3 && *(recbuf+1) == 1)  {
         getDisplaykey(*(recbuf+2) & 0x7f);
       }
     }
     else if(*recbuf == 1)  {  //display report key 
       getDisplaykey(*(recbuf+2) & 0x7f);
     }
     return 0;
   }
   static void keyDisplayProcess(void *arg)
   {
     char* keyBufTemp=NULL;
     stQueueMsg keyDisplayEvent;
     keyBufTemp = (char*) realloc(keyBufTemp, 128);
     if(keyBufTemp == NULL)  {
       ESP_LOGE(ATTAG, "keyDisplayPro xQueueReceive null");
       return;
     }
     while (1) 
     {
         if(xQueueReceive(keyDisplayDataQueue, (void * )&keyDisplayEvent, 10 / portTICK_PERIOD_MS) )
         {
           //  ESP_LOGI(ATTAG, "keyDisplayPro xQueueReceive begin %d\tbytes %d", keyDisplayEvent.start, keyDisplayEvent.len);
            if(drvDisplayCanReadData(keyDisplayEvent.start, keyDisplayEvent.len) == 0)
            {
               // portENTER_CRITICAL(); 
               // else  
               {
                 if(drvDisplayReadData(keyBufTemp, keyDisplayEvent.start, keyDisplayEvent.len) == 0) {
                   keyDisplayUartReceive(keyBufTemp, keyDisplayEvent.len);
                   keyBufTemp[keyDisplayEvent.len] = 0;
                 }
               }
               // portEXIT_CRITICAL();  
            }
         }
     }
     free(keyBufTemp);
     keyBufTemp = NULL;
     vTaskDelete(NULL);
   }
   void keyDisplayPro(void)
   {
     drvDisplayProcess();
     xTaskCreate(keyDisplayProcess, "keyDisplayProcess", 1024*3, NULL, configMAX_PRIORITIES, NULL);
   }
   
   #else
   
   static void keyDisplayUartReceive(char* recbuf, int len)
   {
   }
   uint8_t keyDisplayUartSend(uint8_t cmd, const char *data, uint8_t len)
   {
       uint8_t i=0,j;
       if(data && len) {
       ProtocolSendBuf[i++] = *(data + 1);
       ProtocolSendBuf[i++] = *(data + 2);
       // ProtocolSendBuf[i++] = *(data + 3);
       j = *(data + 4);
       j = ((j&0x04)<<5) | ((j&0x01)<<6);
       ProtocolSendBuf[i++] = j;
         return drvDisplaySendData(ProtocolSendBuf, i);
     }
     return 0;
   }
   void keyDisplayPro(void)
   {
     // drvDisplayProcess();
   }
   #endif
   
   
   static void operationSetTemp(signed char step)
   {
     if(displayStatus == DISPLAY_LOCKED)  {
       setTempDisplay(DISPLAY_TYPE_SET);
     }
     else if(displayStatus == DISPLAY_SET_TEMP)  {
       displayTemp = displayTemp + step;
       if(displayTemp >= -29 && displayTemp <= -14)  {
         if(displayTemp == -29)
           displayTemp  = -15;
         else if(displayTemp == -14)
           displayTemp  = -28;
         iotinfo_module.operateTime        = SET_OPERATION_TIME;
         resetDisplay();
         setTempDisplay(DISPLAY_TYPE_SET);
       }
     }
   }
   
   static void operationSetParam(signed char step)
   {
     if(displayStatus == DISPLAY_LOCKED)  {
       selectParamDisplay(DISPLAY_TYPE_SET);
     }
     else if(displayStatus >= DISPLAY_SELECT_PARAM_C1 && displayStatus <= DISPLAY_SELECT_PARAM_F1){
       if(step == -1 || step == 1) {
         if(step == 1) {
           if(++displayStatus > DISPLAY_SELECT_PARAM_F1)
             displayStatus = DISPLAY_SELECT_PARAM_C1;
         }
         else if(step == -1) {
           if(--displayStatus < DISPLAY_SELECT_PARAM_C1)
             displayStatus = DISPLAY_SELECT_PARAM_F1;
         }
         iotinfo_module.operateTime   = SET_OPERATION_TIME;
         resetDisplay();
       }
     }
     else if(displayStatus >= DISPLAY_SET_C1 && displayStatus <= DISPLAY_SET_F1){
       if(step == -2 || step == 2) {
         if(step == 2) { 
           if( (displayStatus == DISPLAY_SET_C1 || displayStatus == DISPLAY_SET_C2)
               && ++displayTemp > 999){
             displayTemp = 0;//999;
           }
           else if( (displayStatus == DISPLAY_SET_C3 || displayStatus == DISPLAY_SET_C4)
               && ++displayTemp > 99){
             displayTemp = 0;//999;
           }
           else if(displayStatus == DISPLAY_SET_C5 && ++displayTemp > 999){
             displayTemp = -999;
           }
           else if(displayStatus == DISPLAY_SET_F1 && ++displayTemp > 2){
             displayTemp = 0;//2;
           }
         }
         else if(step == -2) {
           if( (displayStatus == DISPLAY_SET_C1 || displayStatus == DISPLAY_SET_C2)
               && --displayTemp < 0){
             displayTemp = 999;
           }
           else if( (displayStatus == DISPLAY_SET_C3 || displayStatus == DISPLAY_SET_C4)
               && --displayTemp < 0){
             displayTemp = 99;
           }
           else if(displayStatus == DISPLAY_SET_C5 && --displayTemp < -999){
             displayTemp = 999;
           }
           else if(displayStatus == DISPLAY_SET_F1 && --displayTemp < 0){
             displayTemp = 2;
           }
         }
         iotinfo_module.operateTime   = SET_OPERATION_TIME;
         resetDisplay();
       }
     }
   }
   
   static void operationSetUp()
   {
     if(displayStatus == DISPLAY_SET_TEMP)
       operationSetTemp(1);
     else if(displayStatus >= DISPLAY_SELECT_PARAM_C1 && displayStatus <= DISPLAY_SELECT_PARAM_F1)
       operationSetParam(1);
     else if(displayStatus >= DISPLAY_SET_C1 && displayStatus <= DISPLAY_SET_F1)
       operationSetParam(2);
   }
   static void operationSetDown()
   {
     if(displayStatus == DISPLAY_SET_TEMP)
       operationSetTemp(-1);
     else if(displayStatus >= DISPLAY_SELECT_PARAM_C1 && displayStatus <= DISPLAY_SELECT_PARAM_F1)
       operationSetParam(-1);
     else if(displayStatus >= DISPLAY_SET_C1 && displayStatus <= DISPLAY_SET_F1)
       operationSetParam(-2);
   }
   static void operationSetQuit(void)
   {
     if(displayStatus == DISPLAY_SET_TEMP)  {
       setDisplay_0(displayTemp - iotinfo_module.devparam->settemp);
       settingQuit(DISPLAY_TYPE_SET);
     }
     else if(displayStatus >= DISPLAY_SELECT_PARAM_C1 && displayStatus <= DISPLAY_SELECT_PARAM_F1)
       settingQuit(DISPLAY_TYPE_SET);
     else if(displayStatus >= DISPLAY_SET_C1 && displayStatus <= DISPLAY_SET_F1)
       settingQuit(DISPLAY_TYPE_SET);
   }
   
   static char fastCount=0;
   static void operationSetFastUp(void)
   {
     if(++fastCount > FAST_COUNT_TIME) {
       fastCount = 0;
       operationSetUp();
     }
   }
   static void operationSetFastDown(void)
   {
     if(++fastCount > FAST_COUNT_TIME) {
       fastCount = 0;
       operationSetDown();
     }
   }
   static void operationSetPower(signed char step)
   {
     if(iotinfo_module.devparam->powerstatus == POWER_STATUS_ON) {//off
       powerOnControl(POWER_STATUS_LOCALOFF);
     }
     else if(iotinfo_module.devparam->powerstatus == POWER_STATUS_LOCALOFF) {//on
       powerOnControl(POWER_STATUS_ON);
     }
   }
   
   static void operationSetErr(signed char step)
   {
     // iotinfo_module.error.errbit = 0xffff;
     if(displayStatus == DISPLAY_LOCKED)  
       setErrDisplay(DISPLAY_TYPE_SET);
   }
   
   
   static char getDisplaykey(unsigned char key)
   {
     displaySetRekey(key);
     if(key == KEY_NONE)
       return -1;
   // extern void displayTest(void);
     // displayTest();
     ESP_LOGI(ATTAG, "%s key [0x%x] time:%d", __FUNCTION__ , key, iotinfo_module.operateTime);
     switch (key)
     {
       case KEY_HAVE_PRESS:
         fastCount = FAST_COUNT_TIME;
         if(displayStatus != DISPLAY_LOCKED)
           iotinfo_module.operateTime = SET_OPERATION_TIME;
         else
           displayTempAlarm(1);
         break;
       case KEY_UPDOWN_8S:
         operationSetPower(0);
         break;
       case KEY_SET_3S:
         if(displayStatus == DISPLAY_LOCKED)
           operationSetTemp(0);
         else if(displayStatus >= DISPLAY_SELECT_PARAM_C1 && displayStatus <= DISPLAY_SELECT_PARAM_F1)
           setParamDisplay(DISPLAY_TYPE_SET);
         break;
       case KEY_UPSET_3S:
         operationSetErr(0);
         break;
       case KEY_DOWNSET_3S:
         operationSetParam(0);
         break;
       
       case KEY_UP_SHORT:
         operationSetUp();
         break;
       case KEY_DOWN_SHORT:
         operationSetDown();
         break;
       case KEY_SET_SHORT:
         operationSetQuit();
         break;
       case KEY_UP_CONTINUE:
         operationSetFastUp();
         break;
       case KEY_DOWN_CONTINUE:
         operationSetFastDown();
         break;
         
       default:
         break;
     }  
     return 0;
   }
   void keyDisplayInit(void)
   {
     drvDisplayInit(keyDisplayUartReceive, &keyDisplayDataQueue);
   }
