
.. _program_listing_file_G__code_esp_code_freezer_main_iot_displaypro.c:

Program Listing for File displaypro.c
=====================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_iot_displaypro.c>` (``G:\code\esp\code\freezer\main\iot\displaypro.c``)

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
   #include "iotmain.h"
   #include "keydisplay.h"
   static const char *ATTAG = "displayPro";
   typedef struct {
     unsigned short  count;
     unsigned short  on;
     unsigned short  off;
   } display_param_t;
   static display_param_t  display_param;
   static display_buf_t    display_buf;
   static uint8_t s0Display = 0;
   unsigned char   displayStatus = DISPLAY_NONE;
   signed short    displayTemp = 0;
   
   const char numTable[16]={0x3F,0x06,0x5B,0x4F,0x66,0x6D,0x7D,0x07,0x7F,0x6F,
       0x77,DISPLAY_CHAR_b,0x58,DISPLAY_CHAR_d,DISPLAY_CHAR_E,DISPLAY_CHAR_F};
   #define ALARM_TIME_1S           (1*100 / DISPLAY_FLASH_500MS + 1)
   #define ALARM_TIME_1M           (60*100 / DISPLAY_FLASH_500MS + 1)
   #define DISPLAY_TIME_UP         (6*100 / DISPLAY_FLASH_500MS + 1)
   #define DISPLAY_TIME_DOWN       (3*100 / DISPLAY_FLASH_500MS + 1)
   #define DISPLAY_TIME_1_2        (6*100 / DISPLAY_FLASH_500MS + 1)
   #define DISPLAY_TIME_3_5        (8*100 / DISPLAY_FLASH_500MS + 1)
   #define DISPLAY_TIME_6_13       (12*100 / DISPLAY_FLASH_500MS + 1)
   #define DISPLAY_TEMP_MIN        (-320)
   
   static uint8_t keyDisplayRefresh(const char *data, int len)
   {
     return keyDisplayUartSend(1, data, len);
   }
   static char getErrorBit(char nowstatu)
   {
     if(iotinfo_module.error.errbit == 0 && nowstatu == DISPLAY_ERR_NONE)  {
         iotinfo_module.error.errnone = 1;
         return DISPLAY_ERR_NONE;
     }
     else if(nowstatu >= DISPLAY_ERR_NONE && nowstatu <= DISPLAY_ERR_SHOW_CSQ){
       iotinfo_module.error.errnone = 0;
       if(nowstatu < DISPLAY_ERR_NOSIM && iotinfo_module.error.errnosim)
         return DISPLAY_ERR_NOSIM;
       else if(nowstatu < DISPLAY_ERR_FROZEN && iotinfo_module.error.errfrozen)
         return DISPLAY_ERR_FROZEN;
       else if(nowstatu < DISPLAY_ERR_REFRIGERA && iotinfo_module.error.errrefrigera)
         return DISPLAY_ERR_REFRIGERA;
       else if(nowstatu < DISPLAY_ERR_REGISTER && iotinfo_module.error.errregister)
         return DISPLAY_ERR_REGISTER;
       else if(nowstatu < DISPLAY_ERR_ERRSIM && iotinfo_module.error.errsim)
         return DISPLAY_ERR_ERRSIM;
       else if(nowstatu < DISPLAY_ERR_CSQ && iotinfo_module.error.errcsq)
         return DISPLAY_ERR_CSQ;
       else if(nowstatu < DISPLAY_ERR_MCU && iotinfo_module.error.errmcu)
         return DISPLAY_ERR_MCU;
       else if(nowstatu < DISPLAY_ERR_WIFI && iotinfo_module.error.errwifi)
         return DISPLAY_ERR_WIFI;
       else if(nowstatu < DISPLAY_ERR_BLE && iotinfo_module.error.errble)
         return DISPLAY_ERR_BLE;
       else if(nowstatu < DISPLAY_ERR_SHOW_CSQ)
         return DISPLAY_ERR_SHOW_CSQ;
     }
     
     return 255;
   }
   static void setDigitalDisplay(signed char isnum, int dig)
   {
     display_buf.frozenred = 1;
     if(isnum < DISPLAY_TYPE_PARAM1) {
       if(dig < 0) {
         display_buf.signled = 1;
         dig = ~(dig - 1);
       }
       else  
         display_buf.signled = 0;
     }
   
     display_buf.point1    = 0;
     if(isnum == DISPLAY_TYPE_DIGITAL) {
       display_buf.digital0  = numTable[dig%100/10];
       display_buf.digital1  = numTable[dig%10];
       display_buf.digital2  = 0;
     }
     else if(isnum == DISPLAY_TYPE_DECIMAL) {
       display_buf.digital0  = numTable[dig/100];
       display_buf.digital1  = numTable[dig%100/10];
       display_buf.digital2  = numTable[dig%10];
       display_buf.point1    = 1;
     }
     else if(isnum == DISPLAY_TYPE_PARAM1) {
       display_buf.digital0  = (dig&0xff00)>>8;
       display_buf.digital1  = numTable[dig&0x0f];
       display_buf.digital2  = 0;
       display_buf.signled   = 0;
     }
     else if(isnum == DISPLAY_TYPE_CSQ) {
       display_buf.digital0  = DISPLAY_CHAR_C;
       dig = dig % 100;
       display_buf.digital1  = numTable[dig/10];
       display_buf.digital2  = numTable[dig%10];
       display_buf.signled   = 0;
     }
     else if(isnum == DISPLAY_TYPE_HHH) {
       display_buf.digital0  = DISPLAY_CHAR_H;
       display_buf.digital1  = DISPLAY_CHAR_H;
       display_buf.digital2  = DISPLAY_CHAR_H;
       display_buf.signled   = 0;
     }
     else if(isnum == DISPLAY_TYPE_LLL) {
       display_buf.digital0  = DISPLAY_CHAR_L;
       display_buf.digital1  = DISPLAY_CHAR_L;
       display_buf.digital2  = DISPLAY_CHAR_L;
       display_buf.signled   = 0;
     }
     else if(isnum == DISPLAY_TYPE_bd) {
       display_buf.digital0  = DISPLAY_CHAR_b;
       display_buf.digital1  = DISPLAY_CHAR_d;
       display_buf.digital2  = 0;
       display_buf.signled   = 0;
     }
     else if(isnum == DISPLAY_TYPE_OF) {
       display_buf.digital0  = numTable[0];
       display_buf.digital1  = DISPLAY_CHAR_F;
       display_buf.digital2  = 0;
       display_buf.signled   = 0;
     }
     else if(isnum == DISPLAY_TYPE_ERROR) {
       display_buf.digital0  = DISPLAY_CHAR_E;
       display_buf.digital1  = DISPLAY_CHAR_r;
       display_buf.digital2  = DISPLAY_CHAR_r;
       display_buf.signled   = 0;
     }
     else if(isnum == DISPLAY_TYPE_NONE) {
       display_buf.digital0  = 0;
       display_buf.digital1  = 0;
       display_buf.digital2  = 0;
       display_buf.signled   = 0;
     }
   }
   static void displayTempAlarm_0(unsigned short *alarmtime)//10ms
   {
     signed short distemp=0, settemp=0;
     settemp = iotinfo_module.devparam->settemp * 10;
     if(iotinfo_module.firstOn)  {
       if(iotinfo_module.showTemp <= settemp)  {
         iotinfo_module.firstOn      = 0;
         s0Display   = 0;
         return;
       }
       distemp = iotinfo_module.realTemp + iotinfo_module.devparam->tempoffset;//Tx=Tc+C5
       iotinfo_module.showTemp = distemp;
     }
     else if(s0Display)  {
       distemp = settemp + iotinfo_module.devparam->tempoffset + (iotinfo_module.realTemp - settemp) / 6;//Tx=Ts+C5+(1/6)*(Tc-Ts)
       if(iotinfo_module.tempDisTime == 0) {
         if(iotinfo_module.showTemp == settemp)  
           s0Display   = 0;
         else  {
           if(s0Display < 4)
             iotinfo_module.showTemp += 1;
           else
             iotinfo_module.showTemp -= 1;
   
           if(s0Display == 1 || s0Display == 4)
             iotinfo_module.tempDisTime = DISPLAY_TIME_1_2;
           else if(s0Display == 2 || s0Display == 5)
             iotinfo_module.tempDisTime = DISPLAY_TIME_3_5;
           else if(s0Display == 3 || s0Display == 6)
             iotinfo_module.tempDisTime = DISPLAY_TIME_6_13;
         }
       }
     }
     else  {
       distemp = settemp + iotinfo_module.devparam->tempoffset + (iotinfo_module.realTemp - settemp) / 6;//Tx=Ts+C5+(1/6)*(Tc-Ts)
       if(distemp != iotinfo_module.showTemp && iotinfo_module.tempDisTime == 0 ) {
         if(distemp > iotinfo_module.showTemp) {
           iotinfo_module.tempDisTime  = DISPLAY_TIME_UP;
           iotinfo_module.showTemp     +=  1;
         }
         else  {
           iotinfo_module.tempDisTime  = DISPLAY_TIME_DOWN;
           iotinfo_module.showTemp     -=  1;
         }
       }
     }
     
     ESP_LOGI(ATTAG, "firston:%d distemp:%d todistemp:%d settemp:%d realTemp:%d s0Display:%d tempDisTime:%d powerOnTime:%d", iotinfo_module.firstOn, iotinfo_module.showTemp, distemp, 
                 iotinfo_module.devparam->settemp, iotinfo_module.realTemp, s0Display, iotinfo_module.tempDisTime, iotinfo_module.powerOnTime);
     if( (distemp >= iotinfo_module.devparam->hightempalarm || distemp <= iotinfo_module.devparam->lowtempalarm) &&
         (iotinfo_module.firstOn == 0 || (iotinfo_module.firstOn && iotinfo_module.powerOnTime >= POWERON_6HOUR)) ) { //alarm 0
       if(distemp >= iotinfo_module.devparam->hightempalarm) {
         if( (*alarmtime && --*alarmtime == 0) || *alarmtime == 0) {
           if(display_param.count < display_param.on + 1)
             setDigitalDisplay(DISPLAY_TYPE_HHH, 0);
           else
             setDigitalDisplay(DISPLAY_TYPE_NONE, 0);
         }
       }
       else if(distemp <= iotinfo_module.devparam->lowtempalarm) {
         if( (*alarmtime && --*alarmtime == 0) || *alarmtime == 0) {
           if(display_param.count < display_param.on + 1)
             setDigitalDisplay(DISPLAY_TYPE_LLL, 0);
           else
             setDigitalDisplay(DISPLAY_TYPE_NONE, 0);
         }
       }
       else
         *alarmtime = ALARM_TIME_1S;
     }
     else {
       if(iotinfo_module.showTemp <= DISPLAY_TEMP_MIN)  
         iotinfo_module.showTemp = DISPLAY_TEMP_MIN;
       setDigitalDisplay(DISPLAY_TYPE_DECIMAL, iotinfo_module.showTemp);
       *alarmtime = ALARM_TIME_1S;
     }
     
   }
   static void displayTempAlarm_1(unsigned short *alarmtime)//10ms
   {
     signed short distemp=0, settemp=0;
     settemp = iotinfo_module.realTemp + iotinfo_module.devparam->tempoffset;
     
     if(iotinfo_module.realTemp >= 0)
       distemp = settemp;
     else if(iotinfo_module.realTemp < -400)
       distemp = settemp + 60;
     else  
       distemp = ((-iotinfo_module.realTemp) / 100 + 1) * 10 + settemp;
     
     
     if(distemp <= DISPLAY_TEMP_MIN)  
       iotinfo_module.showTemp = DISPLAY_TEMP_MIN;
     else if(distemp != iotinfo_module.showTemp && iotinfo_module.tempDisTime == 0){
       if(distemp > iotinfo_module.showTemp) {
         iotinfo_module.tempDisTime  = DISPLAY_TIME_UP;
         iotinfo_module.showTemp     +=  1;
       }
       else  {
         iotinfo_module.tempDisTime  = DISPLAY_TIME_DOWN;
         iotinfo_module.showTemp     -=  1;
       }
     }
     
     ESP_LOGI(ATTAG, "showway:%d distemp:%d todistemp:%d settemp:%d realTemp:%d s0Display:%d tempDisTime:%d powerOnTime:%d", iotinfo_module.devparam->showway, iotinfo_module.showTemp, distemp, 
                 iotinfo_module.devparam->settemp, iotinfo_module.realTemp, s0Display, iotinfo_module.tempDisTime, iotinfo_module.powerOnTime);
     if(  (distemp >= iotinfo_module.devparam->hightempalarm || distemp <= iotinfo_module.devparam->lowtempalarm) &&
           iotinfo_module.powerOnTime >= POWERON_6HOUR ) { //alarm 0
       if(distemp >= iotinfo_module.devparam->hightempalarm) {
         if( (*alarmtime && --*alarmtime == 0) || *alarmtime == 0) {
           if(display_param.count < display_param.on + 1)
             setDigitalDisplay(DISPLAY_TYPE_HHH, 0);
           else
             setDigitalDisplay(DISPLAY_TYPE_NONE, 0);
         }
       }
       else if(distemp <= iotinfo_module.devparam->lowtempalarm) {
         if( (*alarmtime && --*alarmtime == 0) || *alarmtime == 0) {
           if(display_param.count < display_param.on + 1)
             setDigitalDisplay(DISPLAY_TYPE_LLL, 0);
           else
             setDigitalDisplay(DISPLAY_TYPE_NONE, 0);
         }
       }
       else
         *alarmtime = ALARM_TIME_1M;
     }
     else
       setDigitalDisplay(DISPLAY_TYPE_DECIMAL, iotinfo_module.showTemp);
   }
   static void displayTempAlarm_2(unsigned short *alarmtime)//10ms
   {
     signed short alarmtemp=0, settemp=0;
     if(iotinfo_module.devparam->settemp * 10 <= DISPLAY_TEMP_MIN)  
       iotinfo_module.showTemp = DISPLAY_TEMP_MIN / 10;
     else
       iotinfo_module.showTemp = iotinfo_module.devparam->settemp;
     setDigitalDisplay(DISPLAY_TYPE_DIGITAL, iotinfo_module.showTemp);
   
     settemp = iotinfo_module.realTemp + iotinfo_module.devparam->tempoffset;
     
     if(iotinfo_module.realTemp >= 0)
       alarmtemp = settemp;
     else if(iotinfo_module.realTemp < -400)
       alarmtemp = settemp + 60;
     else  
       alarmtemp = ((-iotinfo_module.realTemp) / 100 + 1) * 10 + settemp;
   
     ESP_LOGI(ATTAG, "showway:%d alarmtemp:%d settemp:%d realTemp:%d tempDisTime:%d powerOnTime:%d high:%d low:%d", iotinfo_module.devparam->showway, alarmtemp, 
                 iotinfo_module.devparam->settemp, iotinfo_module.realTemp, iotinfo_module.tempDisTime, iotinfo_module.powerOnTime, iotinfo_module.devparam->hightempalarm, iotinfo_module.devparam->lowtempalarm);
     if( (alarmtemp >= iotinfo_module.devparam->hightempalarm || alarmtemp <= iotinfo_module.devparam->lowtempalarm) &&
         iotinfo_module.powerOnTime >= POWERON_6HOUR ) { //alarm 0
       if(alarmtemp >= iotinfo_module.devparam->hightempalarm) {
         if( (*alarmtime && --*alarmtime == 0) || *alarmtime == 0){
           if(display_param.count < display_param.on + 1)
             setDigitalDisplay(DISPLAY_TYPE_HHH, 0);
           else
             setDigitalDisplay(DISPLAY_TYPE_NONE, 0);
         }
       }
       else if(alarmtemp <= iotinfo_module.devparam->lowtempalarm) {
         if( (*alarmtime && --*alarmtime == 0) || *alarmtime == 0) {
           if(display_param.count < display_param.on + 1)
             setDigitalDisplay(DISPLAY_TYPE_LLL, 0);
           else
             setDigitalDisplay(DISPLAY_TYPE_NONE, 0);
         }
       }
       else
         *alarmtime = ALARM_TIME_1M;
     }
   }
   void displayTempAlarm(int settime)//500ms
   {
     static unsigned short alarmtime=ALARM_TIME_1S;
     if(settime > 0)   {
       // if()
       //   return;
       if(iotinfo_module.devparam->showway == 0) //show 0
         alarmtime=ALARM_TIME_1S;
       else if(iotinfo_module.devparam->showway == 1) //show 1
         alarmtime=ALARM_TIME_1M;
       else if(iotinfo_module.devparam->showway == 2) //show 2
         alarmtime=ALARM_TIME_1M;
     }
     else  {
       if(iotinfo_module.tempDisTime)
         iotinfo_module.tempDisTime--;
       if(iotinfo_module.devparam->showway == 0) //show 0
         displayTempAlarm_0(&alarmtime);
       else if(iotinfo_module.devparam->showway == 1) //show 1
         displayTempAlarm_1(&alarmtime);
       else if(iotinfo_module.devparam->showway == 2) //show 2
         displayTempAlarm_2(&alarmtime);
     }
   }
   static void lockedDisplay(signed char type)
   {
     if(type == DISPLAY_TYPE_SET) {//set display param
       displayStatus = DISPLAY_LOCKED;
       iotinfo_module.operateTime   = 0;
   
       display_param.on    = DISPLAY_FLASH_500MS;
       display_param.off   = DISPLAY_FLASH_500MS;
       display_param.count = 0;
     }
     else if(type == DISPLAY_TYPE_ON)  {//refresh display on
       displayTempAlarm(0);
     }
     else if(type == DISPLAY_TYPE_OFF)  {//refresh display off
       displayTempAlarm(0);
     }
     else  {
     }//
   }
   static void temp3SDisplay(signed char type)
   {
     if(type == DISPLAY_TYPE_SET) {//set display param
       displayStatus = DISPLAY_3STEMP;
       iotinfo_module.operateTime   = 0;
   
       display_param.on    = 3 * DISPLAY_FLASH_1S;
       display_param.off   = 0;
       display_param.count = 0;
     }
     else if(type == DISPLAY_TYPE_ON)  {//refresh display on
       setDigitalDisplay(DISPLAY_TYPE_DECIMAL, iotinfo_module.realTemp);
     }
     else if(type == DISPLAY_TYPE_ONOVER)  {//refresh display off
       lockedDisplay(DISPLAY_TYPE_SET);
     }
     else  {
     }//
   }
   void resetDisplay(void)
   {
     display_param.count = 0;
     return ;
   }
   int settingQuit(signed char type)
   {
     if(type == DISPLAY_TYPE_SET || (iotinfo_module.operateTime && --iotinfo_module.operateTime == 0) ) {
       ESP_LOGI(ATTAG, "%s time[%d], type[%d]", __FUNCTION__ , iotinfo_module.operateTime, type);
       if(displayStatus == DISPLAY_SET_TEMP)  {
         if(iotinfo_module.devparam->settemp != displayTemp)  {
           iotinfo_module.devparam->settemp = displayTemp;
           type = 100;
         }
         if(iotinfo_module.operateTime == 0 && iotinfo_module.powerOnTime < 1100U)
           temp3SDisplay(DISPLAY_TYPE_SET);
         else  
           lockedDisplay(DISPLAY_TYPE_SET);
       }
       else if(displayStatus >= DISPLAY_SET_C1 && displayStatus <= DISPLAY_SET_F1){
         iotinfo_module.operateTime   = SET_OPERATION_TIME;
         if(setParamInOut(DISPLAY_TYPE_OFF) == 0)
           type = 100;
         displayStatus = displayStatus - DISPLAY_SET_C1 + DISPLAY_SELECT_PARAM_C1;
       }
       else if( (displayStatus >= DISPLAY_SELECT_PARAM_C1 && displayStatus <= DISPLAY_SELECT_PARAM_F1) ||
                 displayStatus == DISPLAY_ERR_SHOW_CSQ ) {
         lockedDisplay(DISPLAY_TYPE_SET);
         return 0;
       }
   
       if(type == 100 && SaveInfoToFlash(NULL) <= 0)
         return -1;
       else
         return 0;
     }
     return -1;
   }
   int powerOffRemote(signed char type)
   {
     if(type == DISPLAY_TYPE_SET) {//set display param
       displayStatus = DISPLAY_REMOTEOFF;
   
       display_param.on    = DISPLAY_FLASH_500MS;
       display_param.off   = DISPLAY_FLASH_500MS;
       display_param.count = 0;
     }
     else if(type == DISPLAY_TYPE_ON)  {//refresh display on
       setDigitalDisplay(DISPLAY_TYPE_OF, 0);
     }
     else if(type == DISPLAY_TYPE_OFF)  {//refresh display off
     }
     else  {
     }//
     return -1;
   }
   int powerOffLocal(signed char type)
   {
     if(type == DISPLAY_TYPE_SET) {//set display param
       displayStatus = DISPLAY_LOCALOFF;
   
       display_param.on    = DISPLAY_FLASH_500MS;
       display_param.off   = DISPLAY_FLASH_500MS;
       display_param.count = 0;
     }
     else if(type == DISPLAY_TYPE_ON)  {//refresh display on
       setDigitalDisplay(DISPLAY_TYPE_bd, 0);
     }
     else if(type == DISPLAY_TYPE_OFF)  {//refresh display off
     }
     else  {
     }//
     return -1;
   }
   int powerOnDisplay(signed char type)
   {
     char *data = (char *)&display_buf;
     if(type == DISPLAY_TYPE_SET) {//set display param
       displayStatus = DISPLAY_POWERON;
   
       display_param.on    = DISPLAY_FLASH_1S;
       display_param.off   = DISPLAY_FLASH_1S;
       display_param.count = 0;
     }
     else if(type == DISPLAY_TYPE_ON)  {//refresh display on
       type = display_buf.reskey;
       for(int i=0; i<sizeof(display_buf_t); i++)
         *(data + i) = 0xff;
       display_buf.reskey = type;
     }
     else if(type == DISPLAY_TYPE_OFF)  {//refresh display off
       for(int i=0; i<sizeof(display_buf_t); i++)
         *(data + i) = 0;
     }
     else if(type == DISPLAY_TYPE_ONOVER)  {//refresh display off
       setTempDisplay(DISPLAY_TYPE_SET);
     }
     else  {
     }//
     return -1;
   }
   void setDisplay_0(signed char settemp)
   {
     char sign=0;
     if(iotinfo_module.firstOn == 0 && iotinfo_module.devparam->showway == 0)  {
       // settemp = settemp - iotinfo_module.devparam->settemp;
       if(settemp < 0) {
         settemp = ~(settemp-1);
         sign = 1;
       }
   
       if(1 <= settemp && 2 >= settemp){
         iotinfo_module.tempDisTime  = DISPLAY_TIME_1_2;
         s0Display   = 1;
       }
       else if(3 <= settemp && 5 >= settemp){
         iotinfo_module.tempDisTime = DISPLAY_TIME_3_5;
         s0Display   = 2;
       }
       else if(6 <= settemp && 13 >= settemp){
         iotinfo_module.tempDisTime = DISPLAY_TIME_6_13;
         s0Display   = 3;
       }
       else
         return;
       if(sign)
         s0Display += 3;
     }
   }
   int setTempDisplay(signed char type)
   {
     if(type == DISPLAY_TYPE_SET) {//set display param
       if(displayStatus == DISPLAY_LOCKED || displayStatus == DISPLAY_POWERON)
         displayStatus = DISPLAY_SET_TEMP;
       else
         return -1;
       iotinfo_module.operateTime   = SET_OPERATION_TIME;
       displayTemp = iotinfo_module.devparam->settemp;
   
       display_param.on    = DISPLAY_FLASH_500MS;
       display_param.off   = DISPLAY_FLASH_500MS;
       display_param.count = 0;
       return 0;
     }
     else if(type == DISPLAY_TYPE_ON)  {//refresh display on
       setDigitalDisplay(DISPLAY_TYPE_DIGITAL, displayTemp);//iotinfo_module.devparam->settemp);
     }
     else if(type == DISPLAY_TYPE_OFF)  {//refresh display off
       setDigitalDisplay(DISPLAY_TYPE_NONE, 0);
     }
     else  {
     }//
     return -1;
   }
   int selectParamDisplay(signed char type)
   {
     if(type == DISPLAY_TYPE_SET) {//set display param
       if(displayStatus == DISPLAY_LOCKED || displayStatus == DISPLAY_SELECT_PARAM_F1)
         displayStatus = DISPLAY_SELECT_PARAM_C1;
       else if(displayStatus >= DISPLAY_SELECT_PARAM_C1 && displayStatus < DISPLAY_SELECT_PARAM_F1)
         displayStatus++;
       else if(displayStatus >= DISPLAY_SET_C1 && displayStatus <= DISPLAY_SET_F1)
         displayStatus = displayStatus - DISPLAY_SET_C1 + DISPLAY_SELECT_PARAM_C1;
       else
         return -1;
   
       iotinfo_module.operateTime   = SET_OPERATION_TIME;
   
       display_param.on    = DISPLAY_FLASH_500MS;
       display_param.off   = DISPLAY_FLASH_500MS;
       display_param.count = 0;
       return 0;
     }
     else if(type == DISPLAY_TYPE_ON)  {//refresh display on
       signed short dig;
       if(displayStatus == DISPLAY_SELECT_PARAM_F1)  {
         dig = DISPLAY_CHAR_F;
         dig = (dig << 8) | 1;
       }
       else  {
         dig = DISPLAY_CHAR_C;
         dig = (dig << 8) | (displayStatus - DISPLAY_SELECT_PARAM_C1 + 1);
       }
       
       setDigitalDisplay(DISPLAY_TYPE_PARAM1, dig);
     }
     else if(type == DISPLAY_TYPE_OFF)  {//refresh display off
       setDigitalDisplay(DISPLAY_TYPE_NONE, 0);
     }
     else  {
     }//
     return -1;
   }
   
   int setParamInOut(signed char type)
   {
     if(type == DISPLAY_TYPE_ON) {
       if(displayStatus == DISPLAY_SET_C1)
         displayTemp = iotinfo_module.devparam->upbacklash;
       else if(displayStatus == DISPLAY_SET_C2)
         displayTemp = iotinfo_module.devparam->downbacklash;
       else if(displayStatus == DISPLAY_SET_C3)
         displayTemp = iotinfo_module.devparam->erroron;
       else if(displayStatus == DISPLAY_SET_C4)
         displayTemp = iotinfo_module.devparam->erroroff;
       else if(displayStatus == DISPLAY_SET_C5)
         displayTemp = iotinfo_module.devparam->tempoffset;
       else if(displayStatus == DISPLAY_SET_F1)
         displayTemp = iotinfo_module.devparam->showway;
     }
     else if(type == DISPLAY_TYPE_OFF) {
       if(displayStatus == DISPLAY_SET_C1) {
         if(iotinfo_module.devparam->upbacklash != displayTemp)  {
           iotinfo_module.devparam->upbacklash = displayTemp;
           return 0;
         }
       }
       else if(displayStatus == DISPLAY_SET_C2)  {
         if(iotinfo_module.devparam->downbacklash != displayTemp)  {
           iotinfo_module.devparam->downbacklash = displayTemp;
           return 0;
         }
       }
       else if(displayStatus == DISPLAY_SET_C3)  {
         if(iotinfo_module.devparam->erroron != displayTemp)  {
           iotinfo_module.devparam->erroron = displayTemp;
           return 0;
         }
       }
       else if(displayStatus == DISPLAY_SET_C4)  {
         if(iotinfo_module.devparam->erroroff != displayTemp)  {
           iotinfo_module.devparam->erroroff = displayTemp;
           return 0;
         }
       }
       else if(displayStatus == DISPLAY_SET_C5)  {
         if(iotinfo_module.devparam->tempoffset != displayTemp)  {
           iotinfo_module.devparam->tempoffset = displayTemp;
           return 0;
         }
       }
       else if(displayStatus == DISPLAY_SET_F1)  {
         if(iotinfo_module.devparam->showway != displayTemp)  {
           iotinfo_module.devparam->showway = displayTemp;
           return 0;
         }
       }
     }
     return -1;
   }
   int setParamDisplay(signed char type)
   {
     if(type == DISPLAY_TYPE_SET) {//set display param
       if(displayStatus >= DISPLAY_SELECT_PARAM_C1 && displayStatus <= DISPLAY_SELECT_PARAM_F1)
         displayStatus = displayStatus - DISPLAY_SELECT_PARAM_C1 + DISPLAY_SET_C1;
       else
         return -1;
   
       iotinfo_module.operateTime   = SET_OPERATION_TIME;
       setParamInOut(DISPLAY_TYPE_ON);
   
       display_param.on    = DISPLAY_FLASH_500MS;
       display_param.off   = DISPLAY_FLASH_500MS;
       display_param.count = 0;
       return 0;
     }
     else if(type == DISPLAY_TYPE_ON)  {//refresh display on
       // signed short dig=0;
       if(displayStatus == DISPLAY_SET_C1 || displayStatus == DISPLAY_SET_C2 || displayStatus == DISPLAY_SET_C5)  {
         // if(displayStatus == DISPLAY_SET_C1)
         //   dig = iotinfo_module.devparam->upbacklash;
         // else if(displayStatus == DISPLAY_SET_C2)
         //   dig = iotinfo_module.devparam->downbacklash;
         // else if(displayStatus == DISPLAY_SET_C5)
         //   dig = iotinfo_module.devparam->tempoffset;
         setDigitalDisplay(DISPLAY_TYPE_DECIMAL, displayTemp);
       }
       else  {
         // if(displayStatus == DISPLAY_SET_C3)
         //   dig = iotinfo_module.devparam->erroron;
         // else if(displayStatus == DISPLAY_SET_C4)
         //   dig = iotinfo_module.devparam->erroroff;
         // else if(displayStatus == DISPLAY_SET_F1)
         //   dig = iotinfo_module.devparam->showway;
         setDigitalDisplay(DISPLAY_TYPE_DIGITAL, displayTemp);
       }
     }
     else if(type == DISPLAY_TYPE_OFF)  {//refresh display off
       setDigitalDisplay(DISPLAY_TYPE_NONE, 0);
     }
     else  {
     }//
     return -1;
   }
   int setErrDisplay(signed char type)
   {
     char errsta;
     if(type == DISPLAY_TYPE_SET) {//set display param
       if(displayStatus == DISPLAY_LOCKED)  {
         errsta = getErrorBit(DISPLAY_ERR_NONE);
         if(errsta != 255 && errsta >= DISPLAY_ERR_NONE)
           displayStatus = errsta;
       }
       else
         return -1;
   
       iotinfo_module.operateTime   = SET_OPERATION_TIME * 2;
   
       display_param.on    = DISPLAY_FLASH_1S;
       display_param.off   = 0;
       display_param.count = 0;
       return 0;
     }
     else if(type == DISPLAY_TYPE_ON)  {//refresh display on
       signed short dig;
       if(displayStatus == DISPLAY_ERR_SHOW_CSQ)  {
         dig = iotinfo_module.at_module->csqRssi;
         setDigitalDisplay(DISPLAY_TYPE_CSQ, dig);
       }
       else  {
         dig = DISPLAY_CHAR_E;
         if(displayStatus >= DISPLAY_ERR_NONE && displayStatus <= DISPLAY_ERR_ERRSIM)
           dig = (dig << 8) | (displayStatus - DISPLAY_ERR_NONE);
         else if(displayStatus == DISPLAY_ERR_CSQ)
           dig = (dig << 8) | (7);
         else if(displayStatus >= DISPLAY_ERR_MCU && displayStatus <= DISPLAY_ERR_BLE)
           dig = (dig << 8) | (displayStatus - DISPLAY_ERR_MCU + 10);
         setDigitalDisplay(DISPLAY_TYPE_PARAM1, dig);
       }
       
     }
     // else if(type == DISPLAY_TYPE_OFF)  {//refresh display off
     // }
     else if(type == DISPLAY_TYPE_ONOVER)  {//refresh display on over
       if(displayStatus >= DISPLAY_ERR_NONE && displayStatus < DISPLAY_ERR_SHOW_CSQ)  {
         if(displayStatus == DISPLAY_ERR_NONE)
           displayStatus = DISPLAY_ERR_SHOW_CSQ;
         else  {
           errsta = getErrorBit(displayStatus);
           if(errsta >= DISPLAY_ERR_NONE && errsta != 255) 
             displayStatus = errsta;
         }
         display_param.on    = DISPLAY_FLASH_1S;
         display_param.off   = 0;
         display_param.count = 0;
       }//displayStatus
       else if(displayStatus == DISPLAY_ERR_SHOW_CSQ)  
         settingQuit(DISPLAY_TYPE_SET);
     }
     else  {
     }//
     return -1;
   }
   static int tempErrDisplay(signed char type)
   {
     if(iotinfo_module.error.errfrozen || iotinfo_module.error.errrefrigera){
       display_param.on    = DISPLAY_FLASH_500MS;
       display_param.off   = DISPLAY_FLASH_500MS;
     }
     if(type == DISPLAY_TYPE_ON){
       setDigitalDisplay(DISPLAY_TYPE_ERROR, 0);
     }
     else if(type == DISPLAY_TYPE_OFF){
       setDigitalDisplay(DISPLAY_TYPE_NONE, 0);
     }
     return -1;
   }
   
   int canFastDisplay(void)
   {
     if(display_param.on == DISPLAY_FLASH_500MS && display_param.off == DISPLAY_FLASH_500MS && display_param.count < 20)  {
       return 0;
     }
     return -1;
   }
   void otherDisplay(void)//10ms
   {
     if(displayStatus != DISPLAY_POWERON) {
       // compressor led
   int ret=0;
   extern int isCompressorDelay(void);
       ret = isCompressorDelay();
       if(ret == 1) {//delay
         if(display_param.on == DISPLAY_FLASH_500MS && display_param.off == DISPLAY_FLASH_500MS && display_param.count == 1)  
           display_buf.compressorled -= 1;
       }
       else if(ret == 2)//work on 
         display_buf.compressorled = 1;
       else
         display_buf.compressorled = 0;
     //4g led
       if(iotinfo_module.at_module->csqRssi >= 10) {
         if(iotinfo_module.iotStatus == IOT_CONNECTMQTT)
           display_buf.lteled = 1;
         else if(display_param.on == DISPLAY_FLASH_500MS && display_param.off == DISPLAY_FLASH_500MS)  {
           if(display_param.count < display_param.on + 1)
             display_buf.lteled = 1;
           else
             display_buf.lteled = 0;
         }
       }
       else
         display_buf.lteled = 0;
   
     //wifi led
       if(iotinfo_module.devinfo->wifinum == 0) {
         display_buf.wifiled = 0;
         // iotinfo_module.error.errwifi = 1;
       }
       else  {
         iotinfo_module.error.errwifi = 0;
         if(iotinfo_module.devinfo->wifinum < 4) {
           if(display_param.on == DISPLAY_FLASH_500MS && display_param.off == DISPLAY_FLASH_500MS)  {
             if(display_param.count < display_param.on + 1)
               display_buf.wifiled = 1;
             else
               display_buf.wifiled = 0;
           }
         }
         else
           display_buf.wifiled = 1;
       }
     }//DISPLAY_POWERON
   }
   
   
   void displayProcess(void)//10ms
   {
     static unsigned char distime=DISPLAY_REFRESH_TIME;
     char showway=DISPLAY_TYPE_NONE;
   #if 1
     settingQuit(DISPLAY_TYPE_NONE);
     otherDisplay();
     if(display_param.on || display_param.off) {
       if(++display_param.count == 1)  {
         if(display_param.on)
           showway = DISPLAY_TYPE_ON;
         else  {
           display_param.count = 0;
           showway = DISPLAY_TYPE_OFF;
         }
       }
       else if(display_param.count == display_param.on + 1)  {
         if(display_param.off)
           showway = DISPLAY_TYPE_OFF;
         else  {
           display_param.count = 0;
           showway = DISPLAY_TYPE_ONOVER;
         }
       }
       else if(display_param.count == display_param.on + display_param.off + 1)  {
         showway = DISPLAY_TYPE_ON;
         display_param.count = 0;
   
         if(displayStatus == DISPLAY_POWERON) 
           showway = DISPLAY_TYPE_ONOVER;
       }
   
       if( !(displayStatus >= DISPLAY_ERR_NONE && displayStatus <= DISPLAY_ERR_SHOW_CSQ) && 
           (iotinfo_module.error.errfrozen || iotinfo_module.error.errrefrigera) )
         tempErrDisplay(showway);
       else if(displayStatus)  {
         if(displayStatus == DISPLAY_POWERON)
           powerOnDisplay(showway);
         else if(displayStatus == DISPLAY_3STEMP)
           temp3SDisplay(showway);
         else if(displayStatus == DISPLAY_REMOTEOFF)
           powerOffRemote(showway);
         else if(displayStatus == DISPLAY_LOCALOFF)
           powerOffLocal(showway);
         else if(displayStatus == DISPLAY_SET_TEMP)
           setTempDisplay(showway);
         else if(displayStatus >= DISPLAY_SELECT_PARAM_C1 && displayStatus <= DISPLAY_SELECT_PARAM_F1)
           selectParamDisplay(showway);
         else if(displayStatus >= DISPLAY_SET_C1 && displayStatus <= DISPLAY_SET_F1)
           setParamDisplay(showway);
         else if(displayStatus >= DISPLAY_ERR_NONE && displayStatus <= DISPLAY_ERR_SHOW_CSQ)
           setErrDisplay(showway);
         else if(displayStatus == DISPLAY_LOCKED)
           lockedDisplay(showway);
       }
     }//ON DISPLAY
   #else
   extern void displayTest(void);
     displayTest();
   #endif
     
     if(distime && --distime == 0){
       distime = DISPLAY_REFRESH_TIME;
       if(displayStatus)
         keyDisplayRefresh((const char *)&display_buf, sizeof(display_buf_t));
       // else
       //   keyDisplayRefresh(NULL, sizeof(display_buf_t));
       // ESP_LOGI(ATTAG,"ontime:%d displayStatus:%d count:%d on:%d off:%d " , iotinfo_module.powerOnTime, displayStatus, display_param.count, display_param.on, display_param.off);
     }
   }
   void displaySetRekey(unsigned char key)
   {
     display_buf.reskey = key;
   }
   void displayInit(void)
   {
     display_buf.reskey = 0;
     powerOnDisplay(DISPLAY_TYPE_SET);
   }
   
   #if 0
   void displayTest(void)//10ms
   {
     static unsigned char test=0, pers=0;
     if(++pers < 200)
       return;
     pers = 0;
     if(++test == 1)
       display_buf.frozenred = 1;
     else if(test == 2)
       display_buf.frozengreen = 1;
     else if(test == 3)
       display_buf.compressorled = 1;
     else if(test == 4)
       display_buf.wifiled = 1;
     else if(test == 5)
       display_buf.signled = 1;
     else if(test == 6)
       display_buf.lteled = 1;
     else if(test == 7)
       display_buf.digital0 = numTable[2];
     else if(test == 8)
       display_buf.digital0 = numTable[4];
     else if(test == 9)
       display_buf.digital0 = numTable[6];
     else if(test == 10)
       display_buf.digital1 = numTable[8];
     else if(test == 11)
       display_buf.digital1 = numTable[1];
     else if(test == 12)
       display_buf.digital1 = numTable[3];
     else if(test == 13)
       display_buf.digital1 = numTable[5];
     else if(test == 14)
       display_buf.digital2 = numTable[7];
     else if(test == 15)
       display_buf.digital2 = numTable[9];
     else if(test == 16)
       display_buf.digital2 = numTable[0];
     else if(test == 17)
       display_buf.point0 = 1;
     else if(test == 18)
       display_buf.point1 = 1;
     else if(test == 19)
       display_buf.point2 = 1;
   
     ESP_LOGI(ATTAG,"test step:%d " , test);
   }
   #endif
       
