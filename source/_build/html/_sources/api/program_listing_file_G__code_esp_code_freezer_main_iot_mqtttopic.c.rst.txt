
.. _program_listing_file_G__code_esp_code_freezer_main_iot_mqtttopic.c:

Program Listing for File mqtttopic.c
====================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_iot_mqtttopic.c>` (``G:\code\esp\code\freezer\main\iot\mqtttopic.c``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /* topic Example
   /sys/a14XBNg6pCp/123456789012345/user/Rpt/AlarmRpt
   RRPC请求消息Topic：  /sys/${YourProductKey}/${YourDeviceName}/rrpc/request/${messageId}
   RRPC响应消息Topic：  /sys/${YourProductKey}/${YourDeviceName}/rrpc/response/${messageId}
   RRPC订阅Topic：   /sys/${YourProductKey}/${YourDeviceName}/rrpc/request/+
   
   shadow发布Topic   /shadow/update/${YourProductKey}/${YourDeviceName}
   shadow订阅Topic   /shadow/get/${YourProductKey}/${YourDeviceName}
   
   发布告警上报Topic    /${YourProductKey}/${YourDeviceName}/user/Rpt/AlarmRpt
   发布属性上报Topic    /${YourProductKey}/${YourDeviceName}/user/Rpt/AttrRpt
   发布设备信息Topic    /${YourProductKey}/${YourDeviceName}/user/Rpt/DevInfo
   订阅Topic           /${YourProductKey}/${YourDeviceName}/user/Rpt/+
   
   发布升级确认信息Topic    /${YourProductKey}/${YourDeviceName}/user/UpgradeAck
   订阅设备升级Topic    /${YourProductKey}/${YourDeviceName}/user/Upgrade
   */
   
   #include <stdio.h>
   #include <string.h>
   // #include "esp_system.h"
   #include "esp_log.h"
   #include "atcmd.h"
   #include "atmqtt.h"
   #include "iotinfo.h"
   #include "mqtttopic.h"
   #include "fzprotocol.h"
   static const char *ATTAG = "mqttpro";
   
   #pragma pack(1)
   typedef struct {
      char           *prefix;
      char           *suffix;
      char           *flexfix;
   } mqtt_tiopic_t;
   #pragma pack()
   mqtt_topic_map_opt_t mqtttopic_current=MQTT_TOPIC_NONE;
   static const mqtt_tiopic_t mqtt_tiopic[] =
   {
      {"/sys/", "/rrpc/request", "/+"},
      {"/shadow/get/", NULL, NULL},
      {"/", "/user/Rpt", "/+"},
      {"/", "/user/Upgrade", NULL},//MQTT_TOPIC_SUB_END
      {"/shadow/update/", NULL, NULL},
      {"/", "/user/Rpt", "/DevInfo"},
      {"/", "/user/Rpt", "/AttrRpt"},
      {"/", "/user/Rpt", "/AlarmRpt"},//MQTT_TOPIC_PUB_END
      {"/sys/", "/rrpc/response/", NULL},
      {"/", "/user/UpgradeAck", NULL},
   };
   
   static void iotMqttSubSuccessCb(unsigned char status)
   {
      if(mqtttopic_current < MQTT_TOPIC_SUB_END)  {
         if(++mqtttopic_current == MQTT_TOPIC_SUB_END){
            ESP_LOGI(ATTAG, "%s sub all [status:%d] [mqtttopic:%d]", __FUNCTION__ , status, mqtttopic_current);
            fzReportShadowUpdate(-1);
         }
         else
            subMqttTopic(mqtttopic_current, mqtt_tiopic[mqtttopic_current].flexfix);
      }
   }
   static void iotMqttUnSubSuccessCb(unsigned char status)
   {
      if(mqtttopic_current && mqtttopic_current < MQTT_TOPIC_SUB_END)  {
         --mqtttopic_current;
         unSubMqttTopic(mqtttopic_current, mqtt_tiopic[mqtttopic_current].flexfix);
      }
   }
   static void iotMqttPubSuccessCb(unsigned char status)
   {
      if(iotinfo_module.iotStatus < IOT_CONNECTMQTT)  {
         if(++mqtttopic_current < MQTT_TOPIC_PUB_END)  {
            if(mqtttopic_current == MQTT_TOPIC_PUB_DEVINFO) 
               fzReportRegisterInfo();
            else if(mqtttopic_current == MQTT_TOPIC_PUB_ATTRRPT) 
               fzReportAttrsInfo(0);
            else if(mqtttopic_current == MQTT_TOPIC_PUB_ALARMRPT) 
               fzReportAlarmInfo();
         }
         else if(mqtttopic_current == MQTT_TOPIC_PUB_END){
            iotinfo_module.iotStatus = IOT_CONNECTMQTT;
            ESP_LOGI(ATTAG, "%s pub all [status:%d] [mqtttopic:%d]", __FUNCTION__ , status, mqtttopic_current);
         }
      }
      if(mqtttopic_current == MQTT_TOPIC_PUB_UPGRADEACK) {
         if(iotinfo_module.otaStatus == 3)  {//OTA_SUCCESS
            ESP_LOGI(ATTAG, "Prepare to restart system!");
   extern void  esp_restart(void);
            esp_restart();
         }
      }
      
   }
   static void iotMqttOperationCb(unsigned char status)
   {
      ESP_LOGI(ATTAG, "%s status [%d] topic [%d]", __FUNCTION__ , status, mqtttopic_current);
      if(status > MQTT_CONNECT_FAIL && status%2 == 0)
         ATSENDCMD_CHANGE_NULL();
      switch (status)
      {
      case MQTT_CONNECT_SUCCESS:
         subMqttTopic(MQTT_TOPIC_SUB_RRPC, NULL);
         break;
      case MQTT_CONNECT_SUB_SUCCESS:
         iotMqttSubSuccessCb(0);
         break;
      case MQTT_CONNECT_UNSUB_SUCCESS:
         iotMqttUnSubSuccessCb(0);
         break;
      case MQTT_CONNECT_PUB_SUCCESS:
         iotMqttPubSuccessCb(0);
         break;
      case MQTT_DISCONNECT_SUCCESS:
         ESP_LOGI(ATTAG, "%s MQTT_DISCONNECT_SUCCESS", __FUNCTION__ );
         break;
      
      default:
         if(status > MQTT_CONNECT_FAIL && status%2)
            ATSENDCMD_CHANGE_NULL();
         ESP_LOGI(ATTAG, "%s other or fail", __FUNCTION__ );
         break;
      }
   }
   static void iotMqttReceiveCb(char *topic, char *payload)
   {
      unsigned long long imei;
      char *str=NULL, *rectopic=NULL, *recpayload=NULL;
      if(topic && payload)
      {
         // ESP_LOGI(ATTAG, "%s topic [%s] payload [%s]", __FUNCTION__ , topic, payload);
         rectopic = malloc(strlen(topic) + 1 );
         if(rectopic)
         {
            recpayload = malloc(strlen(payload) + 1 );
            if(recpayload == NULL)
            {
               free(rectopic);
               rectopic = NULL;
            }
         }
         else 
            return;
   
         setReportTime(0);//set report clean
         memcpy(rectopic, topic, strlen(topic) + 1);
         memcpy(recpayload, payload, strlen(payload) + 1);
         ESP_LOGI(ATTAG, "%s topic [%s] payload [%s]", __FUNCTION__ , rectopic, payload);
   
         str = strstr(rectopic, iotinfo_module.productKey);
         if(str){
            ESP_LOGI(ATTAG, "%s pk [%s]", __FUNCTION__ , str);
            imei = str2Imei(str + strlen(iotinfo_module.productKey) + 1);
            if(imei == iotinfo_module.imei){
               if( memcmp(rectopic, mqtt_tiopic[MQTT_TOPIC_SUB_SHADOW].prefix, strlen(mqtt_tiopic[MQTT_TOPIC_SUB_SHADOW].prefix) ) == 0 ){
                  fzReceiveShadowMsg(payload, strlen(payload) + 1);
               }//shadow
               else  {
                  str = str + strlen(iotinfo_module.productKey) + AT_CMD_IMEI_IMSI_LEN + 1;
                  ESP_LOGI(ATTAG, "%s topic end [%s]", __FUNCTION__ , str);
                  if( memcmp(str, mqtt_tiopic[MQTT_TOPIC_SUB_RRPC].suffix, strlen(mqtt_tiopic[MQTT_TOPIC_SUB_RRPC].suffix) ) == 0 ){
                     str = str + strlen(mqtt_tiopic[MQTT_TOPIC_SUB_RRPC].suffix) + 1;
                     ESP_LOGI(ATTAG, "%s topic rrpc id [%s]", __FUNCTION__ , str);
                     fzReceiveRrpcMsg(payload, strlen(payload) + 1, str);
                  }//rrpc
                  else if( memcmp(str, mqtt_tiopic[MQTT_TOPIC_SUB_RPT].suffix, strlen(mqtt_tiopic[MQTT_TOPIC_SUB_RPT].suffix) ) == 0 ){
                     str = str + strlen(mqtt_tiopic[MQTT_TOPIC_SUB_RPT].suffix);
                     ESP_LOGI(ATTAG, "%s user topic [%s]", __FUNCTION__, str);
                     if(memcmp(str, mqtt_tiopic[MQTT_TOPIC_PUB_ALARMRPT].flexfix, strlen(mqtt_tiopic[MQTT_TOPIC_PUB_ALARMRPT].flexfix)) == 0){
                     }//AlarmRpt
                     else if(memcmp(str, mqtt_tiopic[MQTT_TOPIC_PUB_ALARMRPT].flexfix, strlen(mqtt_tiopic[MQTT_TOPIC_PUB_ALARMRPT].flexfix)) == 0){
                     }//AttrRpt
                     else if(memcmp(str, mqtt_tiopic[MQTT_TOPIC_PUB_DEVINFO].flexfix, strlen(mqtt_tiopic[MQTT_TOPIC_PUB_DEVINFO].flexfix)) == 0){
                     }//DevInfo
                  }
                  else if( memcmp(str, mqtt_tiopic[MQTT_TOPIC_SUB_UPGRADE].suffix, strlen(mqtt_tiopic[MQTT_TOPIC_SUB_UPGRADE].suffix) ) == 0 ){
                     fzReceiveUpgradeMsg(payload, strlen(payload) + 1);
                  }//upgrade
               }
            }//imei
         }//productKey
   
         free(rectopic);
         free(recpayload);
         rectopic = NULL;
         recpayload = NULL;
      }
   }
   //must free topic
   static char* generaMqttTopic(char *prefix, char *suffix, char *flexfix)
   {
      int len=0;
      char *topic=NULL;
      if(prefix == NULL || iotinfo_module.productKey == NULL || iotinfo_module.imei < 100000000000000ULL)
         return NULL;
      if(suffix == NULL)
         len = strlen(prefix) + strlen(iotinfo_module.productKey) + AT_CMD_IMEI_IMSI_LEN + 4;
      else if(flexfix == NULL)
         len = strlen(prefix) + strlen(suffix) + strlen(iotinfo_module.productKey) + AT_CMD_IMEI_IMSI_LEN + 4;
      else 
         len = strlen(prefix) + strlen(suffix) + strlen(flexfix) + strlen(iotinfo_module.productKey) + AT_CMD_IMEI_IMSI_LEN + 4;
      
      if(len > AT_CMD_MQTT_MAX_TOPIC_LEN)
         return NULL;
      topic = (char*)malloc(len);
      if(topic == NULL)
         return NULL;
      
      if(suffix == NULL)
         len = snprintf(topic, len, "%s%s/%lld", prefix, iotinfo_module.productKey, iotinfo_module.imei);
      else if(flexfix == NULL)
         len = snprintf(topic, len, "%s%s/%lld%s", prefix, iotinfo_module.productKey, iotinfo_module.imei, suffix);
      else 
         len = snprintf(topic, len, "%s%s/%lld%s%s", prefix, iotinfo_module.productKey, iotinfo_module.imei, suffix, flexfix);
   
      if(len > AT_CMD_MQTT_MAX_TOPIC_LEN)
      {
         free(topic);
         topic = NULL;
      }
      ESP_LOGI(ATTAG, "%s success [%s] len:%d", __FUNCTION__, topic, strlen(topic) );
      return topic;
   }
   
   
   int subMqttTopic(unsigned char topicnum, char *flexfix)
   {
      int ret=-1;
      char *topic=NULL;
      // for(int i=0; i<MQTT_TOPIC_SUB_END; i++)
      {   
         mqtttopic_current = topicnum;
         // if(mqtttopic_current > MQTT_TOPIC_SUB_END)
         //    topicnum = mqtttopic_current - 1;
         // ESP_LOGI(ATTAG, "%s in  [%d] :", __FUNCTION__, esp_get_free_heap_size());
         if(flexfix == NULL)
            flexfix = mqtt_tiopic[topicnum].flexfix;
         if(flexfix)
            topic = generaMqttTopic(mqtt_tiopic[topicnum].prefix, mqtt_tiopic[topicnum].suffix, flexfix);
         else
            topic = generaMqttTopic(mqtt_tiopic[topicnum].prefix, mqtt_tiopic[topicnum].suffix, NULL);
         if(topic)
         {
            if(atSubMqttTopic(AT_CMD_MQTT_QOS, topic) == 0)
               ret = 0;
            free(topic);
            topic = NULL;
         }
         // ESP_LOGI(ATTAG, "%s out  [%d] :", __FUNCTION__, esp_get_free_heap_size());
      }
      return ret;
   }
   int unSubMqttTopic(unsigned char topicnum, char *flexfix)
   {
      int ret=-1;
      char *topic=NULL;
      // for(int i=0; i<MQTT_TOPIC_SUB_END; i++)
      {   
         mqtttopic_current = topicnum;
         // if(mqtttopic_current > MQTT_TOPIC_SUB_END)
         //    topicnum = mqtttopic_current - 1;
         // ESP_LOGI(ATTAG, "%s in  [%d] :", __FUNCTION__, esp_get_free_heap_size());
         if(flexfix == NULL)
            flexfix = mqtt_tiopic[topicnum].flexfix;
         if(flexfix)
            topic = generaMqttTopic(mqtt_tiopic[topicnum].prefix, mqtt_tiopic[topicnum].suffix, flexfix);
         else
            topic = generaMqttTopic(mqtt_tiopic[topicnum].prefix, mqtt_tiopic[topicnum].suffix, NULL);
         if(topic)
         {
            if(atUnSubMqttTopic(AT_CMD_MQTT_DUP, topic) == 0)
               ret = 0;
            free(topic);
            topic = NULL;
         }
         // ESP_LOGI(ATTAG, "%s out  [%d] :", __FUNCTION__, esp_get_free_heap_size());
      }
      return ret;
   }
   int publishMqttTopic(unsigned char topicnum, char *flexfix, char *payload)
   {
      int ret=-1;
      char *topic=NULL;
      if(payload == NULL)
         return -1;
      // for(int i=0; i<MQTT_TOPIC_SUB_END; i++)
      {   
         mqtttopic_current = topicnum;
         // if(mqtttopic_current > MQTT_TOPIC_SUB_END)
         //    topicnum = mqtttopic_current - 1;
         // ESP_LOGI(ATTAG, "%s in  [%d] :", __FUNCTION__, esp_get_free_heap_size());
         if(flexfix == NULL)
            flexfix = mqtt_tiopic[topicnum].flexfix;
         if(flexfix)
            topic = generaMqttTopic(mqtt_tiopic[topicnum].prefix, mqtt_tiopic[topicnum].suffix, flexfix);
         else
            topic = generaMqttTopic(mqtt_tiopic[topicnum].prefix, mqtt_tiopic[topicnum].suffix, NULL);
         if(topic)
         {
            if(atPublishMqttTopic(topic, payload) == 0)
               ret = 0;
            free(topic);
            topic = NULL;
         }
         // ESP_LOGI(ATTAG, "%s out  [%d] :", __FUNCTION__, esp_get_free_heap_size());
      }
      return ret;
   }
   int mqttInit(void)
   {
      if(atMqttInfoInit(SECURE_MODE) == 0)
      {
         if(atMqttCallBackInit(iotMqttOperationCb, iotMqttReceiveCb) == 0)
         {
            ESP_LOGI(ATTAG, "%s success []", __FUNCTION__ );
            return atConnectMqtt(10);
         }
      }
      return -1;
   }
