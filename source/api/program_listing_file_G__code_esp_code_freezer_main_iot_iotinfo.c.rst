
.. _program_listing_file_G__code_esp_code_freezer_main_iot_iotinfo.c:

Program Listing for File iotinfo.c
==================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_iot_iotinfo.c>` (``G:\code\esp\code\freezer\main\iot\iotinfo.c``)

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
   #include "drv_nvs_flash.h"
   #include "infra_sha256.h"
   #include "iotinfo.h"
   static const char *ATTAG = "iotinfo";
   static device_info_t devinfo;
   static device_param_t devparam;
   iotinfo_module_t iotinfo_module;
   extern int mqttInit(void);
   extern int operateHistoryTemp(int temp);
   
   
   #define UINT_MAX    (~0U)
   #define DEV_SIGN_CLIENT_ID_MAXLEN         (128)
   #define DEV_SIGN_SOURCE_MAXLEN            (200)
   #define DEV_SIGN_PASSWD_LEN               (32)
   #define DEV_SIGN_USERNAME_LEN             (32)
   /* use fixed timestamp */
   // __TIME__
   #define TIMESTAMP_VALUE             "2009231715"//"2524608000000"//__TIME__ //"1600052689"
   #define IOTX_SDK_VERSION            "3.0.1"
   #define CLIENTID_STRING             "%lld|timestamp=%s,secureMode=%d,signmethod=hmacsha256,gw=0,ext=0,_v=sdk-c-%s|"
   #define SIGN_STRING                 "clientId%llddeviceName%lldproductKey%stimestamp%s"
   
   static void _hex2str(uint8_t *input, uint16_t input_len, char *output)
   {
       char *zEncode = "0123456789ABCDEF";
       int i = 0, j = 0;
   
       for (i = 0; i < input_len; i++) {
           output[j++] = zEncode[(input[i] >> 4) & 0xf];
           output[j++] = zEncode[(input[i]) & 0xf];
       }
   }
   static int flashInfoInit(void)
   {
     unsigned long long   numdata=0;
     char* flashdata;
     //deviceSecret
     flashdata = (char*) malloc(IOTX_DEVICE_SECRET_LEN);
     if(flashdata == NULL)
       goto err1;
     if(halGetDeviceSecret(flashdata) <= 0)
       goto err1;
     ESP_LOGI(ATTAG, "%s deviceSecret:[%s]  ", __FUNCTION__, flashdata);
     if( strlen(flashdata) < 32 )
       goto err1;
     if(iotinfo_module.deviceSecret)
     {
       free(iotinfo_module.deviceSecret);
       iotinfo_module.deviceSecret = NULL;
     }
     iotinfo_module.deviceSecret = flashdata;
   
     //imei deviceName did
     flashdata = (char*) malloc(IOTX_DEVICE_NAME_LEN);
     if(flashdata == NULL)
       goto err2;
     if(halGetDeviceName(flashdata) <= 0)
       goto err2;
     numdata = str2Imei(flashdata);
     ESP_LOGI(ATTAG, "%s imei:[%s] %lld ", __FUNCTION__, flashdata, numdata);
     if(numdata < 100000000000000ULL)
       goto err2;
     iotinfo_module.imei = numdata;
     free(flashdata);
     flashdata = NULL;
   
     //productKey
     flashdata = (char*) malloc(IOTX_PRODUCT_KEY_LEN);
     if(flashdata == NULL)
       goto err3;
     if(halGetProductKey(flashdata) <= 0)
       goto err3;
     ESP_LOGI(ATTAG, "%s productKey:[%s]  ", __FUNCTION__, flashdata);
     if( strlen(flashdata) < 8 )
       goto err3;
     if(iotinfo_module.productKey)
     {
       free(iotinfo_module.productKey);
       iotinfo_module.productKey = NULL;
     }
     iotinfo_module.productKey = flashdata;
   
     //iotDomain
     flashdata = (char*) malloc(IOTX_MQTT_DOMAIN_LEN + 1);
     if(flashdata == NULL)
       goto err4;
     if(halGetIotMqttDomain(flashdata) <= 0)
       goto err4;
     ESP_LOGI(ATTAG, "%s iotDomain:[%s]  ", __FUNCTION__, flashdata);
     // if( flashdata = NULL )
     //   goto err3;
     if(iotinfo_module.iotDomain)
     {
       free(iotinfo_module.iotDomain);
       iotinfo_module.iotDomain = NULL;
     }
     iotinfo_module.iotDomain = flashdata;
     flashdata = strchr(iotinfo_module.iotDomain, ':');
     if(flashdata == NULL)
       goto err4;
     *flashdata = 0;
     iotinfo_module.iotPort = atoi(flashdata+1);
     if(iotinfo_module.iotPort <= 0)
       goto err4;
   
     //iotPort
     // flashdata = (char*) malloc(IOTX_MQTT_PORT_LEN);
     // if(flashdata == NULL)
     //   goto err5;
     // if(halGetIotMqttPort(flashdata) <= 0)
     //   goto err5;
     // numdata = atoi(flashdata);
     // ESP_LOGI(ATTAG, "%s iotPort:[%s] %lld ", __FUNCTION__, flashdata, numdata);
     // if( numdata <= 0 )
     //   goto err5;
     // iotinfo_module.iotPort = numdata;
     // free(flashdata);
     // flashdata = NULL;
   
     return 0;
   
   
   // err5:
   //   free(iotinfo_module.iotDomain);
   //   iotinfo_module.iotDomain = NULL;
   //   ESP_LOGI(ATTAG, "%s err5 [%d] :", __FUNCTION__, esp_get_free_heap_size());
   err4:
     free(iotinfo_module.productKey);
     iotinfo_module.productKey = NULL;
     // ESP_LOGI(ATTAG, "%s err4 [%d] :", __FUNCTION__, esp_get_free_heap_size());
   err3:
     // free(iotinfo_module.imei);
     iotinfo_module.imei = 0;
     // ESP_LOGI(ATTAG, "%s err3 [%d] :", __FUNCTION__, esp_get_free_heap_size());
   err2:
     free(iotinfo_module.deviceSecret);
     iotinfo_module.deviceSecret = NULL;
     // ESP_LOGI(ATTAG, "%s err2 [%d] :", __FUNCTION__, esp_get_free_heap_size());
   err1:
     free(flashdata);
     flashdata = NULL;
     // ESP_LOGI(ATTAG, "%s err1 [%d] :", __FUNCTION__, esp_get_free_heap_size());
     return -1;
   }
   
   
   int getMqttInfo(iotinfo_module_t *pmqtt_module)
   {
     int len;
     ESP_LOGI(ATTAG, "%s IMEI %s:%s %lld", __FUNCTION__ , __DATE__,  __TIME__,  pmqtt_module->imei);
     if(pmqtt_module == NULL || pmqtt_module->imei < 100000000000000ULL || pmqtt_module->secureMode > 3 )
       goto err0;
   
     //client id
     len = strlen(CLIENTID_STRING) + AT_CMD_IMEI_IMSI_LEN + strlen(TIMESTAMP_VALUE) + strlen(IOTX_SDK_VERSION);
     ESP_LOGI(ATTAG, "%s IMEI %d", __FUNCTION__ , len);
     pmqtt_module->clientId = (char*) realloc(pmqtt_module->clientId, len);
     if(pmqtt_module->clientId == NULL)
       goto err1;
     len = snprintf(pmqtt_module->clientId, DEV_SIGN_CLIENT_ID_MAXLEN, CLIENTID_STRING, 
                                             pmqtt_module->imei, TIMESTAMP_VALUE, pmqtt_module->secureMode, IOTX_SDK_VERSION);
     if( len >= DEV_SIGN_CLIENT_ID_MAXLEN)
       goto err1;
     ESP_LOGI(ATTAG, "%s clientId [%s] len %d ", __FUNCTION__ , pmqtt_module->clientId, strlen(pmqtt_module->clientId) );
   
     //sign source
     char *signsource= (char*) malloc(DEV_SIGN_SOURCE_MAXLEN);
     if(signsource == NULL)
       goto err2;
     char *sign_hex= (char*) malloc(DEV_SIGN_PASSWD_LEN);
     if(sign_hex == NULL)
       goto err3;
     pmqtt_module->passWord = (char*) realloc(pmqtt_module->passWord, DEV_SIGN_PASSWD_LEN*2+1);
     if(pmqtt_module->passWord == NULL)
       goto err4;
     len = snprintf(signsource, DEV_SIGN_SOURCE_MAXLEN, SIGN_STRING, 
                                             pmqtt_module->imei, pmqtt_module->imei, pmqtt_module->productKey, TIMESTAMP_VALUE);
     if( len >= DEV_SIGN_SOURCE_MAXLEN)
       goto err4;
     ESP_LOGI(ATTAG, "%s signsource [%s] len %d ", __FUNCTION__ , signsource, strlen(signsource) );
     utilsHmacSha256((uint8_t*)signsource, strlen(signsource), (uint8_t*)pmqtt_module->deviceSecret, strlen(pmqtt_module->deviceSecret), (uint8_t*)sign_hex);
     _hex2str((uint8_t*)sign_hex, DEV_SIGN_PASSWD_LEN, pmqtt_module->passWord);
     *(pmqtt_module->passWord + DEV_SIGN_PASSWD_LEN*2) = '\0';
     ESP_LOGI(ATTAG, "%s passWord [%s] len %d ", __FUNCTION__ , pmqtt_module->passWord, strlen(pmqtt_module->passWord) );
     free(sign_hex);
     sign_hex = NULL;
     free(signsource);
     signsource = NULL;
     
     len = AT_CMD_IMEI_IMSI_LEN + strlen(pmqtt_module->productKey) + 2;
     pmqtt_module->userName = (char*) realloc(pmqtt_module->userName, len);
     if(pmqtt_module->userName == NULL)
       goto err5;
     len = snprintf(pmqtt_module->userName, DEV_SIGN_USERNAME_LEN, "%lld&%s", 
                                             pmqtt_module->imei, pmqtt_module->productKey);
     if( len >= DEV_SIGN_USERNAME_LEN)
       goto err5;
     ESP_LOGI(ATTAG, "%s userName [%s] len %d ", __FUNCTION__ , pmqtt_module->userName, strlen(pmqtt_module->userName) );
   
     return 0;
   
   err5:
     free(pmqtt_module->userName);
     pmqtt_module->userName = NULL;
   err4:
     free(pmqtt_module->passWord);
     pmqtt_module->passWord = NULL;
   err3:
     free(sign_hex);
     sign_hex = NULL;
   err2:
     free(signsource);
     signsource = NULL;
   err1:
     free(pmqtt_module->clientId);
     pmqtt_module->clientId = NULL;
   err0:
     return -1;
   }
   int SaveInfoToFlash(char *domain)
   {
     if(domain == NULL)  {
       ESP_LOGI(ATTAG, "%s save param info ", __FUNCTION__  );
       return halSetDeviceParam((char *)iotinfo_module.devparam, sizeof(device_param_t));
     }
     else
     {
       halSetIotMqttDomain(domain);
       halSetProductKey(iotinfo_module.productKey);
       halSetDeviceSecret(iotinfo_module.deviceSecret);
       halSetNumDeviceName(iotinfo_module.imei);
       ESP_LOGI(ATTAG, "%s save iot info ", __FUNCTION__  );
     }
     return 0;
   }
   unsigned long long str2Imei(char* buf)
   {
     char * pbuf;
     unsigned long long ret=0,ret1=0;
     ret1 = atol(buf+8);
     if(ret1 < 1000000)
       return 0;
     pbuf= (char*) malloc(9);
     if(pbuf == NULL)
       return 0;
     memcpy(pbuf, buf, 8);
     *(pbuf + 8) = 0;
     ret = atol(pbuf);
     free(pbuf);
     pbuf = NULL;
     if(ret < 10000000)
       return 0;
     ret = ret*10000000 + ret1;
     return ret;
   }
   int iotInfoInit(void)
   {
     iotinfo_module.devinfo      = &devinfo;
     iotinfo_module.devparam     = &devparam;
     iotinfo_module.iotStatus    = IOT_NONE;
     iotinfo_module.powerOnTime  = 0;
     iotinfo_module.firstOn      = 1;
     ESP_LOGI(ATTAG, "%s comein[%d] ", __FUNCTION__, esp_get_free_heap_size());
     // for(int i =0 ; i < 10; i++)
     // {
     // ESP_LOGI(ATTAG, "%s comein[%d] ", __FUNCTION__, esp_get_free_heap_size());
     //   flashInfoInit();
     // ESP_LOGI(ATTAG, "%s comeout[%d] ", __FUNCTION__, esp_get_free_heap_size());
     // }
     if(flashInfoInit() == 0)
     {
       ESP_LOGI(ATTAG, "%s getflash success  [%d] domain [%s] : %d", __FUNCTION__, esp_get_free_heap_size(), 
                                                           iotinfo_module.iotDomain, iotinfo_module.iotPort);
       iotinfo_module.iotStatus = IOT_GETBASEINFO;//
     }
     
     ESP_LOGI(ATTAG, "%s after getflash heap %d", __FUNCTION__, esp_get_free_heap_size() );
     if(halGetDeviceParam((char *)iotinfo_module.devparam, sizeof(device_param_t)) <= 0)
     {//default param
       iotinfo_module.devparam->settemp = -23;
       iotinfo_module.devparam->showway     = 0;
       iotinfo_module.devparam->upbacklash = 20*10;
       iotinfo_module.devparam->downbacklash = 20*10;
       iotinfo_module.devparam->erroron = 70;
       iotinfo_module.devparam->erroroff = 10;
       iotinfo_module.devparam->tempoffset = 0;
       iotinfo_module.devparam->settemphigh = -15;
       iotinfo_module.devparam->settemplow = -28;
       iotinfo_module.devparam->hightempalarm = -179;
       iotinfo_module.devparam->lowtempalarm = -241;
       iotinfo_module.devparam->heartTime = DEFAULT_MQTT_HEART_FREQUENCY;
       iotinfo_module.devparam->reportTime = DEFAULT_ATTRS_FREQUENCY;
       ESP_LOGI(ATTAG, "%s default param heap %d", __FUNCTION__, esp_get_free_heap_size() );
     }
     
     // iotinfo_module.imei = 860461046399476ULL;
     // iotinfo_module.devparam->showway     = 2;
     // iotinfo_module.devparam->upbacklash = 50;
     // iotinfo_module.devparam->downbacklash = 60;
     // iotinfo_module.devparam->reportTime = 60;
     // iotinfo_module.iotStatus = IOT_NONE;//test
   
     ESP_LOGI(ATTAG, "%s param  settemp:%d tempoffset:%d upbacklash:%d downbacklash:%d erroron:%d erroroff:%d lowtempalarm:%d hightempalarm:%d", __FUNCTION__, 
                                                                   iotinfo_module.devparam->settemp, iotinfo_module.devparam->tempoffset, 
                                                                   iotinfo_module.devparam->upbacklash, iotinfo_module.devparam->downbacklash, 
                                                                   iotinfo_module.devparam->erroron, iotinfo_module.devparam->erroroff, 
                                                                   iotinfo_module.devparam->lowtempalarm, iotinfo_module.devparam->hightempalarm);
     
     if(bleIbeaconInit() != 0)
       iotinfo_module.error.errble = 1;
     WifiScanInit();
     wifiScanStart(1);
     iotinfo_module.error.errwifi = 0;
     ESP_LOGI(ATTAG, "%s getout  [%d] ", __FUNCTION__, esp_get_free_heap_size());
   
     return 0;
   }
   static unsigned int attrReportTime=0;
   void setReportTime(unsigned int time)//10ms
   {
     attrReportTime = 0;
   }
   extern void fzReceiveUpgradeMsg(char * rbuf, int len);
   void iotInfoPro(void)//10ms
   {
     static unsigned short errbit=0;
     if(iotinfo_module.powerOnTime < UINT_MAX)
       iotinfo_module.powerOnTime++;
     else
       iotinfo_module.powerOnTime = UINT_MAX;
       
     if(iotinfo_module.iotStatus == IOT_SAVEBASEINFO) {
       ESP_LOGI(ATTAG,"iotStatus %d", iotinfo_module.iotStatus);
       
       if(mqttInit() == 0) {
         iotinfo_module.iotStatus = IOT_SETMQTTINFO;
       }
       attrReportTime = 0;
     }
     else if(iotinfo_module.iotStatus == IOT_CONNECTMQTT) {
       if(iotinfo_module.otaStatus != 1 && errbit != iotinfo_module.error.errbit)  {
         errbit = iotinfo_module.error.errbit;
         fzReportAlarmInfo();
       }
       else  {
         if(++attrReportTime > UINT_MAX)
           attrReportTime = UINT_MAX;
         // else if(iotinfo_module.otaStatus == 0 && attrReportTime == 500 )//30s
         //   fzReceiveUpgradeMsg(0,0);
         else if(attrReportTime % 3000 == 1 )//30s
           wifiScanStart(1);
         else if(attrReportTime % 30000 == 2)//5min
           operateHistoryTemp(iotinfo_module.realTemp/10);
   
         if(attrReportTime > iotinfo_module.devparam->reportTime * 60 * 100) { 
           if(iotinfo_module.otaStatus)  {
             attrReportTime = 0;
             ESP_LOGI(ATTAG,"otaStatus 1min attr");
           }else
           if(iotinfo_module.at_module->atSendtime >= 100)  {
             attrReportTime = 0;
             fzReportAttrsInfo(1);
           }
         }
       }
     }
   }
   
