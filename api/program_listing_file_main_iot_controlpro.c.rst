
.. _program_listing_file_main_iot_controlpro.c:

Program Listing for File controlpro.c
=====================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_iot_controlpro.c>` (``main\iot\controlpro.c``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /* ADC1 Example
   
      This example code is in the Public Domain (or CC0 licensed, at your option.)
   
      Unless required by applicable law or agreed to in writing, this
      software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
      CONDITIONS OF ANY KIND, either express or implied.
   */
   #include <stdio.h>
   #include <string.h>
   #include "esp_log.h"
   #include "drv_adc.h"
   #include "drv_gpio.h"
   #include "iotinfo.h"
   #include "controlpro.h"
   #include "keydisplay.h"
   //3134mv 1100
   //3209mv 1200
   //3246mv 1250
   //3283mv 1300
   //3320mv 1350
   static const char *ATTAG = "controlPro";
   static int compressorTime=-1;
   static int firstCloseTime=0;
   static unsigned char controlFlag=(1 << SET_FIRST_TSG);
   extern uint32_t esp_get_free_heap_size( void );
   
   static void setControlFlagBit(char setflag) 
   {
      if(setflag <= SET_REMOTE_OFF) {
         if(setflag == SET_REMOTE_OFF) {
            compressorTime = -1;
            SET_COMPRESSOR_OUTPUT(OUTPUT_LOW);
            iotinfo_module.devparam->powerstatus  = POWER_STATUS_REMOTEOFF;
         }
         controlFlag = controlFlag | (1 << setflag);
      }
      else if(setflag <= RESET_REMOTE_OFF){
         setflag = setflag - 8;
         controlFlag = controlFlag & ~(1 << setflag);
         iotinfo_module.devparam->powerstatus  = POWER_STATUS_ON;
      }
      else if(setflag == SET_ERROR_ON){
         if (compressorTime >= 0)   {
            compressorTime = -1;
            SET_COMPRESSOR_OUTPUT(OUTPUT_LOW);
         }
      }
     ESP_LOGI(ATTAG, "setControlFlagBit 0x%02X  ", controlFlag);
   }
   static void compressorControl(void) //10ms
   {
      //tsk = ts - 4 + c1 = settemp + c1//upbacklash
      //tsg = ts - 4 - c2 = settemp - c2//downbacklash
      static unsigned char persec = 0;
      unsigned char flag=controlFlag & (1 << SET_FIRST_TSG);
      int settemp = 0;
      if( iotinfo_module.devparam->powerstatus == POWER_STATUS_ON && (controlFlag >> SET_REMOTE_OFF == 0) )   {
         if(iotinfo_module.error.errrefrigera)  {//error on/off
            if(compressorTime > 0)  {
               settemp = 60U * 100 * iotinfo_module.devparam->erroron;
               if(compressorTime >= settemp) {
                  compressorTime = -1;
                  SET_COMPRESSOR_OUTPUT(OUTPUT_LOW);
               }
               else
                  SET_COMPRESSOR_OUTPUT(OUTPUT_HIGH);
            }
            else if(compressorTime < 0)  {
               settemp = 60U * 100 * iotinfo_module.devparam->erroroff;
               if(-compressorTime >= settemp) {
                  setControlFlagBit(RESET_WORK_TIMEOUT);
                  compressorTime = 1;
                  SET_COMPRESSOR_OUTPUT(OUTPUT_HIGH);
               }
               SET_COMPRESSOR_OUTPUT(OUTPUT_LOW);
            }
         }
         else  {//normal on
            settemp = iotinfo_module.devparam->settemp * 10 - 40;
            if( (compressorTime < 0) && ((iotinfo_module.powerOnTime < 2*FIRSTON_TIME && iotinfo_module.powerOnTime > FIRSTON_TIME && iotinfo_module.realTemp > settemp - iotinfo_module.devparam->downbacklash) ||
                  (compressorTime < -CLOSE_TIME_MINI && iotinfo_module.realTemp > settemp + iotinfo_module.devparam->upbacklash)) ){
               //real > TSG //real > TSK && offtime > lowstop  //open
               flag = controlFlag & (1 << SET_WORK_TIMEOUT);
               if(flag) {
                  if(compressorTime < -CLOSE_TIME_CONTINUE)
                  flag = 0;
               }
               if(flag == 0)  {
                  compressorTime = 1;
                  setControlFlagBit(RESET_WORK_TIMEOUT);
                  SET_COMPRESSOR_OUTPUT(OUTPUT_HIGH);
               }
            }
            else if( compressorTime >= 0 && (compressorTime >= POWERON_6HOUR || 
                           (iotinfo_module.powerOnTime > FIRSTON_TIME && iotinfo_module.realTemp < settemp - iotinfo_module.devparam->downbacklash) ) ){
               //real < TSG //offtime > 6h  //close
               flag = controlFlag & (1 << SET_FIRST_TSG);
               if(flag){
                  if (firstCloseTime == 0)
                     firstCloseTime = compressorTime + CLOSE_TIME_FIRSTTSG;
                  else if (compressorTime >= firstCloseTime)
                     firstCloseTime = 0;
               }
               else
                  firstCloseTime = 0;
               if(firstCloseTime == 0){
                  setControlFlagBit(RESET_FIRST_TSG);
                  if(compressorTime >= POWERON_6HOUR)
                     setControlFlagBit(SET_WORK_TIMEOUT);
                  compressorTime = -1;
                  SET_COMPRESSOR_OUTPUT(OUTPUT_LOW);
               }//firstCloseTime
            }
         }
      }
      else
         SET_COMPRESSOR_OUTPUT(OUTPUT_LOW);
   
      if(compressorTime < 0){//close time
         firstCloseTime = 0;
         if(--compressorTime < -POWERON_6HOUR)
            compressorTime = -POWERON_6HOUR;
      }
      else if(compressorTime){//working time
         if(++compressorTime > POWERON_6HOUR + CLOSE_TIME_FIRSTTSG)
            compressorTime = POWERON_6HOUR + CLOSE_TIME_FIRSTTSG;
      }
      if(++persec > 100){
         persec = 0;
         signed short temp = setAdcOperation(NTC1_ADC_STRUCT, NTC_CONVER_GET);
         if( temp != NTC_ERROR){
            if(temp == NTC_RANGE_ERROR)
               iotinfo_module.error.errrefrigera = 1;
            else{
               iotinfo_module.realTemp = temp;
               iotinfo_module.error.errrefrigera = 0;
            }
            setAdcOperation(NTC1_ADC_STRUCT, NTC_CONVER_START);
         }
         ESP_LOGI(ATTAG,"Flag:0x%x off:%d cpTime:%d settemp:%d realTemp:%d TSG:%d TSK:%d err:0x%04X flag:%d ftime:%d esp_get_free_heap_size[%d]", controlFlag, iotinfo_module.devparam->powerstatus, compressorTime, 
                           iotinfo_module.devparam->settemp, iotinfo_module.realTemp, settemp - iotinfo_module.devparam->downbacklash, settemp + iotinfo_module.devparam->upbacklash,
                           iotinfo_module.error.errbit, flag, firstCloseTime, esp_get_free_heap_size());
      }
   }
   
   int isCompressorDelay(void) 
   {
      if( firstCloseTime )//(controlFlag & SET_FIRST_TSG) &&
         return 1;
      else if( compressorTime > 0 )//(controlFlag & SET_FIRST_TSG) &&
         return 2;
      else
         return 0;
   }
   
   void powerOnControl(char status) 
   {
      if(status == POWER_STATUS_ON || status == POWER_STATUS_LOCALON) {
         if(status == POWER_STATUS_LOCALON && iotinfo_module.devparam->powerstatus == POWER_STATUS_REMOTEOFF)
            return;
         if (compressorTime >= 0)
            SET_COMPRESSOR_OUTPUT(OUTPUT_LOW);
         iotinfo_module.firstOn        = 1;
         iotinfo_module.powerOnTime    = 0;
         firstCloseTime                = 0;
         compressorTime                = -1;
         controlFlag                   = (1 << SET_FIRST_TSG);
         iotinfo_module.devparam->powerstatus = POWER_STATUS_ON;
         powerOnDisplay(DISPLAY_TYPE_SET);
      }
      else if(status == POWER_STATUS_LOCALOFF || status == POWER_STATUS_REMOTEOFF) {
         if(iotinfo_module.devparam->powerstatus == POWER_STATUS_ON) {
            if (compressorTime >= 0)   
               compressorTime = -1;
            SET_COMPRESSOR_OUTPUT(OUTPUT_LOW);
            iotinfo_module.devparam->powerstatus = status;
            if(status == POWER_STATUS_LOCALOFF)
               powerOffLocal(DISPLAY_TYPE_SET);
            else if(status == POWER_STATUS_REMOTEOFF)
               powerOffRemote(DISPLAY_TYPE_SET);
         }
      }
   }
   
   void controlProcess(void)//10ms
   {
      iotInfoPro();
      drvGpioPro();
      updateAdcValue();
      compressorControl();
   }
   void controlInit(void) 
   {
     drvOutputGpioInit();
     drvAdcInit();
     iotInfoInit();
     setModuleStatus(1);
   
     setAdcOperation(NTC1_ADC_STRUCT, NTC_CONVER_START);
   }
