
.. _program_listing_file_main_iot_cmdprase.c:

Program Listing for File cmdprase.c
===================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_iot_cmdprase.c>` (``main\iot\cmdprase.c``)

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
   #include <stdio.h>
   #include <string.h>
   #include "esp_system.h"
   #include "esp_log.h"
   #include "cJSON.h"
   #include "iotinfo.h"
   #include "mqtttopic.h"
   #include "fzprotocol.h"
   #include "cmdprase.h"
   #include "controlpro.h"
   
   
   
   #define CMD_PREFIX_LEN           4
   #define CMD_PREFIX_20GA          "20ga"
   #define CMD_PREFIX_20GA_OFFSET   0
   #define CMD_PREFIX_20GW          "20gw"
   #define CMD_PREFIX_20GW_OFFSET   -2
   #define CMD_PREFIX_20GW_OFFSET_1 3
   #define CMD_PREFIX_60GW          "60gw"
   #define CMD_PREFIX_60GW_OFFSET   -2
   #define CMD_PREFIX_50GW          "50gw"
   #define CMD_PREFIX_50GW_OFFSET   -2
   
   
   #define CMD_SET_ONOFF            "20gw00"
   #define CMD_SET_TEMPERATURE      "20gw01"
   #define CMD_SET_REPORTTIME       "20gw07"
   #define CMD_SET_HEARTTIME        "20gw08"
   static const char *ATTAG = "cmd_prase";
   
   static void setQueryBit(const char* cmd, unsigned int *flag)
   {
      int num=-1;
      if(cmd == NULL)
         return;
      else if( memcmp(cmd, CMD_PREFIX_20GA, CMD_PREFIX_LEN) == 0 )
      {
         num = atoi(cmd + CMD_PREFIX_LEN);
         if(num == 0)
            *flag |= CMD_BIT_MASK(QUERY_IMSI);
         else if(num == 1)
            *flag |= CMD_BIT_MASK(QUERY_WIFI_MAC);
      }
      else if( memcmp(cmd, CMD_PREFIX_20GW, CMD_PREFIX_LEN) == 0 )
      {
         num = atoi(cmd + CMD_PREFIX_LEN);
         if(num == 0)
            *flag |= CMD_BIT_MASK(SRQ_POWERON);
         else if(num == 1)
            *flag |= CMD_BIT_MASK(SQ_FROZEN_TEMP);
         else if(num == 7)
            *flag |= CMD_BIT_MASK(QUERY_REPORT_FREQUENCY);
         else if(num == 8)
            *flag |= CMD_BIT_MASK(QUERY_MQTT_HEARTTIME);
      }
      else if( memcmp(cmd, CMD_PREFIX_60GW, CMD_PREFIX_LEN) == 0 )
      {
         num = atoi(cmd + CMD_PREFIX_LEN);
         if(num == 0)
            *flag |= CMD_BIT_MASK(RQ_FROZEN_TEMP);
         else if(num == 1)
            *flag |= CMD_BIT_MASK(RQ_FROZEN_TEMP_REVERSAVE);
         else if(num == 2)
            *flag |= CMD_BIT_MASK(RQ_HUAN_TEMP_REVERSAVE);
         else if(num == 3)
            *flag |= CMD_BIT_MASK(RQ_HISTORY_TEMP);
         else if(num == 4)
            *flag |= CMD_BIT_MASK(RQ_MAIN_CELL);
         else if(num == 5)
            *flag |= CMD_BIT_MASK(RQ_OTHER_CELL);
         else if(num == 6)
            *flag |= CMD_BIT_MASK(RQ_SCAN_WIFI_MAC_NUM);
         else if(num == 7)
            *flag |= CMD_BIT_MASK(RQ_SCAN_WIFI_MAC);
      }
      else if( memcmp(cmd, CMD_PREFIX_50GW, CMD_PREFIX_LEN) == 0 )
      {
         num = atoi(cmd + CMD_PREFIX_LEN);
         if(num == 0)
            *flag |= CMD_BIT_MASK(ALARM_CANCLE);
         else if(num == 1)
            *flag |= CMD_BIT_MASK(ALARM_PROBE_CANCLE);
         else if(num == 2)
            *flag |= CMD_BIT_MASK(ALARM_PROBE);
         else if(num == 7)
            *flag |= CMD_BIT_MASK(ALARM_COMMUNICATE_CANCLE);
         else if(num == 8)
            *flag |= CMD_BIT_MASK(ALARM_COMMUNICATE);
      }
   }
   
   int operateHistoryTemp(int temp)
   {
   #define HISTORY_ONE_TEMP 6
      int num,len;
      char *buf=NULL, *str=NULL;
      if(iotinfo_module.devinfo->historytemp)   {
         len = strlen(iotinfo_module.devinfo->historytemp);
         buf = (char *)malloc(len + HISTORY_ONE_TEMP);
         if(buf == NULL)
            return -1;
         num = atoi(iotinfo_module.devinfo->historytemp);
         if(num <= 0)   
            snprintf(buf, len + HISTORY_ONE_TEMP, "1,%d", temp);
         else  {
            num = num + 1;
            str = strchr(iotinfo_module.devinfo->historytemp, ',');
            if(num <= 10)
               snprintf(buf, len + HISTORY_ONE_TEMP, "%d%s/%d", num, str, temp);
            else  {
               str = strchr(str, '/');
               snprintf(buf, len + HISTORY_ONE_TEMP, "10,%s/%d", str+1, temp);
            } //num > 10
         }//num > 0
      }//historytemp
      else  {
         buf = (char *)malloc(HISTORY_ONE_TEMP);
         if(buf == NULL)
            return -1;
         snprintf(buf, HISTORY_ONE_TEMP, "1,%d", temp);
      }
      
      if(iotinfo_module.devinfo->historytemp)
         free(iotinfo_module.devinfo->historytemp);
      iotinfo_module.devinfo->historytemp = buf;
      return 0;
   }
   
   int operateCellInfo(char type, int mcc, int mnc, int lac, int cellid, int signal)
   {
   #define CELLINFO_ONE_TEMP 32
      int num,len;
      char *buf=NULL, **cellinfo=NULL;
      if(type == 1)
         cellinfo = &iotinfo_module.devinfo->mainlbs;
      else if(type == 3)
         cellinfo = &iotinfo_module.devinfo->otherlbs;
      else if(type == 2)   {
         if(iotinfo_module.devinfo->mainlbs)
            free(iotinfo_module.devinfo->mainlbs);
         iotinfo_module.devinfo->mainlbs = NULL;
         return 0;
      }
      else if(type == 4)   {
         if(iotinfo_module.devinfo->otherlbs)
            free(iotinfo_module.devinfo->otherlbs);
         iotinfo_module.devinfo->otherlbs = NULL;
         iotinfo_module.devinfo->otherlbsnum = 0;
         return 0;
      }
      else
         return -2;
   
   // mcc,mnc,lac,cellid,signal
   // 460,0,33306,19732,-68
      if(type == 1)   {
         buf = (char *)malloc(CELLINFO_ONE_TEMP);
         if(buf == NULL)
            return -1;
         snprintf(buf, CELLINFO_ONE_TEMP, "%03d,%01d,%d,%d,%d", mcc, mnc, lac, cellid, signal);
      }
      else  if(type == 3 && iotinfo_module.devinfo->otherlbsnum < 3)  {
         if(*cellinfo)
            len = strlen(*cellinfo);
         else
            len = 0;
         buf = (char *)malloc(len + CELLINFO_ONE_TEMP);
         if(buf == NULL)
            return -1;
         num = iotinfo_module.devinfo->otherlbsnum;
         if(num == 0)   {
            iotinfo_module.devinfo->otherlbsnum = 1;
            snprintf(buf, len + CELLINFO_ONE_TEMP, "%03d,%01d,%d,%d,%d", mcc, mnc, lac, cellid, signal);
         }
         else if(num < 3)   {
            iotinfo_module.devinfo->otherlbsnum++;
            snprintf(buf, len + CELLINFO_ONE_TEMP, "%s|%03d,%01d,%d,%d,%d", *cellinfo, mcc, mnc, lac, cellid, signal);
         }
         // else  {
         //    str = strchr(*cellinfo, '|');
         //    snprintf(buf, len + CELLINFO_ONE_TEMP, "%s|%03d,%01d,%d,%d,%d", str+1, mcc, mnc, lac, cellid, signal);
         //    iotinfo_module.devinfo->otherlbsnum = 3;
         // } //num > 10
      }//historytemp
      else
         return -2;
      
      if(*cellinfo)
         free(*cellinfo);
      *cellinfo = buf;
      return 0;
   }
   
   static void resetScanWifi(void)
   {
      // ESP_LOGI(ATTAG, "%s wifi:%d macstr:[%s]", __FUNCTION__ , iotinfo_module.devinfo->wifinum, iotinfo_module.devinfo->wifimac);
      if(iotinfo_module.devinfo->wifimac)
         free(iotinfo_module.devinfo->wifimac);
      iotinfo_module.devinfo->wifimac = NULL;
      iotinfo_module.devinfo->wifinum     = 0;
      
   }
   
   static int addScanWifi(const char *ssid, const signed char signal, const char *mac)
   {
   #define WIFIMAC_ONE_TEMP 64
      int num,len;
      char *buf=NULL, *str=NULL;
      
      // ESP_LOGI(ATTAG, "%s in  [%d] :", __FUNCTION__, esp_get_free_heap_size());
   
      if(iotinfo_module.devinfo->wifinum && iotinfo_module.devinfo->wifimac)   {
         len = strlen(iotinfo_module.devinfo->wifimac);
         buf = (char *)malloc(len + WIFIMAC_ONE_TEMP);
         if(buf == NULL)
            return -1;
         num = iotinfo_module.devinfo->wifinum;
         if(num < 10)   {
            iotinfo_module.devinfo->wifinum++;
            snprintf(buf, len + WIFIMAC_ONE_TEMP, "%s|%02x:%02x:%02x:%02x:%02x:%02x,%d,%s", iotinfo_module.devinfo->wifimac, mac[0], mac[1], mac[2], mac[3], mac[4], mac[5], signal, ssid);
         }
         else  {
            str = strchr(iotinfo_module.devinfo->wifimac, '|');
            snprintf(buf, len + WIFIMAC_ONE_TEMP, "%s|%02x:%02x:%02x:%02x:%02x:%02x,%d,%s", str+1, mac[0], mac[1], mac[2], mac[3], mac[4], mac[5], signal, ssid);
            iotinfo_module.devinfo->wifinum = 10;
            
            // wifiScanStart(0);
         } //num > 10
      }//wifimac
      else  {
         buf = (char *)malloc(WIFIMAC_ONE_TEMP);
         if(buf == NULL)
            return -1;
         snprintf(buf, WIFIMAC_ONE_TEMP, "%02x:%02x:%02x:%02x:%02x:%02x,%d,%s", mac[0], mac[1], mac[2], mac[3], mac[4], mac[5], signal, ssid);
         iotinfo_module.devinfo->wifinum = 1;
      }
      
      if(iotinfo_module.devinfo->wifimac)
         free(iotinfo_module.devinfo->wifimac);
      iotinfo_module.devinfo->wifimac = buf;
      // ESP_LOGI(ATTAG, "%s out  [%d] :", __FUNCTION__, esp_get_free_heap_size());
      return 0;
   }
   int operateScanWifi(char type, const char *ssid, const signed char signal, const char *mac)
   {
      if(type == 1)  
         return addScanWifi(ssid, signal, mac);
      else if(type == 0)  
         resetScanWifi();
      else
         return -2;
      return 0;
   }
   
   int setCmdObj(cJSON *array, unsigned int flag, const char isobj)
   {
      int len=0;
      char keytemp[8];
      static char *buf;
      cJSON *jsontemp=NULL;
      if(flag && array && ( (isobj < 2 && cJSON_IsArray(array)) || (isobj == 2 && cJSON_IsObject(array)) )  )  {
         if(isobj == 1)  
            cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
         else if(isobj == 2)  
            jsontemp = array;
   
         if( flag & CMD_BIT_MASK(QUERY_IMSI) ){
            buf = (char*) realloc(buf, AT_CMD_IMEI_IMSI_LEN+1);
            if(buf == NULL)
               return -1;
            snprintf(buf, AT_CMD_IMEI_IMSI_LEN+1, "%lld", iotinfo_module.imsi);
            if(isobj)
               cJSON_AddStringToObject(jsontemp, "20ga00", buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, "20ga00", buf);
            }
         }
   
         if( flag & CMD_BIT_MASK(QUERY_WIFI_MAC) ){
            buf = (char*) realloc(buf, 13);
            if(buf == NULL)
               return -1;
            if( esp_read_mac((uint8_t *)keytemp, ESP_MAC_WIFI_STA) )//ESP_MAC_WIFI_STA
               return -1;
            snprintf(buf, 13, "%02X%02X%02X%02X%02X%02X", keytemp[0], keytemp[1], keytemp[2], keytemp[3], keytemp[4], keytemp[5]);
            if(isobj)
               cJSON_AddStringToObject(jsontemp, "20ga01", buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, "20ga01", buf);
            }
         }
   
         if( flag & CMD_BIT_MASK(SRQ_POWERON) ){
            buf = (char*) realloc(buf, 7);
            if(buf == NULL)
               return -1;
            if(iotinfo_module.devparam->powerstatus == POWER_STATUS_ON)
               len = 0;
            else
               len = 1;
            snprintf(buf, 7, "30g10%01d", len);
            if(isobj)
               cJSON_AddStringToObject(jsontemp, CMD_SET_ONOFF, buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, CMD_SET_ONOFF, buf);
            }
         }
   
         if( flag & CMD_BIT_MASK(SQ_FROZEN_TEMP) ){
            buf = (char*) realloc(buf, 8);
            if(buf == NULL)
               return -1;
            snprintf(buf, 8, "%d", iotinfo_module.devparam->settemp);
            if(isobj)
               cJSON_AddStringToObject(jsontemp, CMD_SET_TEMPERATURE, buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, CMD_SET_TEMPERATURE, buf);
            }
         }
   
         if( flag & CMD_BIT_MASK(QUERY_REPORT_FREQUENCY) ){
            buf = (char*) realloc(buf, 4);
            if(buf == NULL)
               return -1;
            snprintf(buf, 4, "%d", iotinfo_module.devparam->reportTime);
            if(isobj)
               cJSON_AddStringToObject(jsontemp, CMD_SET_REPORTTIME, buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, CMD_SET_REPORTTIME, buf);
            }
         }
   
         if( flag & CMD_BIT_MASK(QUERY_MQTT_HEARTTIME) ){
            buf = (char*) realloc(buf, 5);
            if(buf == NULL)
               return -1;
            snprintf(buf, 5, "%d", iotinfo_module.devparam->heartTime);
            if(isobj)
               cJSON_AddStringToObject(jsontemp, CMD_SET_HEARTTIME, buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, CMD_SET_HEARTTIME, buf);
            }
         }
   
         if( flag & CMD_BIT_MASK(RQ_FROZEN_TEMP) ){
            buf = (char*) realloc(buf, 8);
            if(buf == NULL)
               return -1;
            snprintf(buf, 8, "%d", iotinfo_module.devparam->settemp);
            if(isobj)
               cJSON_AddStringToObject(jsontemp, "60gw00", buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, "60gw00", buf);
            }
         }
   
         if( iotinfo_module.devinfo->historytemp && (flag & CMD_BIT_MASK(RQ_HISTORY_TEMP)) ){
            len = strlen(iotinfo_module.devinfo->historytemp) + 1;
            buf = (char*) realloc(buf, len);
            if(buf == NULL)
               return -1;
            snprintf(buf, len, "%s", iotinfo_module.devinfo->historytemp);
            if(isobj)
               cJSON_AddStringToObject(jsontemp, "60gw03", buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, "60gw03", buf);
            }
         }
   
         if( (flag & CMD_BIT_MASK(RQ_MAIN_CELL)) ){
            len = iotinfo_module.devinfo->mainlbs ? strlen(iotinfo_module.devinfo->mainlbs) + 1 : 4;
            buf = (char*) realloc(buf, len);
            if(buf == NULL)
               return -1;
            if( iotinfo_module.devinfo->mainlbs )   
               snprintf(buf, len, "%s", iotinfo_module.devinfo->mainlbs);
            else 
               snprintf(buf, len, "%s", "");
            
            if(isobj)
               cJSON_AddStringToObject(jsontemp, "60gw04", buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, "60gw04", buf);
            }
         }
   
         if( (flag & CMD_BIT_MASK(RQ_OTHER_CELL)) ){
            len = iotinfo_module.devinfo->otherlbs ? strlen(iotinfo_module.devinfo->otherlbs) + 1 : 4;
            buf = (char*) realloc(buf, len);
            if(buf == NULL)
               return -1;
            if( iotinfo_module.devinfo->otherlbs )   
               snprintf(buf, len, "%s", iotinfo_module.devinfo->otherlbs);
            else 
               snprintf(buf, len, "%s", "");
   
            if(isobj)
               cJSON_AddStringToObject(jsontemp, "60gw05", buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, "60gw05", buf);
            }
         }
   
         if( flag & CMD_BIT_MASK(RQ_SCAN_WIFI_MAC_NUM) ){
            buf = (char*) realloc(buf, 4);
            if(buf == NULL)
               return -1;
            snprintf(buf, 4, "%d", iotinfo_module.devinfo->wifinum);
            if(isobj)
               cJSON_AddStringToObject(jsontemp, "60gw06", buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, "60gw06", buf);
            }
         }
   
         if( (flag & CMD_BIT_MASK(RQ_SCAN_WIFI_MAC)) ){
            len = iotinfo_module.devinfo->wifimac ? strlen(iotinfo_module.devinfo->wifimac) + 1 : 4;
            buf = (char*) realloc(buf, len);
            if(buf == NULL)
               return -1;
   
            if( iotinfo_module.devinfo->wifimac )   
               snprintf(buf, len, "%s", iotinfo_module.devinfo->wifimac);
            else 
               snprintf(buf, len, "%s", "");
   
            if(isobj)
               cJSON_AddStringToObject(jsontemp, "60gw07", buf);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, "60gw07", buf);
            }
         }
   
         if(iotinfo_module.devinfo->noalarms && (flag & CMD_BIT_MASK(ALARM_CANCLE)) ){
            snprintf(keytemp, 7, "50gw00");
            if(isobj)
               cJSON_AddStringToObject(jsontemp, keytemp, keytemp);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, keytemp, keytemp);
            }
         }
   
         if( iotinfo_module.devinfo->noprobealarms && (flag & CMD_BIT_MASK(ALARM_PROBE_CANCLE)) ){
            snprintf(keytemp, 7, "50gw01");
            if(isobj)
               cJSON_AddStringToObject(jsontemp, keytemp, keytemp);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, keytemp, keytemp);
            }
         }
   
         if( iotinfo_module.devinfo->probealarms && (flag & CMD_BIT_MASK(ALARM_PROBE)) ){
            snprintf(keytemp, 7, "50gw02");
            if(isobj)
               cJSON_AddStringToObject(jsontemp, keytemp, keytemp);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, keytemp, keytemp);
            }
         }
   
         if( iotinfo_module.devinfo->communicatealarms && (flag & CMD_BIT_MASK(ALARM_COMMUNICATE_CANCLE)) ){
            snprintf(keytemp, 7, "50gw07");
            if(isobj)
               cJSON_AddStringToObject(jsontemp, keytemp, keytemp);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, keytemp, keytemp);
            }
         }
   
         if( iotinfo_module.devinfo->nocommunicatealarms && (flag & CMD_BIT_MASK(ALARM_COMMUNICATE)) ){
            snprintf(keytemp, 7, "50gw08");
            if(isobj)
               cJSON_AddStringToObject(jsontemp, keytemp, keytemp);
            else  {
               cJSON_AddItemToArray(array, jsontemp = cJSON_CreateObject());
               if(jsontemp == NULL)
                  return -1;
               cJSON_AddStringToObject(jsontemp, keytemp, keytemp);
            }
         }
   
      }
      return -1;
   }
   int rrpcAttrQueryParam(const char* messageId, const cJSON *array)
   {
      int size=0;
      unsigned int flag=0;
      cJSON *aitem=NULL;
      if(array && cJSON_IsArray(array))  {
         size = cJSON_GetArraySize(array);
         if(size) {
            flag = 0;
            for(int i=0; i<size; i++)  {
               aitem = cJSON_GetArrayItem(array, i);
               if(aitem && cJSON_IsString(aitem))  {
                  ESP_LOGI(ATTAG, "%s rrpc action query [%s]", __FUNCTION__ , cJSON_GetStringValue(aitem));
                  setQueryBit(cJSON_GetStringValue(aitem), &flag);
               }
            }
            ESP_LOGI(ATTAG, "%s rrpc action query 0x%x", __FUNCTION__ , flag);
            // if(flag)
            //    fzRespondAttrsQuery(messageId, flag);
   
            return 0;
         }
      }
      return -1;
   }
   
   extern void displayTempAlarm(int settime);
   extern void setDisplay_0(signed char settemp);
   void shadowRemoteControl(const cJSON *desired, int vid)
   {
      int temp=100;
      unsigned int flag = 0;
      cJSON *tnode = NULL;
      if(desired && cJSON_IsObject(desired))  {
         cJSON_ArrayForEach(tnode, desired)
         {
            if(cJSON_IsString(tnode)){
               ESP_LOGI(ATTAG, "%s desired %s:%s", __FUNCTION__ , tnode->string, tnode->valuestring);
               setQueryBit(tnode->string, &flag);
            }
            else  {
               ESP_LOGI(ATTAG, "%s desired no string", __FUNCTION__ );
            }
         }
   
         if(flag & CMD_BIT_MASK(SQ_FROZEN_TEMP))   {
            tnode = cJSON_GetObjectItem(desired, CMD_SET_TEMPERATURE);
            if(tnode && cJSON_IsString(tnode) )   {
               temp = atoi(tnode->valuestring);
               if(temp >= -50 && temp <= 50 && temp != iotinfo_module.devparam->settemp) {
                  setDisplay_0(temp - iotinfo_module.devparam->settemp);
                  iotinfo_module.devparam->settemp = temp;
                  displayTempAlarm(1);
               }
               else
                  flag &= ~CMD_BIT_MASK(SQ_FROZEN_TEMP);
            }
         }
         if(flag & CMD_BIT_MASK(QUERY_REPORT_FREQUENCY))   {
            tnode = cJSON_GetObjectItem(desired, CMD_SET_REPORTTIME);
            if(tnode && cJSON_IsString(tnode) )   {
               temp = atoi(tnode->valuestring);
               if(temp >= 1 && temp <= 60 && temp != iotinfo_module.devparam->reportTime) {
                  iotinfo_module.devparam->reportTime = temp;
               }
               else
                  flag &= ~CMD_BIT_MASK(QUERY_REPORT_FREQUENCY);
            }
         }
         if(flag & CMD_BIT_MASK(QUERY_MQTT_HEARTTIME))   {
            tnode = cJSON_GetObjectItem(desired, CMD_SET_HEARTTIME);
            if(tnode && cJSON_IsString(tnode) )   {
               temp = atoi(tnode->valuestring);
               if(temp >= 30 && temp <= 1200 && temp != iotinfo_module.devparam->heartTime) {
                  iotinfo_module.devparam->heartTime = temp;
               }
               else
                  flag &= ~CMD_BIT_MASK(QUERY_MQTT_HEARTTIME);
            }
         }
         if(flag & CMD_BIT_MASK(SRQ_POWERON))   {
            tnode = cJSON_GetObjectItem(desired, CMD_SET_ONOFF);
            if( tnode && cJSON_IsString(tnode) ) {
               temp = (memcmp(tnode->valuestring, "30g101", strlen("30g101")) == 0) ? 1:0;
               if(temp && iotinfo_module.devparam->powerstatus == POWER_STATUS_ON) //poweron
                  powerOnControl(POWER_STATUS_REMOTEOFF);
               else if(temp == 0 && iotinfo_module.devparam->powerstatus) //force poweroff
                  powerOnControl(POWER_STATUS_ON);
               else
                  flag &= ~CMD_BIT_MASK(SRQ_POWERON);
            }
         }
         
         if(flag)
            fzRespondShadowUpdate(flag, vid); 
      }
   }
