
.. _program_listing_file_G__code_esp_code_freezer_main_iot_cmdprase.h:

Program Listing for File cmdprase.h
===================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_iot_cmdprase.h>` (``G:\code\esp\code\freezer\main\iot\cmdprase.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __CMD_PARASE__
   #define __CMD_PARASE__
   
   #define PRODUCT_SOFT_VERSION        "1.1.0"
   //send
   
   typedef enum {
       QUERY_IMSI                   ,
       QUERY_WIFI_MAC               ,
   
       SRQ_POWERON                  ,
       SQ_FROZEN_TEMP               ,
       QUERY_REPORT_FREQUENCY       ,
       QUERY_MQTT_HEARTTIME         ,
   
       RQ_FROZEN_TEMP               ,
       RQ_FROZEN_TEMP_REVERSAVE     ,
       RQ_HUAN_TEMP_REVERSAVE       ,
       RQ_HISTORY_TEMP              ,
       RQ_MAIN_CELL                 ,
       RQ_OTHER_CELL                ,
       RQ_SCAN_WIFI_MAC_NUM         ,
       RQ_SCAN_WIFI_MAC             ,
   
       ALARM_CANCLE                 ,
       ALARM_PROBE_CANCLE           ,
       ALARM_PROBE                  ,
       ALARM_COMMUNICATE_CANCLE     ,
       ALARM_COMMUNICATE            ,
   } cmd_opt_t;
   #define CMD_BIT_MASK(X)            ( 1UL << (X) )
   
   //rec
   int rrpcAttrQueryParam(const char* messageId, const cJSON *array);
   int setCmdObj(cJSON *array, unsigned int flag, const char isobj);
   void shadowRemoteControl(const cJSON *desired, int vid);
   int operateHistoryTemp(int temp);
   int operateCellInfo(char type, int mcc, int mnc, int lac, int cellid, int signal);
   int operateScanWifi(char type, const char *ssid, const signed char signal, const char *mac);
   
   
   #endif
   
