
.. _program_listing_file_main_iot_fzprotocol_send.c:

Program Listing for File fzprotocol_send.c
==========================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_iot_fzprotocol_send.c>` (``main\iot\fzprotocol_send.c``)

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
   #include "cmdprase.h"
   #include "mqtttopic.h"
   #include "fzprotocol.h"
   static const char *ATTAG = "fzProtocol_SEND";
   
   void fzReportRegisterInfo(void)
   {
      // /{ProductKey}/{DevId}/user/Rpt/DevInfo
      char tempbuf[16], *jsonstr=NULL;
      cJSON *reginfo_obj = NULL, *devModules_array = NULL, *array_obj = NULL, *moduleInfos_obj = NULL;
      reginfo_obj = cJSON_CreateObject();
      if(reginfo_obj)
      {
         cJSON_AddStringToObject(reginfo_obj, "protocolVers", "1.0");
         cJSON_AddStringToObject(reginfo_obj, "devType", "aokema9E");
         cJSON_AddStringToObject(reginfo_obj, "devState", "online");
         cJSON_AddStringToObject(reginfo_obj, "softwareVers", FREEZER_APP_VERSION);
         cJSON_AddStringToObject(reginfo_obj, "hardwareVers", "1.0.1");
         cJSON_AddItemToObject(reginfo_obj, "devModules", devModules_array = cJSON_CreateArray());
         if(devModules_array == NULL)
            goto end;
   
         cJSON_AddItemToArray(devModules_array, array_obj = cJSON_CreateObject());
         if(array_obj == NULL)
            goto end;
         cJSON_AddStringToObject(array_obj, "moduleType", "4G");
         snprintf(tempbuf, 16, "%lld", iotinfo_module.imei);
         cJSON_AddStringToObject(array_obj, "moduleId", tempbuf);
         cJSON_AddItemToObject(array_obj, "moduleInfos", moduleInfos_obj = cJSON_CreateObject());
         if(moduleInfos_obj == NULL)
            goto end;
         snprintf(tempbuf, 16, "%lld", iotinfo_module.imsi);
         cJSON_AddStringToObject(moduleInfos_obj, "IMSI", tempbuf);
         cJSON_AddStringToObject(moduleInfos_obj, "softwareVers", "1.0.1");
         cJSON_AddStringToObject(moduleInfos_obj, "hardwareVers", "1.0.1");
         cJSON_AddStringToObject(moduleInfos_obj, "supportUpgrade", "0");
   
         // cJSON_AddItemToArray(devModules_array, array_obj = cJSON_CreateObject());
         // if(array_obj == NULL)
         //    goto end;
         // cJSON_AddStringToObject(array_obj, "moduleType", "bluetooth");
            // if( esp_read_mac((uint8_t *)keytemp, ESP_MAC_BT) )//ESP_MAC_WIFI_STA
            //    return -1;
            // snprintf(buf, 13, "%02X%02X%02X%02X%02X%02X", keytemp[0], keytemp[1], keytemp[2], keytemp[3], keytemp[4], keytemp[5]);
         // cJSON_AddStringToObject(array_obj, "moduleId", tempbuf);
         // cJSON_AddItemToObject(array_obj, "moduleInfos", moduleInfos_obj = cJSON_CreateObject());
         // if(moduleInfos_obj == NULL)
         //    goto end;
         // cJSON_AddStringToObject(moduleInfos_obj, "IMSI", "imsi");
         // cJSON_AddStringToObject(moduleInfos_obj, "softwareVers", "1.0.1");
         // cJSON_AddStringToObject(moduleInfos_obj, "hardwareVers", "1.0.1");
         // cJSON_AddStringToObject(moduleInfos_obj, "supportUpgrade", "0");
   
         jsonstr = cJSON_PrintUnformatted(reginfo_obj);//malloc pointer
         if(jsonstr == NULL)
            goto end;
         // ESP_LOGI(ATTAG, "cjson is [%s]", jsonstr);
         publishMqttTopic(MQTT_TOPIC_PUB_DEVINFO, NULL, jsonstr);
         cJSON_free(jsonstr);
         jsonstr = NULL;
      }
   
   end:
      cJSON_Delete(reginfo_obj);
   }
   
   void fzReportAttrsInfo(char step)
   {
      // /{ProductKey}/{DevId}/user/Rpt/AttrRpt
      char *jsonstr=NULL;
      cJSON *attrsinfo_obj = NULL, *attrs_array = NULL;
      attrsinfo_obj = cJSON_CreateObject();
      if(attrsinfo_obj)
      {
         cJSON_AddItemToObject(attrsinfo_obj, "attrs", attrs_array = cJSON_CreateArray());
         if(attrs_array == NULL)
            goto end;
   
         unsigned int flag = CMD_BIT_MASK(SRQ_POWERON) | CMD_BIT_MASK(RQ_FROZEN_TEMP) | CMD_BIT_MASK(RQ_HISTORY_TEMP) | 
                  CMD_BIT_MASK(RQ_MAIN_CELL) | CMD_BIT_MASK(RQ_OTHER_CELL) | 
                  CMD_BIT_MASK(RQ_SCAN_WIFI_MAC_NUM) | CMD_BIT_MASK(RQ_SCAN_WIFI_MAC);
         if(step == 0)
            flag |= CMD_BIT_MASK(QUERY_WIFI_MAC);
         setCmdObj(attrs_array, flag, 0);
   
         jsonstr = cJSON_PrintUnformatted(attrsinfo_obj);//malloc pointer
         if(jsonstr == NULL)
            goto end;
         ESP_LOGI(ATTAG, "cjson is [%s]", jsonstr);
         publishMqttTopic(MQTT_TOPIC_PUB_ATTRRPT, NULL, jsonstr);
         cJSON_free(jsonstr);
         jsonstr = NULL;
      }
   
   end:
         // ESP_LOGI(ATTAG, "cjson is [in end]");
      cJSON_Delete(attrsinfo_obj);
   }
   
   void fzReportAlarmInfo(void)
   {
      // /{ProductKey}/{DevId}/user/Rpt/AlarmRpt
      char *jsonstr=NULL;
      cJSON *alarminfo_obj = NULL, *alarms_array = NULL;
      alarminfo_obj = cJSON_CreateObject();
      if(alarminfo_obj)
      {
         cJSON_AddItemToObject(alarminfo_obj, "alarms", alarms_array = cJSON_CreateArray());
         if(alarms_array == NULL)
            goto end;
   
         if(iotinfo_module.error.errrefrigera || iotinfo_module.error.errfrozen) {
            iotinfo_module.devinfo->probealarms    = 1;
            iotinfo_module.devinfo->noprobealarms  = 0;
         }
         else  {
            if(iotinfo_module.devinfo->probealarms)
               iotinfo_module.devinfo->noprobealarms  = 1;
            iotinfo_module.devinfo->probealarms = 0;
         }
   
         if(iotinfo_module.error.errnosim || iotinfo_module.error.errsim || iotinfo_module.error.errregister || iotinfo_module.at_module->atStatus)
               // iotinfo_module.error.errwifi || iotinfo_module.error.errmcu || iotinfo_module.error.errble)  
         {
            iotinfo_module.devinfo->communicatealarms    = 1;
            iotinfo_module.devinfo->nocommunicatealarms  = 0;
         }
         else  {
            if(iotinfo_module.devinfo->communicatealarms)
               iotinfo_module.devinfo->nocommunicatealarms  = 1;
            iotinfo_module.devinfo->communicatealarms = 0;
         }
   
         if(iotinfo_module.devinfo->noprobealarms && iotinfo_module.devinfo->nocommunicatealarms)
            iotinfo_module.devinfo->noalarms = 1;
         else if(iotinfo_module.devinfo->probealarms || iotinfo_module.devinfo->communicatealarms || iotinfo_module.devinfo->noprobealarms || iotinfo_module.devinfo->nocommunicatealarms)
            iotinfo_module.devinfo->noalarms = 0;
         else
            iotinfo_module.devinfo->noalarms = 1;
         
         unsigned int flag = CMD_BIT_MASK(ALARM_CANCLE) | 
                  CMD_BIT_MASK(ALARM_PROBE_CANCLE) | CMD_BIT_MASK(ALARM_PROBE) | 
                  CMD_BIT_MASK(ALARM_COMMUNICATE_CANCLE) | CMD_BIT_MASK(ALARM_COMMUNICATE);
         setCmdObj(alarms_array, flag, 0);
   
         iotinfo_module.devinfo->noalarms             = 0;
         iotinfo_module.devinfo->noprobealarms        = 0;
         iotinfo_module.devinfo->nocommunicatealarms  = 0;
   
         jsonstr = cJSON_PrintUnformatted(alarminfo_obj);//malloc pointer
         if(jsonstr == NULL)
            goto end;
         // ESP_LOGI(ATTAG, "cjson is [%s]", jsonstr);
         publishMqttTopic(MQTT_TOPIC_PUB_ALARMRPT, NULL, jsonstr);
         cJSON_free(jsonstr);
         jsonstr = NULL;
      }
   
   end:
      cJSON_Delete(alarminfo_obj);
   }
   
   // { 
   // "action":"alarmQuery",
   // "param" : {}
   // }
   void fzRespondAlarmQuery(const char* messageId, unsigned int respondbit)
   {
      // /{ProductKey}/{DevId}/user/Rpt/DevInfo
      char *jsonstr=NULL;
      cJSON *alarmresp_obj = NULL, *alarms_array = NULL;
      alarmresp_obj = cJSON_CreateObject();
      if(alarmresp_obj)
      {
         cJSON_AddStringToObject(alarmresp_obj, "action", "alarmQueryRep");
         cJSON_AddItemToObject(alarmresp_obj, "param", alarms_array = cJSON_CreateObject());
         if(alarms_array == NULL)
            goto end;
   
         iotinfo_module.devinfo->noprobealarms        = 0;
         iotinfo_module.devinfo->nocommunicatealarms  = 0;
         if(iotinfo_module.devinfo->probealarms == 0 && iotinfo_module.devinfo->communicatealarms == 0)
            iotinfo_module.devinfo->noalarms = 1;
         respondbit = CMD_BIT_MASK(ALARM_CANCLE) | CMD_BIT_MASK(ALARM_PROBE) | CMD_BIT_MASK(ALARM_COMMUNICATE);
         setCmdObj(alarms_array, respondbit, 2);
         iotinfo_module.devinfo->noalarms             = 0;
   
         jsonstr = cJSON_PrintUnformatted(alarmresp_obj);//malloc pointer
         if(jsonstr == NULL)
            goto end;
         // ESP_LOGI(ATTAG, "cjson is [%s]", jsonstr);
         publishMqttTopic(MQTT_TOPIC_PUB_RRPC, (char*)messageId, jsonstr);
         cJSON_free(jsonstr);
         jsonstr = NULL;
      }
   
   end:
      cJSON_Delete(alarmresp_obj);
   }
   
   // { "action":"attrQuery",
   // "param" : [
   // "20ga01", 
   // "20gw00", 
   // "60gw00" 
   // ] }
   void fzRespondAttrsQuery(const char* messageId, unsigned int respondbit)
   {
      // /{ProductKey}/{DevId}/user/Rpt/DevInfo
      char *jsonstr=NULL;
      cJSON *attrsinfo_obj = NULL, *attrs_array = NULL;
      attrsinfo_obj = cJSON_CreateObject();
      if(attrsinfo_obj)
      {
         cJSON_AddStringToObject(attrsinfo_obj, "action", "attrQueryRep");
         cJSON_AddItemToObject(attrsinfo_obj, "param", attrs_array = cJSON_CreateArray());
         if(attrs_array == NULL)
            goto end;
   
         setCmdObj(attrs_array, respondbit, 1);
   
         jsonstr = cJSON_PrintUnformatted(attrsinfo_obj);//malloc pointer
         if(jsonstr == NULL)
            goto end;
         // ESP_LOGI(ATTAG, "cjson is [%s]", jsonstr);
         publishMqttTopic(MQTT_TOPIC_PUB_RRPC, (char*)messageId, jsonstr);
         cJSON_free(jsonstr);
         jsonstr = NULL;
      }
   
   end:
      cJSON_Delete(attrsinfo_obj);
   }
   
   
   void fzReportShadowUpdate(int vid)
   {
      // /{ProductKey}/{DevId}/user/Rpt/DevInfo
      char *jsonstr=NULL;
      cJSON *update_obj = NULL, *state_obj = NULL, *reported_obj = NULL;
      update_obj = cJSON_CreateObject();
      if(update_obj)
      {
         cJSON_AddItemToObject(update_obj, "state", state_obj = cJSON_CreateObject());
         if(state_obj == NULL)
            goto end;
         cJSON_AddItemToObject(state_obj, "reported", reported_obj = cJSON_CreateObject());
         if(reported_obj == NULL)
            goto end;
   
         unsigned int flag = CMD_BIT_MASK(SRQ_POWERON) | CMD_BIT_MASK(RQ_FROZEN_TEMP) | CMD_BIT_MASK(QUERY_REPORT_FREQUENCY) | CMD_BIT_MASK(QUERY_MQTT_HEARTTIME) |
                  CMD_BIT_MASK(RQ_HISTORY_TEMP) |CMD_BIT_MASK(RQ_MAIN_CELL) | CMD_BIT_MASK(RQ_OTHER_CELL) | 
                  CMD_BIT_MASK(RQ_SCAN_WIFI_MAC_NUM) | CMD_BIT_MASK(RQ_SCAN_WIFI_MAC);
         setCmdObj(reported_obj, flag, 2);
   
         cJSON_AddStringToObject(update_obj, "method", "update");
         cJSON_AddNumberToObject(update_obj, "version", vid);
         jsonstr = cJSON_PrintUnformatted(update_obj);//malloc pointer
         if(jsonstr == NULL)
            goto end;
         // ESP_LOGI(ATTAG, "cjson is [%s]", jsonstr);
         publishMqttTopic(MQTT_TOPIC_PUB_SHADOW, NULL, jsonstr);
         cJSON_free(jsonstr);
         jsonstr = NULL;
      }
   
   end:
      cJSON_Delete(update_obj);
   }
   
   void fzRespondShadowUpdate(unsigned int setbit, int vid)
   {
      // /{ProductKey}/{DevId}/user/Rpt/DevInfo
      char *jsonstr=NULL;
      cJSON *update_obj = NULL, *state_obj = NULL, *reported_obj = NULL;
      update_obj = cJSON_CreateObject();
      if(update_obj)
      {
         cJSON_AddItemToObject(update_obj, "state", state_obj = cJSON_CreateObject());
         if(state_obj == NULL)
            goto end;
         cJSON_AddItemToObject(state_obj, "reported", reported_obj = cJSON_CreateObject());
         if(reported_obj == NULL)
            goto end;
   
         setCmdObj(reported_obj, setbit, 2);
   
         // cJSON_AddStringToObject(state_obj, "desired", "null");
         cJSON_AddStringToObject(update_obj, "method", "update");
         cJSON_AddNumberToObject(update_obj, "version", vid);
         jsonstr = cJSON_PrintUnformatted(update_obj);//malloc pointer
         if(jsonstr == NULL)
            goto end;
         // ESP_LOGI(ATTAG, "cjson is [%s]", jsonstr);
         publishMqttTopic(MQTT_TOPIC_PUB_SHADOW, NULL, jsonstr);
         cJSON_free(jsonstr);
         jsonstr = NULL;
      }
   
   end:
      cJSON_Delete(update_obj);
   }
   
   
   void fzRespondUgrade(int errnum)
   {
      // /{ProductKey}/{DevId}/user/Rpt/DevInfo
      char numbuf[5], *jsonstr=NULL;
      cJSON *update_obj = NULL;
      update_obj = cJSON_CreateObject();
      if(update_obj)
      {
         if(iotinfo_module.upgradeSn)
            cJSON_AddStringToObject(update_obj, "sn", iotinfo_module.upgradeSn);
         else
            cJSON_AddStringToObject(update_obj, "sn", "");
         snprintf(numbuf, 5, "%d", errnum);
         cJSON_AddStringToObject(update_obj, "errorNo", numbuf);
   
         jsonstr = cJSON_PrintUnformatted(update_obj);//malloc pointer
         if(jsonstr == NULL)
            goto end;
         // ESP_LOGI(ATTAG, "cjson is [%s]", jsonstr);
         publishMqttTopic(MQTT_TOPIC_PUB_UPGRADEACK, NULL, jsonstr);
         cJSON_free(jsonstr);
         jsonstr = NULL;
      }
   
   end:
      cJSON_Delete(update_obj);
   }
   
