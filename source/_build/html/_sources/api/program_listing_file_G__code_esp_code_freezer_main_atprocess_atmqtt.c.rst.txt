
.. _program_listing_file_G__code_esp_code_freezer_main_atprocess_atmqtt.c:

Program Listing for File atmqtt.c
=================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_atprocess_atmqtt.c>` (``G:\code\esp\code\freezer\main\atprocess\atmqtt.c``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /* WiFi station Example
   
      This example code is in the Public Domain (or CC0 licensed, at your option.)
   
      Unless required by applicable law or agreed to in writing, this
      software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
      CONDITIONS OF ANY KIND, either express or implied.
   */
   #include <stdio.h>
   #include <string.h>
   #include "esp_system.h"
   #include "esp_log.h"
   #include "iotinfo.h"
   #include "atcmd.h"
   #include "atmqtt.h"
   static const char *ATTAG = "atmqtt";
   
   #pragma pack(1)
   typedef struct {
     void (*IotMqttOperatoinCb)(unsigned char status);
     void (*IotMqttReceiveCb)(char *topic, char *payload);
     unsigned int      mqttClienNum    : 2;
     unsigned int      qos             : 2;
     unsigned int      dup             : 2;
     unsigned int      retain          : 2;
   } atmqtt_module_t;
   #pragma pack()
   static atmqtt_module_t atmqtt_module;
   
   int atMqttCallBackInit(void (*IotMqttOperatoinCb)(unsigned char), void (*IotMqttReceiveCb)(char *, char *))
   {
     if(IotMqttOperatoinCb == NULL || IotMqttReceiveCb == NULL )
       return -1;
     atmqtt_module.IotMqttOperatoinCb  = IotMqttOperatoinCb;
     atmqtt_module.IotMqttReceiveCb    = IotMqttReceiveCb;
     return 0;
   }
   int atMqttInfoInit(char secureMode)
   {
     iotinfo_module.secureMode = secureMode;
     atmqtt_module.mqttClienNum = AT_CMD_MQTT_CLIENT_NUM;
     if(getMqttInfo(&iotinfo_module) ==0 )
     {
       ESP_LOGI(ATTAG, "%s getMqttInfo success", __FUNCTION__ );
     }
     return 0;
   }
   static int atSetMqttAccq( const char* clientid, unsigned char securemode)
   {
     int ret=0;
     char *sendbuf=NULL;
     unsigned char clientnum = atmqtt_module.mqttClienNum;
     if(clientnum > AT_CMD_MQTT_CLIENT_MAXNUM || securemode > AT_CMD_MQTT_SECURE_MAXNUM || clientid == NULL )
       return -1;
     sendbuf = (char *)malloc( strlen(AT_CMD_ATCMQTTACCQ_SEND) + strlen(clientid));
     if(sendbuf == NULL)
       return -1;
     if(securemode == MODE_TCP_DIRECT_PLAIN)
       ret = snprintf(sendbuf, 128, AT_CMD_ATCMQTTACCQ_SEND, clientnum, clientid, 0);
     else
       ret = snprintf(sendbuf, 128, AT_CMD_ATCMQTTACCQ_SEND, clientnum, clientid, 1);
     if(ret < 128)
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATCMQTTACCQ [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATCMQTTACCQ, sendbuf, AT_CMD_ATCMQTTACCQ_RESPOND) == 0)  
         atSendCmdChange(ATCMQTTACCQ, sendbuf, AT_CMD_ATCMQTTACCQ_RESPOND, AT_CMD_ATCMQTTACCQ_TIMEOUT, AT_CMD_ATCMQTTACCQ_RETRY); 
       else
         ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   static int atSetMqttConnect(unsigned int keepalive, unsigned char cleansession, iotinfo_module_t *pmqtt_module)
   {
     int ret=0;
     char *sendbuf=NULL;
     unsigned char clientnum = atmqtt_module.mqttClienNum;
     if(clientnum > AT_CMD_MQTT_CLIENT_MAXNUM || cleansession > AT_CMD_ATCMQTTCONNECT_CLEANSEESION_MAX || keepalive > AT_CMD_ATCMQTTCONNECT_KEEPALIVE_MAX 
                                               || pmqtt_module->iotDomain == NULL || pmqtt_module->userName == NULL 
                                               || pmqtt_module->passWord == NULL || pmqtt_module->iotPort <= 0 )
       return -1;
     sendbuf = (char *)malloc( AT_CMD_ATCMQTTCONNECT_SEND_MAXLEN );
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, AT_CMD_ATCMQTTCONNECT_SEND_MAXLEN, AT_CMD_ATCMQTTCONNECT_SEND, clientnum, 
                               pmqtt_module->productKey, pmqtt_module->iotDomain, pmqtt_module->iotPort, 
                               keepalive, cleansession, pmqtt_module->userName, pmqtt_module->passWord);
     if(ret < AT_CMD_ATCMQTTCONNECT_SEND_MAXLEN)
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATCMQTTCONNECT ", __FUNCTION__);
       if( atMallocCmd(ATCMQTTCONNECT, sendbuf, AT_CMD_ATCMQTTCONNECT_RESPOND) == 0)  
         atSendCmdChange(ATCMQTTCONNECT, sendbuf, AT_CMD_ATCMQTTCONNECT_RESPOND, AT_CMD_ATCMQTTCONNECT_TIMEOUT, AT_CMD_ATCMQTTCONNECT_RETRY); 
       else
           ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   static int atSubMqttConnect(void)
   {
     int ret=0;
     char *sendbuf=NULL;
     unsigned char clientnum = atmqtt_module.mqttClienNum;
     if(clientnum > AT_CMD_MQTT_CLIENT_MAXNUM )
       return -1;
     sendbuf = (char *)malloc( strlen(AT_CMD_ATCMQTTSUB_SEND) );
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, strlen(AT_CMD_ATCMQTTSUB_SEND), AT_CMD_ATCMQTTSUB_SEND, clientnum);
     if(ret < strlen(AT_CMD_ATCMQTTSUB_SEND))
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATCMQTTSUB [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATCMQTTSUB, sendbuf, AT_CMD_ATCMQTTSUB_RESPOND) == 0)  
         atSendCmdChange(ATCMQTTSUB, sendbuf, AT_CMD_ATCMQTTSUB_RESPOND, AT_CMD_ATCMQTTSUB_TIMEOUT, AT_CMD_ATCMQTTSUB_RETRY);
       else
         ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   static int atUnSubMqttConnect(void)
   {
     int ret=0;
     char *sendbuf=NULL;
     unsigned char clientnum = atmqtt_module.mqttClienNum;
     if(clientnum > AT_CMD_MQTT_CLIENT_MAXNUM )
       return -1;
     sendbuf = (char *)malloc( strlen(AT_CMD_ATCMQTTUNSUB_SEND) );
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, strlen(AT_CMD_ATCMQTTUNSUB_SEND), AT_CMD_ATCMQTTUNSUB_SEND, clientnum);
     if(ret < strlen(AT_CMD_ATCMQTTUNSUB_SEND))
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATCMQTTUNSUB [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATCMQTTUNSUB, sendbuf, AT_CMD_ATCMQTTUNSUB_RESPOND) == 0)  
         atSendCmdChange(ATCMQTTUNSUB, sendbuf, AT_CMD_ATCMQTTUNSUB_RESPOND, AT_CMD_ATCMQTTUNSUB_TIMEOUT, AT_CMD_ATCMQTTUNSUB_RETRY); 
       else
         ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   static int atPubMqttTopicPayload(char *payload)
   {
     int ret=0;
     char *sendbuf=NULL;
     unsigned char clientnum = atmqtt_module.mqttClienNum;
     ret = strlen(payload);
     if(clientnum > AT_CMD_MQTT_CLIENT_MAXNUM || payload == NULL || ret < 1 || ret > AT_CMD_MQTT_MAX_PAYLOAD_LEN)
       return -1;
     ret = strlen(AT_CMD_ATCMQTTPAYLOAD_SEND) + 4;
     sendbuf = (char *)malloc( ret );
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, ret, AT_CMD_ATCMQTTPAYLOAD_SEND, clientnum, strlen(payload));
     if(ret < strlen(AT_CMD_ATCMQTTPAYLOAD_SEND) + 4)
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATCMQTTPAYLOAD [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATCMQTTPAYLOAD, sendbuf, payload) == 0)  
       {
         g_atcmd_uri_map_send.wait       = ATCMD_WAIT_INPUT; 
         atSendCmdChange(ATCMQTTPAYLOAD, sendbuf, payload, AT_CMD_ATCMQTTPAYLOAD_TIMEOUT, AT_CMD_ATCMQTTPAYLOAD_RETRY); 
       }
       else
           ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   static int atPublishMqttConnect(unsigned char qos, unsigned char retain, unsigned char dup, unsigned char timeout)
   {
     int ret=0;
     char *sendbuf=NULL;
     unsigned char clientnum = atmqtt_module.mqttClienNum;
     if(clientnum > AT_CMD_MQTT_CLIENT_MAXNUM || qos > AT_CMD_MQTT_QOS_MAXNUM || dup > AT_CMD_MQTT_DUP_MAXNUM || retain > AT_CMD_MQTT_RETAIN_MAXNUM 
           || timeout < AT_CMD_ATCMQTTPUB_TIMEOUT_MINI || retain > AT_CMD_ATCMQTTPUB_TIMEOUT_MAX )
       return -1;
     sendbuf = (char *)malloc( strlen(AT_CMD_ATCMQTTPUB_SEND) );
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, strlen(AT_CMD_ATCMQTTPUB_SEND), AT_CMD_ATCMQTTPUB_SEND, clientnum, qos, timeout, retain, dup);
     if(ret < strlen(AT_CMD_ATCMQTTPUB_SEND))
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATCMQTTPUB [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATCMQTTPUB, sendbuf, AT_CMD_ATCMQTTPUB_RESPOND) == 0)  
         atSendCmdChange(ATCMQTTPUB, sendbuf, AT_CMD_ATCMQTTPUB_RESPOND, AT_CMD_ATCMQTTPUB_TIMEOUT, AT_CMD_ATCMQTTPUB_RETRY); 
       else
         ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   static int atReleaseMqttSession(void)
   {
     int ret=0;
     char *sendbuf=NULL;
     unsigned char clientnum = atmqtt_module.mqttClienNum;
     if(clientnum > AT_CMD_MQTT_CLIENT_MAXNUM)
       return -1;
     ret = strlen(AT_CMD_ATCMQTTREL_SEND);
     sendbuf = (char *)malloc( ret );
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, ret, AT_CMD_ATCMQTTREL_SEND, clientnum);
     if(ret < strlen(AT_CMD_ATCMQTTREL_SEND) )
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATCMQTTREL [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATCMQTTREL, sendbuf, AT_CMD_ATCMQTTREL_RESPOND) == 0)  
         atSendCmdChange(ATCMQTTREL, sendbuf, AT_CMD_ATCMQTTREL_RESPOND, AT_CMD_ATCMQTTREL_TIMEOUT, AT_CMD_ATCMQTTREL_RETRY); 
       else
           ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   static void atPraseMqttReceive( char* recbuf, int len)
   {
     char *str=NULL, *str1=NULL;
     static char *topic=NULL, *payload=NULL;
     ESP_LOGI(ATTAG, "%s in  [%d] ", __FUNCTION__, esp_get_free_heap_size());
     int topiclen=0, payloadlen=0;
     // str = strstr(recbuf, AT_CMD_MQTT_RECEIVE_END);
     // if(atmqtt_module.IotMqttReceiveCb && str) {
     // if(len < 6)
     //   return;
   
       str   = strstr(recbuf, AT_CMD_MQTT_RECEIVE_START);
       str1  = strstr(recbuf+2, "\r\n");
       if(atmqtt_module.IotMqttReceiveCb && str && str1){
     ESP_LOGI(ATTAG, "%s Read OK bytes", __FUNCTION__ );
         str = strchr(str, ',');
         if(str && str < str1){
           topiclen  = atoi(str + 1);
           str       = strchr(str+1, ',');
     ESP_LOGI(ATTAG, "%s Read OK topiclen [%d]", __FUNCTION__ , topiclen);
           if(str && str < str1){
             payloadlen  = atoi(str + 1);
     ESP_LOGI(ATTAG, "%s Read OK payloadlen [%d]", __FUNCTION__ , payloadlen);
             if(topiclen && payloadlen && topiclen < AT_CMD_MQTT_MAX_TOPIC_LEN)
             {
               str   = strchr(str+1, ',');
               if(str) {
                 str1  = strstr(str, "\r\n");
                 if(str && str < str1){
     // ESP_LOGI(ATTAG, "%s Read OK topic", __FUNCTION__ );
                   len  = atoi(str + 1);
                   if(len == topiclen)
                   {
     ESP_LOGI(ATTAG, "%s Read OK topic len:%d", __FUNCTION__ , topiclen);
                     topic = (char *)realloc(topic, topiclen + 1);
     ESP_LOGI(ATTAG, "%s Read OK 32 topic", __FUNCTION__ );
                     if(topic){
                       payload = (char *)realloc(payload, payloadlen + 1);
                       if(payload == NULL){
                         free(topic);
                         topic = NULL;
                         return;
                       }
                     }//get topic
                     else
                       return;
                     
                     memcpy(topic, str1+2, topiclen);
                     *(topic + topiclen ) = '\0';
     ESP_LOGI(ATTAG, "%s Read OK topic [%s]", __FUNCTION__, topic );
                     str = strchr(str1, ',');
                     if(str){
                       len = atoi(str + 1);
                       str1  = strstr(str, "\r\n");
                       if(str1)  {
                         topiclen = 0;
                         while(len && payloadlen > topiclen){
                           memcpy(payload+topiclen, str1+2, len);
                           topiclen += len;
                           len = 0;
                           str  = strstr(str1, "+CMQTTRXPAYLOAD:");
                           if(str) {
                             str = strchr(str, ',');
                             if(str) {
                               len = atoi(str + 1);
                               str1  = strstr(str, "\r\n");
                             }
                           }
                         }
   
                         // if(len == payloadlen) {
                         //   if(payload)
                         //     memcpy(payload, str1+2, payloadlen);
                         // }//equ payloadlen
                         // else if(len && len < payloadlen)
                         // {
   
                         // }
                         *(payload + payloadlen ) = '\0';
                         if(atmqtt_module.IotMqttReceiveCb)
                           atmqtt_module.IotMqttReceiveCb(topic, payload);
                       }
                     }//payloadlen
                     
                     free(payload);
                     payload = NULL;
                     free(topic);
                     topic = NULL;
                   }// equ topiclen
                 }// topic len end
               }//topic len
             }//both len
           }//,
         }//,
       }//AT_CMD_MQTT_RECEIVE_START
     // }//AT_CMD_MQTT_RECEIVE_END
     ESP_LOGI(ATTAG, "%s out  [%d] ", __FUNCTION__, esp_get_free_heap_size());
   }
   static void atPraseMqttOk( char* recbuf, int len)
   {
     ESP_LOGI(ATTAG, "%s Read OK bytes", __FUNCTION__ );
     switch (atcmd_current)
     {
       //start connect
     case ATCMQTTSTART:
       ATCMD_WAIT_SPECIAL(AT_CMD_ATCMQTTSTART_TIMEOUT);
       break;
     case ATCMQTTACCQ:
       atSetMqttConnect(iotinfo_module.devparam->heartTime, AT_CMD_MQTT_CLEAN_SESSION,&iotinfo_module);
       break;
     case ATCMQTTCONNECT:
       ATCMD_WAIT_SPECIAL(AT_CMD_ATCMQTTCONNECT_TIMEOUT);
       break;
       //disconnect stop
     case ATCMQTTDISC:
       ATCMD_WAIT_SPECIAL(AT_CMD_ATCMQTTDISC_TIMEOUT);
       break;
     case ATCMQTTREL:
       if(g_atcmd_uri_map_send.wait == ATCMD_WAIT_NONE)
       {
         ATSENDCMD_CHANGE(ATCMQTTSTOP);
       }
       break;
     case ATCMQTTSTOP:
       break;
       //subtopic
     case ATCMQTTSUBTOPIC:
       if(g_atcmd_uri_map_send.wait == ATCMD_WAIT_NONE)
         atSubMqttConnect();
       break;
     case ATCMQTTSUB:
       ATCMD_WAIT_SPECIAL(AT_CMD_ATCMQTTSUB_TIMEOUT);
       break;
       //unsubtopic
     case ATCMQTTUNSUBTOPIC:
       if(g_atcmd_uri_map_send.wait == ATCMD_WAIT_NONE)
         atUnSubMqttConnect();
       break;
     case ATCMQTTUNSUB:
         ATCMD_WAIT_SPECIAL(AT_CMD_ATCMQTTUNSUB_TIMEOUT);
       break;
       //pubtopic payload
     case ATCMQTTTOPIC:
       if(g_atcmd_uri_map_send.wait == ATCMD_WAIT_NONE)
         atPubMqttTopicPayload(g_atcmd_uri_map_send.payload);
       break;
     case ATCMQTTPAYLOAD:
       if(g_atcmd_uri_map_send.wait == ATCMD_WAIT_NONE)
         atPublishMqttConnect(AT_CMD_MQTT_QOS, AT_CMD_MQTT_DUP, AT_CMD_MQTT_RETAIN, AT_CMD_MQTT_TIMEOUT);
       break;
     case ATCMQTTPUB:
       ATCMD_WAIT_SPECIAL(AT_CMD_ATCMQTTPUB_TIMEOUT);
       break;
     
     default:
       ESP_LOGI(ATTAG, "%s not atcmd_current", __FUNCTION__ );
       break;
     }
   }
   
   static void specialMqttOperatoinCb( char* recbuf, int type)
   {
     char *str=NULL;
     if(MQTT_DISCONNECT_SUCCESS >= type && MQTT_CONNECT_SUCCESS <= type)
     str = strchr(recbuf, ',');
     if(str && atmqtt_module.IotMqttOperatoinCb) {
       if(atoi(str+1) == 0)  {
         ATSENDCMD_CHANGE_NULL();
         atmqtt_module.IotMqttOperatoinCb(type);
       }
       else  {
         atmqtt_module.IotMqttOperatoinCb(type + 1);
         ESP_LOGE(ATTAG, "IotMqttOperatoinCb fail [type:%d] Special bytes ", type);
       }
     }
   }
   
   static void atPraseMqttSpecial( char* recbuf, int len)
   {
     ESP_LOGI(ATTAG, "%s Read Special bytes", __FUNCTION__ );
     if( g_atcmd_uri_map_send.wait == ATCMD_WAIT_ADD)
     {
       if(atcmd_current == ATCMQTTSTART && ATCMD_WAIT_SPECIAL_CMP(recbuf, ATCMQTTSTART) )  {
         g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
         atSetMqttAccq(iotinfo_module.clientId, iotinfo_module.secureMode);
       }
       else if(atcmd_current == ATCMQTTCONNECT && ATCMD_WAIT_SPECIAL_CMP(recbuf, ATCMQTTCONNECT) ) {
         g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
         specialMqttOperatoinCb(recbuf, MQTT_CONNECT_SUCCESS);
       }
       else if(atcmd_current == ATCMQTTSUB && ATCMD_WAIT_SPECIAL_CMP(recbuf, ATCMQTTSUB) ) {
         g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
         specialMqttOperatoinCb(recbuf, MQTT_CONNECT_SUB_SUCCESS);
       }
       else if(atcmd_current == ATCMQTTUNSUB && ATCMD_WAIT_SPECIAL_CMP(recbuf, ATCMQTTUNSUB) ) {
         g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
         specialMqttOperatoinCb(recbuf, MQTT_CONNECT_UNSUB_SUCCESS);
       }
       else if(atcmd_current == ATCMQTTPUB && ATCMD_WAIT_SPECIAL_CMP(recbuf, ATCMQTTPUB) ) {
         g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
         specialMqttOperatoinCb(recbuf, MQTT_CONNECT_PUB_SUCCESS);
       }
       else if(atcmd_current == ATCMQTTDISC && ATCMD_WAIT_SPECIAL_CMP(recbuf, ATCMQTTDISC) ) {
         g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
         specialMqttOperatoinCb(recbuf, MQTT_DISCONNECT_SUCCESS);
         atReleaseMqttSession();
       }
     }
     else if( g_atcmd_uri_map_send.wait == ATCMD_WAIT_INPUT)
     {
       if(AT_CMD_MQTT_INPUTOK(recbuf)) {
         g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
         if( atcmd_current == ATCMQTTSUBTOPIC || atcmd_current == ATCMQTTUNSUBTOPIC || atcmd_current == ATCMQTTTOPIC )
           sendAtCmd(g_atcmd_uri_map_send.receiveprefix, strlen(g_atcmd_uri_map_send.receiveprefix));
         else if( atcmd_current == ATCMQTTPAYLOAD )
           sendAtCmd(g_atcmd_uri_map_send.payload, strlen(g_atcmd_uri_map_send.payload));
       }
     }
     // g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;
   }
   static void atPraseMqttError( char* recbuf, int len)
   {
     char *str=NULL;
     ESP_LOGI(ATTAG, "%s Read ERROR bytes []", __FUNCTION__ );
     if(recbuf == NULL)
       return;
     ESP_LOGI(ATTAG, "[%s] Read ERROR len %d", recbuf, len );
     str = strstr(recbuf, AT_CMD_MQTT_RECEIVE_START);
     if(str)
       atPraseMqttReceive(recbuf, len);
   }
   void atPraseMqttCmd(char* recbuf, int len)
   {
     ESP_LOGI(ATTAG, "%s atcmd_current : %d wait:%d ", __FUNCTION__ , atcmd_current, g_atcmd_uri_map_send.wait);
     if( ATCMDRESPONDOK(recbuf, len) )
       atPraseMqttOk(recbuf, len);
     else if( g_atcmd_uri_map_send.wait )
       atPraseMqttSpecial(recbuf, len);
     else//receive not right
       atPraseMqttError(recbuf, len);
   }
   
   int atConnectMqtt(unsigned char timeout)
   {
     if( atMallocCmd(ATCMQTTSTART, AT_CMD_ATCMQTTSTART_SEND, AT_CMD_ATCMQTTSTART_RESPOND) == 0)  
     {
       atSendCmdChange(ATCMQTTSTART, AT_CMD_ATCMQTTSTART_SEND, AT_CMD_ATCMQTTSTART_RESPOND, AT_CMD_ATCMQTTSTART_TIMEOUT, AT_CMD_ATCMQTTSTART_RETRY); 
       return 0;
     }
     return -1;
   }
   int atDisconnectMqtt(unsigned char timeout)
   {
     int ret=0;
     char *sendbuf=NULL;
     unsigned char clientnum = atmqtt_module.mqttClienNum;
     if(clientnum > AT_CMD_MQTT_CLIENT_MAXNUM || (timeout && (timeout < AT_CMD_ATCMQTTPUB_TIMEOUT_MINI || timeout > AT_CMD_ATCMQTTPUB_TIMEOUT_MAX)) )
       return -1;
     ret = strlen(AT_CMD_ATCMQTTDISC_SEND) + 2;
     sendbuf = (char *)malloc( ret );
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, ret, AT_CMD_ATCMQTTDISC_SEND, clientnum, timeout);
     if(ret < strlen(AT_CMD_ATCMQTTDISC_SEND) + 2)
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATCMQTTDISC [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATCMQTTDISC, sendbuf, AT_CMD_ATCMQTTDISC_RESPOND) == 0)  
         atSendCmdChange(ATCMQTTDISC, sendbuf, AT_CMD_ATCMQTTDISC_RESPOND, AT_CMD_ATCMQTTDISC_TIMEOUT, AT_CMD_ATCMQTTDISC_RETRY); 
       else
           ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   int atSubMqttTopic(unsigned char qos, char *topic)
   {
     int ret=0;
     char *sendbuf=NULL;
     unsigned char clientnum = atmqtt_module.mqttClienNum;
     ret = strlen(topic);
     if(clientnum > AT_CMD_MQTT_CLIENT_MAXNUM || qos > AT_CMD_MQTT_QOS_MAXNUM || topic == NULL || ret < 1 || ret > AT_CMD_MQTT_MAX_TOPIC_LEN)
       return -1;
     ret = strlen(AT_CMD_ATCMQTTSUBTOPIC_SEND) + 1;
     sendbuf = (char *)malloc( ret );
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, ret, AT_CMD_ATCMQTTSUBTOPIC_SEND, clientnum, strlen(topic), qos);
     if(ret < strlen(AT_CMD_ATCMQTTSUBTOPIC_SEND) + 1)
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATCMQTTSUBTOPIC [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATCMQTTSUBTOPIC, sendbuf, topic) == 0)  
       {
         g_atcmd_uri_map_send.wait       = ATCMD_WAIT_INPUT; 
         atSendCmdChange(ATCMQTTSUBTOPIC, sendbuf, topic, AT_CMD_ATCMQTTSUBTOPIC_TIMEOUT, AT_CMD_ATCMQTTSUBTOPIC_RETRY); 
         ESP_LOGI(ATTAG, "%s wait [%d]", __FUNCTION__ , g_atcmd_uri_map_send.wait);
       }
       else
           ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   int atUnSubMqttTopic(unsigned char dup, char *topic)
   {
     int ret=0;
     char *sendbuf=NULL;
     unsigned char clientnum = atmqtt_module.mqttClienNum;
     ret = strlen(topic);
     if(clientnum > AT_CMD_MQTT_CLIENT_MAXNUM || dup > AT_CMD_MQTT_DUP_MAXNUM || topic == NULL || ret < 1 || ret > AT_CMD_MQTT_MAX_TOPIC_LEN)
       return -1;
     ret = strlen(AT_CMD_ATCMQTTUNSUBTOPIC_SEND) + 1;
     sendbuf = (char *)malloc( ret );
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, ret, AT_CMD_ATCMQTTUNSUBTOPIC_SEND, clientnum, strlen(topic), dup);
     if(ret < strlen(AT_CMD_ATCMQTTUNSUBTOPIC_SEND) + 1)
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATCMQTTUNSUBTOPIC [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATCMQTTUNSUBTOPIC, sendbuf, topic) == 0)  
       {
         g_atcmd_uri_map_send.wait       = ATCMD_WAIT_INPUT; 
         atSendCmdChange(ATCMQTTUNSUBTOPIC, sendbuf, topic, AT_CMD_ATCMQTTUNSUBTOPIC_TIMEOUT, AT_CMD_ATCMQTTUNSUBTOPIC_RETRY); 
       }
       else
           ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   int atPublishMqttTopic(char *topic, char *payload)
   {
     int ret=0;
     char *sendbuf=NULL;
     unsigned char clientnum = atmqtt_module.mqttClienNum;
     ret = strlen(topic);
     if(clientnum > AT_CMD_MQTT_CLIENT_MAXNUM || topic == NULL  || payload == NULL || ret < 1 || ret > AT_CMD_MQTT_MAX_TOPIC_LEN)
       return -1;
     ret = strlen(payload);
     if(ret < 1 || ret > AT_CMD_MQTT_MAX_PAYLOAD_LEN)
     {
       ESP_LOGE(ATTAG, "%s ATCMQTTTOPIC max len [%d]", __FUNCTION__ , ret);
       return -1;
     }
     ret = strlen(AT_CMD_ATCMQTTTOPIC_SEND) + 4;
     sendbuf = (char *)malloc( ret );
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, ret, AT_CMD_ATCMQTTTOPIC_SEND, clientnum, strlen(topic));
     if(ret < strlen(AT_CMD_ATCMQTTSUBTOPIC_SEND) + 4)
     {
       ret=1;
       ESP_LOGI(ATTAG, "%s ATCMQTTTOPIC [%s]", __FUNCTION__ , sendbuf);
       g_atcmd_uri_map_send.payload = (char *)realloc(g_atcmd_uri_map_send.payload, strlen(payload)+1 );
       if(g_atcmd_uri_map_send.payload)
       {
         memcpy(g_atcmd_uri_map_send.payload, payload, strlen(payload)+1);
         if( atMallocCmd(ATCMQTTTOPIC, sendbuf, topic) == 0)  
         {
           ESP_LOGI(ATTAG, "%s ATCMQTTTOPIC payload [%s]", __FUNCTION__ , g_atcmd_uri_map_send.payload);
           g_atcmd_uri_map_send.wait       = ATCMD_WAIT_INPUT; 
           atSendCmdChange(ATCMQTTTOPIC, sendbuf, topic, AT_CMD_ATCMQTTSUBTOPIC_TIMEOUT, AT_CMD_ATCMQTTSUBTOPIC_RETRY); 
           ret=0;
         }
       }
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   
