
.. _program_listing_file_main_atprocess_athttp.c:

Program Listing for File athttp.c
=================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_atprocess_athttp.c>` (``main\atprocess\athttp.c``)

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
   #include "esp_log.h"
   #include "cJSON.h"
   #include "iotinfo.h"
   #include "atcmd.h"
   #include "athttp.h"
   static const char *ATTAG = "athttp";
   
   #define URL_TEST "47.242.12.154:8080/freezer.bin"
   // #define URL_TEST "47.242.12.154:8080/1.txt"
   
   #define OTA_READMODE            1
   
   #define OTA_NONE                0
   #define OTA_WORKING             1
   #define OTA_FAIL                2
   #define OTA_SUCCESS             3
   
   #define OTA_FIRST_PACK_SIZE     1024
   #define OTA_ONE_PACK_MAX        1024
   static char *otaBuf=NULL;
   static int otaStep=0;
   static int httpBodySize=0;
   int atOtaQuit(char type)
   {
     if(type == OTA_SUCCESS) {
       iotinfo_module.otaStatus = OTA_SUCCESS;
     }
     otaStep = 0;
     httpBodySize = 0;
     ATSENDCMD_CHANGE(ATHTTPTERM);
     return 0;
   }
   static int atGetOtaInfo(const char *url)
   {
     int ret=0;
     char *sendbuf=NULL;
     if(url == NULL )
       return -1;
     sendbuf = (char *)malloc(128);
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, 128, AT_CMD_ATOTAPARA_SEND, url);
     if(ret < 128)
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATOTAPARA [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATHTTPPARA, sendbuf, AT_CMD_ATOTAPARA_RESPOND) == 0)  
         atSendCmdChange(ATHTTPPARA, sendbuf, AT_CMD_ATOTAPARA_RESPOND, AT_CMD_ATOTAPARA_TIMEOUT, AT_CMD_ATOTAPARA_RETRY); 
       else
         ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   static int atGetOtaBody(unsigned short step)
   {
     int ret=0;
     char *sendbuf=NULL;
     ESP_LOGI(ATTAG, "%s otaStep:%d httpBodySize:%d", __FUNCTION__ , step, httpBodySize);
     if(httpBodySize < OTA_FIRST_PACK_SIZE || step == 0)
       return -1;
     sendbuf = (char *)malloc( AT_CMD_ATHTTPREAD_SEND_MAXLEN );
     if(sendbuf == NULL)
       return -1;
     if(step == 1) 
       ret = snprintf(sendbuf, AT_CMD_ATOTAREAD_SEND_MAXLEN, AT_CMD_ATOTAREAD_SEND, 0, OTA_FIRST_PACK_SIZE);
     else  {
       // if(httpBodySize < OTA_FIRST_PACK_SIZE + OTA_ONE_PACK_MAX * step) 
       //   ret = OTA_ONE_PACK_MAX;
       // else  
   #if (OTA_READMODE == 1)
         ret = OTA_FIRST_PACK_SIZE + OTA_ONE_PACK_MAX * (step-2);
       ret = snprintf(sendbuf, AT_CMD_ATOTAREAD_SEND_MAXLEN, AT_CMD_ATOTAREAD_SEND, ret, OTA_ONE_PACK_MAX);
   #elif
       ret = snprintf(sendbuf, AT_CMD_ATOTAREAD_SEND_MAXLEN, AT_CMD_ATOTAREAD_SEND, 0, OTA_ONE_PACK_MAX);
   #endif
     }
     if(ret < AT_CMD_ATOTAREAD_SEND_MAXLEN)
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATOTAREAD %d [%s]", __FUNCTION__ , step, sendbuf);
       if( atMallocCmd(ATOTAREAD, sendbuf, AT_CMD_ATOTAREAD_RESPOND) == 0)  {
         g_atcmd_uri_map_send.wait = ATCMD_WAIT_ADD; 
         atSendCmdChange(ATOTAREAD, sendbuf, AT_CMD_ATOTAREAD_RESPOND, AT_CMD_ATOTAREAD_TIMEOUT, AT_CMD_ATOTAREAD_RETRY); 
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
   extern int appOtaReceive(int step, char* buf, int buflen);
   static int atMergeOtaInfo( char* recbuf, int len)
   {
     char *str=NULL, *strtemp=NULL;
     // ESP_LOGI(ATTAG, "%s otapack data step:%d len:%d recbuf[%s]", __FUNCTION__ , otaStep, len, recbuf);
     str = strstr(recbuf, AT_CMD_ATOTAREAD_RESPOND);
     if(str)
     {
       // setReportTime(0);//set report clean
       int lentemp = atoi(str + strlen(AT_CMD_ATOTAREAD_RESPOND));
       if( lentemp == 0 || (lentemp == 1024 && len != 1059)  )
         return -1;
       len = lentemp;
       strtemp = strstr(str, "\r\n");
       *(strtemp + 2 + len) = 0; 
       ESP_LOGI(ATTAG, "%s otapack data otaStep:%d len:%d ", __FUNCTION__ , otaStep, len);
       if(otaStep) {
         if(otaStep == 1)  {
           otaBuf = (char *)realloc(otaBuf, OTA_ONE_PACK_MAX);
           if(otaBuf == NULL){
             ESP_LOGE(ATTAG, "%s ota buf error ", __FUNCTION__ );
             return -1;
           }
           if(len == OTA_FIRST_PACK_SIZE){
             memcpy(otaBuf, strtemp + 2, len);
             if(appOtaReceive(otaStep, otaBuf, len) == 0)
               atGetOtaBody(++otaStep);
           }
         }
         else if(otaStep > 1 && otaBuf) {
           if(httpBodySize > OTA_FIRST_PACK_SIZE + OTA_ONE_PACK_MAX * (otaStep - 1) ) {
             // memcpy(otatestbuf+OTA_FIRST_PACK_SIZE +OTA_ONE_PACK_MAX * (otaStep - 2), strtemp + 2, len);
             memcpy(otaBuf, strtemp + 2, len);
             if(appOtaReceive(otaStep, otaBuf, len) == 0)
               atGetOtaBody(++otaStep);
           } 
           else  {
             // memcpy(otatestbuf+OTA_FIRST_PACK_SIZE +OTA_ONE_PACK_MAX * (otaStep - 2), strtemp + 2, len);
             memcpy(otaBuf, strtemp + 2, len);
             ESP_LOGI(ATTAG, "%s otapack receive over step:%d httpBodySize:%d pack [%s] ", __FUNCTION__ , otaStep, httpBodySize, otaBuf);
             if(appOtaReceive(-1, otaBuf, len) == 0)  {
               g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
               atOtaQuit(OTA_SUCCESS);
             }
             else  {
               g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
               atOtaQuit(OTA_FAIL);
             }
             free(otaBuf);
             otaBuf = NULL;
           }
         }//otabuf
       }
       return 0;
     }//head
     else
       return -1;
   }
   
   static int atGetIotInfo(unsigned long long imei, const char* customid)
   {
     int ret=0;
     char *sendbuf=NULL;
     ESP_LOGI(ATTAG, "%s IMEI %lld", __FUNCTION__ , imei);
     if(imei < 100000000000000ULL || customid == NULL )
       return -1;
     sendbuf = (char *)malloc( strlen(AT_CMD_ATHTTPPARA_SEND) + AT_CMD_IMEI_IMSI_LEN + strlen(customid));
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, 128, AT_CMD_ATHTTPPARA_SEND, imei, customid);
     if(ret < 128)
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATHTTPPARA [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATHTTPPARA, sendbuf, AT_CMD_ATHTTPPARA_RESPOND) == 0)  
         atSendCmdChange(ATHTTPPARA, sendbuf, AT_CMD_ATHTTPPARA_RESPOND, AT_CMD_ATHTTPPARA_TIMEOUT, AT_CMD_ATHTTPPARA_RETRY); 
       else
         ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   static int atGetIotBody(unsigned short start, unsigned short end)
   {
     int ret=0;
     char *sendbuf=NULL;
     ESP_LOGI(ATTAG, "%s start:%d end:%d ", __FUNCTION__ , start, end);
     if(start >= end || end < 10 )
       return -1;
     sendbuf = (char *)malloc( AT_CMD_ATHTTPREAD_SEND_MAXLEN );
     if(sendbuf == NULL)
       return -1;
     ret = snprintf(sendbuf, AT_CMD_ATHTTPREAD_SEND_MAXLEN, AT_CMD_ATHTTPREAD_SEND, start, end);
     if(ret < 128)
     {
       ret=0;
       ESP_LOGI(ATTAG, "%s ATHTTPREAD [%s]", __FUNCTION__ , sendbuf);
       if( atMallocCmd(ATHTTPREAD, sendbuf, AT_CMD_ATHTTPPARA_RESPOND) == 0)  
         atSendCmdChange(ATHTTPREAD, sendbuf, AT_CMD_ATHTTPPARA_RESPOND, AT_CMD_ATHTTPPARA_TIMEOUT, AT_CMD_ATHTTPPARA_RETRY); 
       else
         ret=-1;
     }
     else
       ret=-1;
   
     free(sendbuf);
     sendbuf = NULL;
     return ret;
   }
   static int atGetHttpSize(char* recbuf, int len)
   {
     char *str=NULL;
     int statusCode=0;
     str = strchr(recbuf, ',');
     if(str)
     {
       httpBodySize = 0;
       statusCode = atoi(str+1);
       if(statusCode == 200)  {
         str = strchr(str+1, ',');
         if(str)
         {
           httpBodySize = atoi(str+1);
           if(httpBodySize)
           {
             if(iotinfo_module.otaStatus && httpBodySize < OTA_FIRST_PACK_SIZE)//out ota
               atOtaQuit(OTA_FAIL);
             else  {
               // ATSENDCMD_CHANGE(ATHTTPHEAD);
               if(iotinfo_module.otaStatus == OTA_WORKING)  {
                 otaStep = 1;
                 atGetOtaBody(otaStep);
               }
               else  {
                 atGetIotBody(0, httpBodySize);
                 httpBodySize = 0;
               }
             }
             ESP_LOGI(ATTAG, "%s statusCode:%d httpBodySize:%d ", __FUNCTION__, statusCode, httpBodySize);
             return 0;
           }//httpBodySize
         }//,
       }//statusCode
     }//,
     return -1;
   }
   static void atPraseHttpIotInfo( char* recbuf, int len)
   {
     char *str=NULL, *strtemp=NULL;
     cJSON *root = NULL, *root1 = NULL, *root2 = NULL;
     str = strchr(recbuf, '{');
     if(str)
     {
       ESP_LOGI(ATTAG,"JSON Parse str %s", str);
       strtemp = strstr(str, AT_CMD_ATHTTPREAD_RESPOND);
       if(strtemp)
       {
         *strtemp = '\0';
         ESP_LOGI(ATTAG,"JSON Parse strtemp %s", str);
         root = cJSON_Parse(str);
         if (root)// && cJSON_IsObject(root)) 
         {
           strtemp = cJSON_Print(root);
           ESP_LOGI(ATTAG,"JSON Parse root here %s", strtemp);
           cJSON_free(strtemp);
           strtemp = NULL;
           root1 = cJSON_GetObjectItem(root, AT_CMD_JSON_STATUS);
           if(root1 && cJSON_IsString(root1) && *(root1->valuestring) == '1')
           {
             root1 = cJSON_GetObjectItem(root, AT_CMD_JSON_DATA);
             if(root1 && cJSON_IsObject(root1))
             {
               strtemp = cJSON_Print(root1);
               ESP_LOGI(ATTAG,"JSON Parse data %s", strtemp);
               cJSON_free(strtemp);
               strtemp = NULL;
               root2 = cJSON_GetObjectItem(root1, AT_CMD_JSON_PRODUCTKEY);
               if(root2 && cJSON_IsString(root2) && strlen(root2->valuestring))
               {
                 str = cJSON_GetStringValue(root2);
                 iotinfo_module.productKey = (char*) realloc(iotinfo_module.productKey, strlen(root2->valuestring)+1);
                 if(iotinfo_module.productKey)
                   memcpy(iotinfo_module.productKey, root2->valuestring, strlen(root2->valuestring)+1);
                 else 
                   return;
               }//productKey
               else 
                 return;
               ESP_LOGI(ATTAG,"JSON Parse productKey %s", iotinfo_module.productKey);
   
               root2 = cJSON_GetObjectItem(root1, AT_CMD_JSON_DEVICESECRET);
               if(root2 && cJSON_IsString(root2) && strlen(root2->valuestring))
               {
                 iotinfo_module.deviceSecret = (char*) realloc(iotinfo_module.deviceSecret, strlen(root2->valuestring)+1);
                 if(iotinfo_module.deviceSecret)
                   memcpy(iotinfo_module.deviceSecret, root2->valuestring, strlen(root2->valuestring)+1);
                 else 
                   return;
               }//deviceSecret
               else 
                 return;
               ESP_LOGI(ATTAG,"JSON Parse deviceSecret %s", iotinfo_module.deviceSecret);
               root2 = cJSON_GetObjectItem(root1, AT_CMD_JSON_DOMAIN);
               if(root2 && cJSON_IsString(root2))
               {
                 strtemp = cJSON_GetStringValue(root2);
                 str = strchr(strtemp, ':');
                 iotinfo_module.iotPort = atoi(str+1);
                 len = str - strtemp + 1;
                 iotinfo_module.iotDomain = (char*) realloc(iotinfo_module.iotDomain, len);
                 if(iotinfo_module.iotDomain)
                 {
                   ESP_LOGI(ATTAG,"JSON Parse strtemp [%s]", strtemp);
                   memcpy(iotinfo_module.iotDomain, strtemp, len - 1 );
                   *(iotinfo_module.iotDomain + len-1) = '\0';
                 }
                 else 
                   return;
               }//domain
               else 
                 return;
               ESP_LOGI(ATTAG,"JSON Parse iotDomain [%s]:%d", iotinfo_module.iotDomain, iotinfo_module.iotPort);
               SaveInfoToFlash(strtemp);
               ATSENDCMD_CHANGE(ATHTTPTERM);
             }//data
           }//status
           cJSON_Delete(root);
         }//root
       }//AT_CMD_ATHTTPREAD_RESPOND
     }//{
   }
   // static void atPraseHttpHeadInfo( char* recbuf, int len)
   // {
   //   char* str=NULL;
   //   if(recbuf == NULL || len == 0)
   //     return;
   //   str = strstr(recbuf, "HTTP/1.1 200");
   //   if(str) {
   //     if(iotinfo_module.otaStatus == OTA_WORKING)  {
   //       otaStep = 1;
   //       atGetOtaBody(otaStep);
   //     }
   //     else  {
   //       atGetIotBody(0, httpBodySize);
   //       httpBodySize = 0;
   //     }
   //   }
   // }
   static void atPraseHttpOk( char* recbuf, int len)
   {
     switch (atcmd_current)
     {
     case ATHTTPINIT:
         if(iotinfo_module.otaStatus == OTA_WORKING) {
           atGetOtaInfo(iotinfo_module.upgradeUrl);//ATHTTPPARA
         }
         else
           atGetIotInfo(iotinfo_module.imei, AT_CMD_CUSTOMID);//ATHTTPPARA
       break;
     case ATHTTPPARA:
   #if (OTA_READMODE == 1)
         ATSENDCMD_CHANGE(ATREADMODE);
       break;
     case ATREADMODE:
         ATSENDCMD_CHANGE(ATHTTPACTION);
   #elif
         ATSENDCMD_CHANGE(ATHTTPACTION);
   #endif
       break;
     case ATHTTPACTION:
         ATCMD_WAIT_SPECIAL(AT_CMD_ATHTTPACTION_TIMEOUT);
       break;
     // case ATHTTPHEAD:
     //     atPraseHttpHeadInfo(recbuf, len);
     //   break;
     case ATHTTPREAD:
         ATCMD_WAIT_SPECIAL(AT_CMD_ATHTTPREAD_TIMEOUT);
       break;
       
     case ATOTAREAD:
         ATCMD_WAIT_SPECIAL(AT_CMD_ATOTAREAD_TIMEOUT);
       break;
   
     case ATHTTPTERM:
         if(iotinfo_module.otaStatus == OTA_SUCCESS)
           fzRespondUgrade(ERR_OK);
         else  {
           ATSENDCMD_CHANGE_NULL();
           iotinfo_module.iotStatus = IOT_SAVEBASEINFO;
         }
       break;
     
     default:
       ESP_LOGI(ATTAG, "%s not atcmd_current", __FUNCTION__ );
       break;
     }
   }
   static void atPraseHttpSpecial( char* recbuf, int len)
   {
     if( g_atcmd_uri_map_send.wait == ATCMD_WAIT_ADD)
     {
       // ESP_LOGI(ATTAG, "recbuf [%s] [%s] not %s", recbuf, AT_CMD_ATHTTPACTION_SEND+2, AT_CMD_ATHTTPACTION_SEND+1+AT_CMD_ATHTTPACTION_RESPOND_SPLEN );
       if(atcmd_current == ATHTTPACTION && ATCMD_WAIT_SPECIAL_CMP(recbuf, ATHTTPACTION) )  {
         g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
         atGetHttpSize(recbuf, len);
       }
       else if(atcmd_current == ATHTTPREAD && ATCMD_WAIT_SPECIAL_CMP(recbuf, ATHTTPREAD) ){
         g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
         atPraseHttpIotInfo(recbuf, len);
       }
       else if(atcmd_current == ATOTAREAD){// && memcmp(recbuf, ATCMDRESPONDOKSTR, strlen(ATCMDRESPONDOKSTR)) == 0 ){//ATCMD_WAIT_SPECIAL_CMP(recbuf, ATOTAREAD) ){
         // if(atMergeOtaInfo(recbuf, len) == 0)
         //   g_atcmd_uri_map_send.wait = ATCMD_WAIT_NONE;//only one receive
         atMergeOtaInfo(recbuf, len);
       }
       
     }
   }
   static void atPraseHttpError( char* recbuf, int len)
   {
     ESP_LOGI(ATTAG, "%s Read ERROR bytes", __FUNCTION__ );
     // atSendCmd(0);
     // if(atcmd_current == ATOTAREAD && g_atcmd_uri_map_send.wait != ATCMD_WAIT_ADD)
   }
   void atCmdStartGetIotInfo(void)
   {
     httpBodySize = 0;
     iotinfo_module.otaStatus = OTA_NONE;
     ATSENDCMD_CHANGE(ATHTTPINIT);
   }
   void atCmdStartGetOtaInfo(void)
   {
     httpBodySize = 0;
     iotinfo_module.otaStatus = OTA_WORKING;
     ATSENDCMD_CHANGE(ATHTTPINIT);
   }
   void atPraseHttpCmd(char* recbuf, int len)
   {
     ESP_LOGI(ATTAG, "%s atcmd_current : %d wait:%d", __FUNCTION__ , atcmd_current, g_atcmd_uri_map_send.wait);
     if( ATCMDRESPONDOK(recbuf, len) )
       atPraseHttpOk(recbuf, len);
     else if( g_atcmd_uri_map_send.wait )
       atPraseHttpSpecial(recbuf, len);
     else//receive not right
       atPraseHttpError(recbuf, len);
   }
