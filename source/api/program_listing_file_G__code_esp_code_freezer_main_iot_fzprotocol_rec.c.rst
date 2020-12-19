
.. _program_listing_file_G__code_esp_code_freezer_main_iot_fzprotocol_rec.c:

Program Listing for File fzprotocol_rec.c
=========================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_iot_fzprotocol_rec.c>` (``G:\code\esp\code\freezer\main\iot\fzprotocol_rec.c``)

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
   #include "mqtttopic.h"
   #include "fzprotocol.h"
   #include "cmdprase.h"
   #include "iotinfo.h"
   static const char *ATTAG = "fzProtocol_REC";
   
   // [14:08:45.582]收←◆
   // +CMQTTRXSTART: 0,39,338
   // +CMQTTRXTOPIC: 0,39
   // /shadow/get/a14XBNg6pCp/864424042341104
   // +CMQTTRXPAYLOAD: 0,256
   // {"method":"control","payload":{"state":{"desired":{"20gw00":"30g101"},"reported":{"60gw00":"0","20gw00":"30g100","60gw06":"0"}},"metadata":{"desired":{"20gw00":{"timestamp":1603433324}},"reported":{"60gw00":{"timestamp":1603433281},"20gw00":{"timestamp":16
   // +CMQTTRXPAYLOAD: 0,82
   // 03433281},"60gw06":{"timestamp":1603433281}}}},"timestamp":1603433324,"version":0}
   // +CMQTTRXEND: 0
   
   void fzReceiveShadowMsg(char * rbuf, int len)
   {
      char *str=NULL, *strtemp=NULL;
      cJSON *shadowroot = NULL, *root1 = NULL, *root2 = NULL;
      str = strchr(rbuf, '{');
      if(str)
      {
         // ESP_LOGI(ATTAG, "%s receive [%s]", __FUNCTION__ , str);
         strtemp = strrchr(str, '}');
         if(strtemp)
         {
            *(strtemp + 1) = '\0';
            shadowroot = cJSON_Parse(str);
            if(shadowroot && cJSON_IsObject(shadowroot))
            {
               root1 = cJSON_GetObjectItem(shadowroot, "method");
               if(root1 && cJSON_IsString(root1))
               {
                  strtemp = cJSON_GetStringValue(root1);
                  ESP_LOGI(ATTAG, "%s method [%s]", __FUNCTION__ , strtemp);
                  if(memcmp(strtemp, "control", strlen("control")) == 0)// control
                  {
                     root1 = cJSON_GetObjectItem(shadowroot, "payload");
               // ESP_LOGI(ATTAG, "%s receive [%s]", __FUNCTION__ , cJSON_Print(root1));
                     if(root1 && cJSON_IsObject(root1))  {
                        root2 = cJSON_GetObjectItem(root1, "state");
                        if(root2 && cJSON_IsObject(root2))  {
                           root1 = cJSON_GetObjectItem(root2, "desired");
                           root2 = cJSON_GetObjectItem(shadowroot, "version");
                           if(root1 && root2 && cJSON_IsObject(root1) && cJSON_IsNumber(root2) )  {
                              ESP_LOGI(ATTAG, "%s desired success", __FUNCTION__ );
                              shadowRemoteControl(root1, root2->valueint + 1);
                           }
                           else  {//fail{
                              ESP_LOGI(ATTAG, "%s desired fail", __FUNCTION__ );
                           }
                        }
                     }
                  }
                  else if(memcmp(strtemp, "reply", strlen("reply")) == 0)//device update report
                  {
                     root1 = cJSON_GetObjectItem(shadowroot, "payload");
                     if(root1 && cJSON_IsObject(root1))  {
                        root2 = cJSON_GetObjectItem(root1, "status");
                        if(root2 && cJSON_IsString(root2))  {
                           strtemp = cJSON_GetStringValue(root2);
                           if(memcmp(strtemp, "success", strlen("success")) == 0)   {// success
                              ESP_LOGI(ATTAG, "%s reply success", __FUNCTION__ );
                           }
                           else  {//fail
                              ESP_LOGI(ATTAG, "%s reply fail", __FUNCTION__ );
                           }
                           
                        }
                     }
                  }
               }
            }
            else
               ESP_LOGI(ATTAG, "%s receive fail", __FUNCTION__ );
            
            cJSON_Delete(shadowroot);
         }
         // if(ret)
      }
      else
         ESP_LOGI(ATTAG, "%s no json", __FUNCTION__ );
   }
   
   void fzReceiveRrpcMsg(char * rbuf, int len, const char* messageId)
   {
     char *str=NULL, *strtemp=NULL;
     cJSON *rrpcroot = NULL, *root1 = NULL;
     str = strchr(rbuf, '{');
     if(str)   {
         ESP_LOGI(ATTAG, "%s receive [%s]", __FUNCTION__ , str);
         strtemp = strrchr(str, '}');
         if(strtemp)
         {
            *(strtemp + 1) = '\0';
            rrpcroot = cJSON_Parse(str);
            if(rrpcroot && cJSON_IsObject(rrpcroot))
            {
               root1 = cJSON_GetObjectItem(rrpcroot, "action");
               if(root1 && cJSON_IsString(root1))  {//action
                  strtemp = cJSON_GetStringValue(root1);
                  ESP_LOGI(ATTAG, "%s rrpc action [%s]", __FUNCTION__ , strtemp);
                  if(memcmp(strtemp, "attrQuery", strlen("attrQuery")) == 0)   {//attrQuery
                     root1 = cJSON_GetObjectItem(rrpcroot, "param");
                     if(rrpcAttrQueryParam(messageId, root1) == 0)
                        ESP_LOGI(ATTAG, "%s receive attrQuery", __FUNCTION__ );
                     else
                        ESP_LOGI(ATTAG, "%s prase attrQuery fail", __FUNCTION__ );
                  }
                  else if(memcmp(strtemp, "alarmQuery", strlen("alarmQuery")) == 0)   {//alarmQuery
                     root1 = cJSON_GetObjectItem(rrpcroot, "param");
                     if(root1 && cJSON_IsObject(root1))  {
                        ESP_LOGI(ATTAG, "%s receive alarmQuery", __FUNCTION__ );
                        fzRespondAlarmQuery(messageId, 0);
                     }
                  }
               }
            }
            else
               ESP_LOGI(ATTAG, "%s receive fail", __FUNCTION__ );
            
            cJSON_Delete(rrpcroot);
         }
      }
      else
         ESP_LOGI(ATTAG, "%s no json", __FUNCTION__ );
   }
   
   // #pragma pack(1)
   // typedef struct {
   //    char           *sn;
   //    char           *md5;
   //    char           *url;
   //    char           *version;
   //    // int            type;
   // } upgrade_info_t;
   // #pragma pack()
   // static upgrade_info_t upgrade_info;
   extern void atCmdStartGetOtaInfo(void);
   void fzReceiveUpgradeMsg(char * rbuf, int len)
   { 
   // char te[]="{\r\n\"sn\" : \"1234567\", \r\n\"upgradeType\" : \"1\", \r\n\"checkType\" : \"1\", \r\n\"checkValue\" : \"1\", \r\n\"Softwaretype\" : \"1.1.0\", \r\n\"upgradeURL\" : \"http://47.242.12.154:8080/freezer.bin\", \r\n\"force\" : \"1\" \r\n}";
   // rbuf = te;
   // len = 20;
     char *str=NULL, *strtemp=NULL, *ver=NULL;
     cJSON *uproot = NULL, *root1 = NULL;
     if(rbuf == NULL || len < 10)   {
         fzRespondUgrade(ERR_PARAMETER_ILLEGAL);
         return;
     }
     str = strchr(rbuf, '{');
     if(str)
     {
         // ESP_LOGI(ATTAG, "%s receive [%s]", __FUNCTION__ , str);
         strtemp = strrchr(str, '}');
         if(strtemp) {
            *(strtemp + 1) = '\0';
            uproot = cJSON_Parse(str);
            if(uproot && cJSON_IsObject(uproot))   {
   
               root1 = cJSON_GetObjectItem(uproot, "sn");
               if(root1 && cJSON_IsString(root1))  {
                  strtemp = cJSON_GetStringValue(root1);
                  iotinfo_module.upgradeSn = (char*) realloc(iotinfo_module.upgradeSn, strlen(strtemp)+1);
                  if(iotinfo_module.upgradeSn == NULL)   {
                     fzRespondUgrade(ERR_PARAMETER_ILLEGAL);
                     goto errsn;
                  }
                  memcpy(iotinfo_module.upgradeSn, strtemp, strlen(strtemp)+1);
                  *(iotinfo_module.upgradeSn + strlen(strtemp)) = 0;
               }
   
               root1 = cJSON_GetObjectItem(uproot, "upgradeType");
               if(root1 && cJSON_IsString(root1))  {
                  strtemp = cJSON_GetStringValue(root1);
                  if( *strtemp != '1' )   {//upgradeType
                     fzRespondUgrade(ERR_UPGRADE_UNKNOWN_UPGRADE_TYPE);
                     goto errupgradetype;
                  }
               }
   
               root1 = cJSON_GetObjectItem(uproot, "checkType");
               if(root1 && cJSON_IsString(root1))  {
                  strtemp = cJSON_GetStringValue(root1);
                  if( *strtemp != '1' )   {//upgradeType
                     fzRespondUgrade(ERR_UPGRADE_UNKNOWN_CHECKOUT_TYPE);
                     goto errchecktype;
                  }
               }
   
               root1 = cJSON_GetObjectItem(uproot, "Softwaretype");
               if(root1 && cJSON_IsString(root1))  {
                  strtemp = cJSON_GetStringValue(root1);
                  ver = (char*) realloc(ver, strlen(strtemp)+1);
                  if(ver == NULL)
                     goto errsofttype;
                  memcpy(ver, strtemp, strlen(strtemp)+1);
               }
   
               // root1 = cJSON_GetObjectItem(uproot, "checkValue");
               // if(root1 && cJSON_IsString(root1))  {
               //    strtemp = cJSON_GetStringValue(root1);
               //    upgrade_info.md5 = (char*) realloc(upgrade_info.md5, strlen(strtemp)+1);
               //    if(upgrade_info.md5 == NULL)
               //       goto errmd5;
               //    memcpy(upgrade_info.md5, strtemp, strlen(strtemp)+1);
               // }
   
               root1 = cJSON_GetObjectItem(uproot, "upgradeURL");
               if(root1 && cJSON_IsString(root1))  {
                  strtemp = cJSON_GetStringValue(root1);
                  if(memcmp(strtemp, "http://", strlen("http://"))) {
                     fzRespondUgrade(ERR_UPGRADE_URL_INVALID);
                     goto errurl;
                  }
                  strtemp = strtemp + strlen("http://");
                  iotinfo_module.upgradeUrl = (char*) realloc(iotinfo_module.upgradeUrl, strlen(strtemp)+1);
                  if(iotinfo_module.upgradeUrl == NULL) {
                     fzRespondUgrade(ERR_UPGRADE_URL_INVALID);
                     goto errurl;
                  }
                  memcpy(iotinfo_module.upgradeUrl, strtemp, strlen(strtemp)+1);
                  *(iotinfo_module.upgradeUrl + strlen(strtemp)) = 0;
               }
   
               root1 = cJSON_GetObjectItem(uproot, "force");
               if(root1 && cJSON_IsString(root1))  {
                  strtemp = cJSON_GetStringValue(root1);
                  if( *strtemp == '1' ){//upgrade force
                     ESP_LOGW(ATTAG, "%s force upgrade url [%s] upgradeSn:[%s]  version:[%s]", __FUNCTION__ , iotinfo_module.upgradeUrl, iotinfo_module.upgradeSn, ver);
                     // atCmdStartGetOtaInfo();
                  }
                  else if( *strtemp == '0' ){
                     if(memcmp(ver, PRODUCT_SOFT_VERSION, strlen(PRODUCT_SOFT_VERSION)) == 0 )   {
                        fzRespondUgrade(ERR_UPGRADE_NOT_ALLOW);
                        goto errforce;
                     }
                     else  {//begin update
                        ESP_LOGW(ATTAG, "%s normal upgrade url [%s] upgradeSn:[%s] version:[%s]", __FUNCTION__ , iotinfo_module.upgradeUrl, iotinfo_module.upgradeSn, ver);
                        // atCmdStartGetOtaInfo();
                     }
                  }
                  else  {
                     fzRespondUgrade(ERR_PARAMETER_ILLEGAL);
                     goto errforce;
                  }
               }
            }
            else
               ESP_LOGI(ATTAG, "%s receive fail", __FUNCTION__ );
            
            cJSON_Delete(uproot);
         }
      }
      else
         ESP_LOGI(ATTAG, "%s no json", __FUNCTION__ );
      return;
   
   errforce:
      free(iotinfo_module.upgradeUrl);
      iotinfo_module.upgradeUrl = NULL;
   errurl:
      // free(upgrade_info.md5);
      // upgrade_info.md5 = NULL;
   // errmd5:
      free(ver);
      ver = NULL;
   errsofttype:
   errchecktype:
   errupgradetype:
      free(iotinfo_module.upgradeSn);
      iotinfo_module.upgradeSn = NULL;
   errsn:
      cJSON_Delete(uproot);
   }
