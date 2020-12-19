
.. _program_listing_file_main_atprocess_atcmd.c:

Program Listing for File atcmd.c
================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_atprocess_atcmd.c>` (``main\atprocess\atcmd.c``)

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
   #include <string.h>
   #include "esp_log.h"
   #include "drv_at.h"
   #include "iotinfo.h"
   #include "atcmd.h"
   #include "athttp.h"
   #include "atmqtt.h"
   static const char *ATTAG = "atcmd";
   static QueueHandle_t atDataQueue=NULL;
   static atcmd_module_t atcmd_module;
   
   
   //releate with atcmd_uri_map_opt_t
   static unsigned short  atcmd_sendtimeout;
   atcmd_uri_map_opt_t atcmd_current=AT;
   atcmd_uri_map_t g_atcmd_uri_map_send;
   
   #define STR_NULL_BREAK(P)               do{ \
     if(P == NULL) { \
       ESP_LOGI(ATTAG, "%s STRNULL error", __FUNCTION__ );  \
       break;  \
     } \
   }while(0)
   static int hexStr2Num(char *str, int len)
   {
     int num=0;
     while(len)  {
       num = (num<<4) | ((*str > '9' ? *str+9 : *str) & 0x0F);
       str++;
       len--;
     }
     return num;
   }
   static int cellInfoValid(int mcc, int mnc, int lac, int cellid, int signal)
   {
     // mcc for 000-099、100-199 和 800-899 为保留码 MCC，Mobile Country Code，移动国家代码    000～999
     // MNC，Mobile Network Code，移动网络号码(中国移动为0，中国联通为1，中国电信为2)    00～99或000～999
     // LAC，Location Area Code，位置区域码    0001－FFFEH
     // CID，Cell Identity，基站编号   (跟踪区代码（Tracking Area Code，TAC）)   (0,65535) (0,268435455)
     // BSSS，Base station signal strength，基站信号强度   
     if(mcc < 99 || mcc > 999)
       return -1;
     else if(lac < 1 || lac > 0xFFFE)
       return -1;
     else if(cellid < 1 || cellid > 268435455 || cellid == 65535)
       return -1;
     return 0;
   }
   extern int operateCellInfo(char type, int mcc, int mnc, int lac, int cellid, int signal);
   static int updateMainCell( char* recbuf, int len)
   {
   // <System Mode>,<Operation Mode>,<MCC>-<MNC>,<TAC>,<SCellID>,<PCellID>,<Frequency Band>,<earfcn>,<dlbw>,<ulbw>,<RSRQ>,<RSRP>,<RSSI>,<RSSNR>
   // LTE,          Online,           460-00,     0x247C,   225360194,5,   EUTRAN-BAND40,   38950   ,5     ,0     ,30    ,68    ,65    ,28
   // +CPSI: <System Mode>,<Operation Mode>,<MCC>-<MNC>,<LAC>,<Cell ID>,<Frequency Band>,<PSC>,<Freq>,<SSC>,<EC/IO>,<RSCP>,<Qual>,<RxLev>,<TXPWR>
   //   AT+CPSI?
   // +CPSI: LTE,Online,460-00,0x247C,225360194,5,EUTRAN-BAND40,38950,5,0,30,68,65,28
   // OK
   // char testbuf[]="+CPSI: LTE,Online,460-00,0x247C,225360194,5,EUTRAN-BAND40,38950,5,0,30,68,65,28";
   // recbuf = testbuf;
     if(atcmd_module.csqRssi < 1 || atcmd_module.csqRssi > 31)
       return -1;
   
     char *str=NULL, *strtemp=NULL;
     str = strstr(recbuf, "+CPSI: ");
     if (str == NULL)
       return -1;
     str = str + strlen("+CPSI: ");
     if(memcmp(str, "LTE", strlen("LTE")) != 0 && memcmp(str, "GSM", strlen("GSM")) != 0 && memcmp(str, "WCDMA", strlen("WCDMA")) != 0)  
       return -1;
     str = strchr(str, ',');
     if(str == NULL)
       return -1;
     str = str + 1;
     if(memcmp(str, "Online", strlen("Online")) != 0)
       return -1;
   
     int mcc=0,mnc=0,lac=0,cellid=0,signal=0;
     str = strchr(str, ',');
     if(str == NULL)
       return -1;
     mcc = atoi(str+1);
     str = strchr(str, '-');
     if(str == NULL)
       return -1;
     mnc = atoi(str+1);
     str = strstr(str, ",0");
     if(str == NULL)
       return -1;
     str = str + 3;//,0x
     strtemp = strchr(str, ',');
     if(strtemp == NULL)
       return -1;
     lac = hexStr2Num(str, strtemp - str);
     cellid = atoi(strtemp+1);
     signal = atcmd_module.csqRssi * 2 - 113;
     if(cellInfoValid(mcc, mnc, lac, cellid, signal))
       return -1;
     operateCellInfo(1, mcc, mnc, lac, cellid, signal);
     ESP_LOGI(ATTAG, "%s mcc: %d ,mnc: %d ,lac: %d ,cellid: %d ,signal: %d", __FUNCTION__ , mcc, mnc, lac, cellid, signal);
     return 0;
   }
   static int updateOtherCell( char* recbuf, int len)
   {
   // +CNETCINONINFO: 0, MCC-MNC: 460-00,TAC: 9340,cellid: 233029463,rsrp: 38,rsrq: 14
   // +CNETCI: 1
   // char tf[]="+CNETCINONINFO: 0, MCC-MNC: 460-00,TAC: 9340,cellid: 233029463,rsrp: 38,rsrq: 14\r\n+CNETCINONINFO: 1,  MCC-MNC: 460-00,TAC: 13116,cellid: 60660547,rsrp: 42,rsrq: 28\r\n+CNETCI: 1";
   // recbuf = tf;
     int signal=0;
     char *str=NULL;
     str = strstr(recbuf, "+CNETCI: ");
     if (str == NULL)
       return -1;
     str = str + strlen("+CNETCI: ");
     signal = atoi(str);
     if(signal == 0)
       return -1;//not net update
     str = strstr(recbuf, "MCC-MNC:");
     if (str == NULL)
       return -1;
     int mcc=0,mnc=0,lac=0,cellid=0;
     operateCellInfo(4, mcc, mnc, lac, cellid, signal);
   
     do  {
       mcc=0,mnc=0,lac=0,cellid=0;
       mcc = atoi(str + strlen("MCC-MNC:") + 1);
       str = strchr(str + strlen("MCC-MNC:"), '-');
       if(str) {
         mnc = atoi(str + 1);
         str = strstr(str, "AC:");
         if(str) {
           lac = atoi(str + strlen("AC:"));
           str = strstr(str, "cellid:");
           if(str) {
             cellid = atoi(str + strlen("cellid:"));
             str = strstr(str, "rsrq:");
             if(str)   {
               signal = atoi(str + strlen("rsrq:"));
               signal = signal * 2 -113;
               ESP_LOGI(ATTAG, "%s mcc: %d ,mnc: %d ,lac: %d ,cellid: %d ,signal: %d", __FUNCTION__ , mcc, mnc, lac, cellid, signal);
               if(cellInfoValid(mcc, mnc, lac, cellid, signal) == 0)
                 operateCellInfo(3, mcc, mnc, lac, cellid, signal);
               str = strstr(str, "MCC-MNC:");
             }//signal
           }//cellid
         }//lac
       }//mnc
     } while(str);
   
     if(iotinfo_module.devinfo->otherlbsnum)
       return 0;
     else
       return -1;
   }
   static int updateCsq( char* recbuf, int len)
   {
     int tempnum=0;
     char *str=NULL;
     str = strstr(recbuf, AT_CMD_ATCSQ_RESPOND);
     if(str == NULL)
       return -1;
     str = strchr(str, ':');
     if(str == NULL)
       return -1;
     tempnum = atoi(str+1);
     if(tempnum >= 1 && tempnum <= 31) {
       atcmd_module.csqRssi = tempnum;
       if (atcmd_module.csqRssi < 10)
         iotinfo_module.error.errcsq = 1;
       else
         iotinfo_module.error.errcsq = 0;
     }
     else  {
       atcmd_module.csqRssi = 0;
       return -1;
     }
     str = strchr(str, ',');
     if(str == NULL)
       return -1;
     tempnum = atoi(str+1);
     if( tempnum >= 0x0f)
       atcmd_module.csqBer = 0x0f;
     else
       atcmd_module.csqBer = tempnum;
     if(atcmd_current == ATCSQ)
         ATSENDCMD_CHANGE(ATCNETCI1);
     iotinfo_module.error.errnosim = 0; 
     ESP_LOGI(ATTAG, "%s csqRssi: %d , csqRssi: %d", __FUNCTION__ , atcmd_module.csqRssi, atcmd_module.csqBer);
     return 0;
   }
   static int detectSimCard( char* recbuf, int len)
   {
     char *str=NULL;
     str = strstr(recbuf, "SIM REMOVED");
     if (str)
       iotinfo_module.error.errnosim = 1;
     // else
     //   iotinfo_module.error.errnosim = 0;
     return -1;
   }
   static void atPraseBaseOk( char* recbuf, int len)
   {
     char *str=NULL;
     ESP_LOGI(ATTAG, "%s atcmd_currents %d ", __FUNCTION__ , atcmd_current);//, strlen(AT_CMD_CUSTOMIDtest));
     switch (atcmd_current)
     {
     case AT:
         ATSENDCMD_CHANGE(ATIPR);
       break;
     case ATIPR:
         drvSetAtBaudrate(921600);
         vTaskDelay(1 / portTICK_PERIOD_MS);
         ATSENDCMD_CHANGE(ATE0);
       break;
     case ATE0:
         atcmd_module.atStatus = 1;
         ATSENDCMD_CHANGE(ATCGMR);
       break;
     case ATCGMR:
         ATSENDCMD_CHANGE(ATI);
       break;
     case ATI:
       str = strstr(recbuf, AT_CMD_ATI_VERSION_FLAG);
       if(str)
       {
         ESP_LOGI(ATTAG, "%s version : %s ", __FUNCTION__ , str);
         atcmd_module.atversion = atoi(str+strlen(AT_CMD_ATI_VERSION_FLAG));
         atcmd_module.atversion <<= 16;
         str = strchr(str, '.');
         STR_NULL_BREAK(str);
         len = atoi(str+1);
         atcmd_module.atversion = atcmd_module.atversion | (len << 8 );
         str = strchr(str+1, '.');
         STR_NULL_BREAK(str);
         len = atoi(str+1);
         atcmd_module.atversion = atcmd_module.atversion | len;
         
         str = strstr(recbuf, AT_CMD_ATI_IMEI_FLAG);
         STR_NULL_BREAK(str);
   unsigned long long temp = str2Imei(str+strlen(AT_CMD_ATI_IMEI_FLAG));
         if(temp < 100000000000000ULL)
         {
           if(g_atcmd_uri_map_send.retry)
             ESP_LOGI(ATTAG, "%s get IMEI error", __FUNCTION__ );
           // else   //error
         }
         else  {
           if(temp != iotinfo_module.imei) {
             iotinfo_module.imei = temp;
             iotinfo_module.iotStatus = IOT_NONE;
           }
           ATSENDCMD_CHANGE(ATCIMI);
         }
         ESP_LOGI(ATTAG, "%s atcmd_module.IMEI : %lld ", __FUNCTION__ , iotinfo_module.imei);
       }
       else
           ESP_LOGI(ATTAG, "%s get version error", __FUNCTION__ );
       ESP_LOGI(ATTAG, "%s atcmd_module.atversion : 0x%08X ", __FUNCTION__ , atcmd_module.atversion);
       break;
     case ATCIMI:
       iotinfo_module.imsi = str2Imei(recbuf+2);
       if(iotinfo_module.imsi < 100000000000000ULL)
       {
         if(g_atcmd_uri_map_send.retry)
           ESP_LOGI(ATTAG, "%s get imsi error", __FUNCTION__ );
         // else   //error
         // g_atcmd_uri_map_send.timeout = ATCMD_1S_TIMEOUT;
         // g_atcmd_uri_map_send.retry = ATCMD_MIN_RETRY;
         iotinfo_module.error.errnosim = 1; 
       }
       else  {
         iotinfo_module.error.errnosim = 0; 
         ATSENDCMD_CHANGE(ATCSQ);
       }
         
       ESP_LOGI(ATTAG, "%s iotinfo_module.imsi : %lld ", __FUNCTION__ , iotinfo_module.imsi);
       break;
     // case ATCSQ:
     //   if(updateCsq(recbuf, len) == 0)
     //     ATSENDCMD_CHANGE(ATCNETCI1);
     //   iotinfo_module.error.errnosim = 0; 
     //   break;
     case ATCNETCI1:
         ATSENDCMD_CHANGE(ATCREG);
       break;
     case ATCREG:
         ATSENDCMD_CHANGE(ATCEREG);
       break;
     case ATCEREG:
       str = strstr(recbuf, AT_CMD_ATCEREG_RESPOND);
       if (str)
       {
         str = strchr(str, ':');
         STR_NULL_BREAK(str);
         len = atoi(str+1);
         if(len != 2)          break;
         str = strchr(str, ',');
         STR_NULL_BREAK(str);
         len = atoi(str+1);
         if(len == 1 || len == 5)
           atcmd_module.reg4gStatus = len;
         else
           break;
         
         iotinfo_module.error.errsim = 0; 
         // atSendCmdChange(0, NULL, 0, 0, 0);
     // iotinfo_module.iotStatus = IOT_NONE;//test
         if(iotinfo_module.iotStatus == IOT_NONE)
           atCmdStartGetIotInfo();
           // atCmdStartGetOtaInfo();
         else {
           atSendCmdChange(0, NULL, 0, 0, 0);
           iotinfo_module.iotStatus = IOT_SAVEBASEINFO;
         }
         ESP_LOGI(ATTAG, "%s reg4gStatus: %d", __FUNCTION__ , atcmd_module.reg4gStatus);
       }
       else  {
         iotinfo_module.error.errsim = 1; 
         ESP_LOGI(ATTAG, "%s get ATCSQ error", __FUNCTION__ );
       }
       break;
     
     default:
       ESP_LOGI(ATTAG, "%s not atcmd_current", __FUNCTION__ );
       break;
     }
   }
   static void atPraseBaseError( char* recbuf, int len)
   {
     ESP_LOGI(ATTAG, "%s Read ERROR bytes", __FUNCTION__ );
     // atSendCmd(0);
     // if( atcmd_current == ATIPR && atcmd_module.atStatus )  {
     // char *str=NULL;
     //   str = strstr(recbuf, "+CEREG:");
     //   if(str) {
     //     len = atoi(str + strlen("+CEREG:"));
     //     if(len == 1 || len == 5)
     //       ATSENDCMD_CHANGE(ATE0);
     //   }
     // }
   }
   static void atPraseBaseCmd(char* recbuf, int len)
   {
     if( ATCMDRESPONDOK(recbuf, len) )
       atPraseBaseOk(recbuf, len);
     else//receive not right
       atPraseBaseError(recbuf, len);
   }
   
   static void atUartReceive(char* recbuf, int len)
   {
     int ret=-1;
     //maybe need copy data
     if(recbuf == NULL || len < 2)
       return;
     if(len < 256)
       ESP_LOGI(ATTAG, "Read %d bytes: '%s'  ", len, recbuf);
     else
       ESP_LOGI(ATTAG, "Read %d bytes:   ", len);
     // ESP_LOGI(ATTAG, "lld %lld ", str2Imei(recbuf));
     // ESP_LOG_BUFFER_HEXDUMP(ATTAG, recbuf, len, ESP_LOG_INFO);
     atcmd_module.uartStatus = 1;
     // if(atcmd_module.atCmdChange == AT_NONE)
     {
       
   // extern void fzReceiveUpgradeMsg(char * rbuf, int len);
   //       fzReceiveUpgradeMsg(0,0);
   //       return;
   // extern void fzReceiveRrpcMsg(char * rbuf, int len, const char* messageId);
   //   fzReceiveRrpcMsg(recbuf, len, "123");
   //   return;
       
       ret = updateCsq(recbuf, len);
       if(ret)
         ret = updateMainCell(recbuf, len);
       if(ret)
         ret = updateOtherCell(recbuf, len);
   
       if(ret) {
         if(atcmd_current < AT_BASE_MAX) {
           detectSimCard(recbuf, len);
           atPraseBaseCmd(recbuf, len);
         }
         else if(atcmd_current < AT_HTTP_MAX)
           atPraseHttpCmd(recbuf, len);
         else if(atcmd_current < AT_MQTT_MAX)
           atPraseMqttCmd(recbuf, len);
       }
     }
   }
   static unsigned short querytime=0;
   static void setQueryTime(unsigned int time)//10ms
   {
     if(time == 0)
       querytime = 0;
     else  {
       if(querytime < 1500)  //csq
         querytime = 0;
       else if(querytime < 3000) //cpsi
         querytime = 1500;
       else if(querytime < 4500) //CNETCI
         querytime = 3000;
     }
   }
   static void atPrasePro(void *arg)
   {
     char* bufTemp=NULL;
     stQueueMsg event;
     while (1) 
     {
       // esp_task_wdt_feed();
       if(xQueueReceive(atDataQueue, (void * )&event, 10 / portTICK_PERIOD_MS) )
       {
         ESP_LOGI(ATTAG, "atPrasePro xQueueReceive begin %d\tbytes %d", event.start, event.len);
         if(drvAtCanReadData(event.start, event.len) == 0)
         {
           // portENTER_CRITICAL(); 
           bufTemp = (char*) realloc(bufTemp, event.len+1);
           if(bufTemp == NULL)
             ESP_LOGI(ATTAG, "buftemp null");
           else  {
             if(drvAtReadData(bufTemp, event.start, event.len) == 0) {
               bufTemp[event.len] = 0;
               atUartReceive(bufTemp, event.len);
             }
             free(bufTemp);
             bufTemp = NULL;
           }
           // portEXIT_CRITICAL();  
         }
       }
       else 
       {
         if(++atcmd_module.atSendtime > 120)
           atcmd_module.atSendtime = 120;
         if(g_atcmd_uri_map_send.timeout && --g_atcmd_uri_map_send.timeout == 0)
           atSendCmd(0);
         if(g_atcmd_uri_map_send.retry == 0 && g_atcmd_uri_map_send.timeout == 0)  {
           if (atcmd_module.atStatus == 0)  
             iotinfo_module.error.errmcu = 1; 
           else  {
             if(atcmd_current == ATCIMI)
               iotinfo_module.error.errnosim = 1; 
   //           else if(atcmd_current == ATHTTPACTION)  {
   // extern int atOtaQuit(char type);
   //             fzRespondUgrade();
   //           }
             else if(++querytime == 1500)  {
               if(iotinfo_module.iotStatus != IOT_CONNECTMQTT)
                 querytime = 0;
               sendAtCmd(AT_CMD_ATCSQ_SEND, strlen(AT_CMD_ATCSQ_SEND));
             }
             else if(iotinfo_module.iotStatus == IOT_CONNECTMQTT)  {
               if(querytime == 3000) 
                 sendAtCmd(AT_CMD_ATCPSI_SEND, strlen(AT_CMD_ATCPSI_SEND));
               else if(querytime == 4500) {
                 querytime = 0;
                 sendAtCmd(AT_CMD_ATCNETCI_SEND, strlen(AT_CMD_ATCNETCI_SEND));
               }
             }
           }
         }
       }
       // vTaskDelay(10 / portTICK_PERIOD_MS);
     }
     vTaskDelete(NULL);
   }
   
   void atSendCmd(bool first)
   {
     if(g_atcmd_uri_map_send.retry)  {
       if(--g_atcmd_uri_map_send.retry == 0) {
         ESP_LOGE(ATTAG, "%s sendcmd last %s len %d timeout %d", __FUNCTION__ , g_atcmd_uri_map_send.sendcmd, strlen(g_atcmd_uri_map_send.sendcmd), atcmd_sendtimeout);
       }
       else  {
         g_atcmd_uri_map_send.timeout = atcmd_sendtimeout;
         if(g_atcmd_uri_map_send.sendcmd)  {
           ESP_LOGI(ATTAG, "%s sendcmd [%d] %s len %d timeout %d", __FUNCTION__ , g_atcmd_uri_map_send.retry, g_atcmd_uri_map_send.sendcmd, strlen(g_atcmd_uri_map_send.sendcmd), atcmd_sendtimeout);
           sendAtCmd(g_atcmd_uri_map_send.sendcmd, strlen(g_atcmd_uri_map_send.sendcmd));
         }
       }
     }
     // else  {
     //   if(g_atcmd_uri_map_send.timeout && --g_atcmd_uri_map_send.timeout == 0)
     // }
   }
   int sendAtCmd(const char* data, int len)
   {
       int txBytes=0;
       atcmd_module.atSendtime = 0;
     // ESP_LOGW(ATTAG, "send %d bytes: [%s]  ", len, data);
       txBytes = drvAtSendData(data, len);
       return txBytes;
   }
   int atMallocCmd(char cmdstate, char *sendcmdin, char *receiveprefixin)
   {
     int len;
     len = strlen(sendcmdin)+1;
     g_atcmd_uri_map_send.sendcmd = (char*) realloc(g_atcmd_uri_map_send.sendcmd, len);
     if (g_atcmd_uri_map_send.sendcmd == NULL)
     {
       g_atcmd_uri_map_send.retry    = 0;
       g_atcmd_uri_map_send.timeout  = 0;
       return  -1;
     }
     len = strlen(receiveprefixin)+1;
     g_atcmd_uri_map_send.receiveprefix = (char*)realloc(g_atcmd_uri_map_send.receiveprefix, len);
     if (g_atcmd_uri_map_send.receiveprefix == NULL)
     {
       free( g_atcmd_uri_map_send.sendcmd);
       g_atcmd_uri_map_send.sendcmd  = NULL;
       g_atcmd_uri_map_send.retry    = 0;
       g_atcmd_uri_map_send.timeout  = 0;
       return  -1;
     }
     return 0;
   
   }
   int atSendCmdChange(char cmdstate, char *sendcmd, char *receiveprefix, unsigned short timeout, unsigned short retry)
   {
     if(sendcmd == NULL && timeout == 0 && retry == 0) {
       g_atcmd_uri_map_send.retry      = 0;
       g_atcmd_uri_map_send.timeout    = 0;
       // free(g_atcmd_uri_map_send.sendcmd);
       // g_atcmd_uri_map_send.sendcmd = NULL;
       // free(g_atcmd_uri_map_send.receiveprefix);
       // g_atcmd_uri_map_send.receiveprefix = NULL;
       return 0;
     }
     else if(sendcmd == NULL || timeout < 1 || retry < 1)
       return -1;
     atcmd_current                   = cmdstate;
     atcmd_sendtimeout               = timeout;
     g_atcmd_uri_map_send.retry      = retry;
     g_atcmd_uri_map_send.timeout    = timeout;
     
     memcpy(g_atcmd_uri_map_send.sendcmd, sendcmd, strlen(sendcmd)+1);
     memcpy(g_atcmd_uri_map_send.receiveprefix, receiveprefix, strlen(receiveprefix)+1);
     setQueryTime(1);
     if(ATOTAREAD == atcmd_current) 
       g_atcmd_uri_map_send.timeout = 5;
     else if(ATE0 == atcmd_current) 
       g_atcmd_uri_map_send.timeout = 20;
     else  
       atSendCmd(1);
     return 0;
   }
   int atCmdInit(void)
   {
     memset((void*)&iotinfo_module, 0, sizeof(iotinfo_module_t));
     memset((void*)&atcmd_module, 0, sizeof(atcmd_module_t));
     iotinfo_module.at_module = &atcmd_module;
     ESP_LOGI(ATTAG, "%s getout  [%d] :", __FUNCTION__, esp_get_free_heap_size());
     atcmd_current = AT;
     // iotinfo_module.imei = 860461046399475ULL;
     if( atMallocCmd(atcmd_current, AT_CMD_AT_SEND, AT_CMD_AT_RESPOND) )
     {
       ESP_LOGI(ATTAG, "atCmdInit set send cmd fail");
       return -1;
     }
     drvAtInit(atUartReceive, &atDataQueue);
     atSendCmdChange(atcmd_current, AT_CMD_AT_SEND, AT_CMD_AT_RESPOND, AT_CMD_AT_TIMEOUT, AT_CMD_AT_RETRY);
     ESP_LOGI(ATTAG, "init");
     return 0;
   }
   
   void atCmdPro(void)
   {
     drvAtProcess();
     xTaskCreate(atPrasePro, "atPrasePro", 1024*3, NULL, configMAX_PRIORITIES, NULL);
   }
