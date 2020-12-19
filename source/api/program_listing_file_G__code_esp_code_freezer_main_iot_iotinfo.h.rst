
.. _program_listing_file_G__code_esp_code_freezer_main_iot_iotinfo.h:

Program Listing for File iotinfo.h
==================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_iot_iotinfo.h>` (``G:\code\esp\code\freezer\main\iot\iotinfo.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __IOT_INFO__
   #define __IOT_INFO__
   
   #pragma pack(1)
   typedef struct {
       unsigned int          atversion;
       unsigned int    atSendtime  : 7;
       unsigned int    atStatus    : 1;
       unsigned int    uartStatus  : 1;
       unsigned int    csqRssi     : 5;
       unsigned int    csqBer      : 4;
       unsigned int    reg2gStatus : 3;
       unsigned int    reg4gStatus : 3;
   } atcmd_module_t;
   typedef struct {
       char            *mainlbs;
       char            *otherlbs;
       char            *historytemp;
       char            *wifimac;
       unsigned int    wifinum     : 5;
       unsigned int    otherlbsnum : 3;
       unsigned int    noalarms    : 1;
       unsigned int    noprobealarms   : 1;
       unsigned int    probealarms : 1;
       unsigned int    nocommunicatealarms : 1;
       unsigned int    communicatealarms   : 1;
       unsigned int    powerstatus1    : 1;
   } device_info_t;
   typedef struct {
       unsigned short      upbacklash;//0.1°C  //C1
       unsigned short      downbacklash;//0.1°C    //C2
       signed   short      hightempalarm;//0.1°C    
       signed   short      lowtempalarm;//0.1°C    
   
       signed   char       settemphigh;//1°C   
       signed   char       settemplow;//1°C   
       unsigned char       erroron;//1min  //C3
       unsigned char       erroroff;//1min //C4
   
       unsigned int        heartTime       : 12;//1s
       unsigned int        showway         : 2;//
       unsigned int        powerstatus     : 2;//
       unsigned int        reportTime      : 6;//1min
       unsigned int        flagsave        : 10;//
   
       signed   short      tempoffset;//0.1°C    //C5
       signed   char       settemp;//1°C   //Ts
   } device_param_t;
   // #pragma anon_unions
   typedef union 
   {
       unsigned short  errbit;
       struct {
           unsigned short  errnone     : 1;
           unsigned short  errnosim    : 1;
           unsigned short  errfrozen  : 1;
           unsigned short  errrefrigera: 1;
           unsigned short  errregister : 1;
           unsigned short  errsim      : 1;
           unsigned short  errcsq      : 1;
           unsigned short  errmcu      : 1;
           unsigned short  errwifi     : 1;
           unsigned short  errble      : 1;
       };
   }error_info_t;
   typedef struct {
       atcmd_module_t  *at_module;
       device_info_t   *devinfo;
       device_param_t  *devparam;
       unsigned long long      imei;//deviceName deviceId
       unsigned long long      imsi;
       char            *productKey;
       char            *deviceSecret;
       char            *clientId;
       char            *userName;
       char            *passWord;
       char            *iotDomain;
       char            *upgradeSn;
       char            *upgradeUrl;
       error_info_t    error;
       unsigned short  operateTime;
       unsigned short  tempDisTime;
       signed   short  realTemp;
       signed   short  showTemp;
       unsigned int    powerOnTime;
       unsigned short  iotPort     : 12;
       unsigned short  setdisplay1 : 3;
       unsigned short  displayalarm: 1;
       unsigned char   iotStatus   : 5;
       unsigned char   otaStatus  : 2;
       unsigned char   firstOn     : 1;
       // unsigned char    displayStatus;
       signed   char   secureMode;
   } iotinfo_module_t;
   #pragma pack()
   extern iotinfo_module_t iotinfo_module;
   
   typedef enum {
       IOT_NONE,
       IOT_GETBASEINFO,
       IOT_SAVEBASEINFO,
       IOT_SETMQTTINFO,
       IOT_CONNECTMQTT,
   } iot_status_opt_t;
   typedef enum {
       ERR_OK,
       ERR_COMMON,
   
       ERR_CMD_TIMEOUT=11,
       ERR_CMD_INVALID,
       ERR_CMD_ILLEGAL,
       ERR_PARAMETER_ILLEGAL,
   
       ERR_UPGRADE_URL_INVALID=91,
       ERR_UPGRADE_CHECK_FAILED,
       ERR_UPGRADE_UNKNOWN_CHECKOUT_TYPE,
       ERR_UPGRADE_UNKNOWN_UPGRADE_TYPE,
       ERR_UPGRADE_NOT_ALLOW,
       ERR_UPGRADE_CONTENT_ERROR,
   
       ERR_DEV_BASE=1000,
   } iot_error_opt_t;
   //common
   int SaveInfoToFlash(char *domain);
   unsigned long long str2Imei(char* buf);
   int getMqttInfo(iotinfo_module_t *pmqtt_module);
   extern void fzReportAttrsInfo(char step);
   void setReportTime(unsigned int time);
   void fzRespondUgrade(int errnum);
   void fzReportAlarmInfo(void);
   int iotInfoInit(void);
   void iotInfoPro(void);
   int bleIbeaconInit(void);
   void WifiScanInit(void);
   void wifiScanStart(char channel);
   #define FREEZER_APP_VERSION          CONFIG_APP_PROJECT_VER
   #define POWER_STATUS_ON              (0)
   #define POWER_STATUS_LOCALOFF        (1)
   #define POWER_STATUS_REMOTEOFF       (2)
   #define POWER_STATUS_LOCALON         (3)
   
   #define AT_CMD_IMEI_IMSI_LEN              (15)
   #define AT_CMD_CUSTOMID                  "9ad61d41da6c46ab81"
   #define AT_CMD_SUCCESSS              (0)
   #define AT_CMD_FAIL                  (-1)
   /* all secure mode define */
   #define MODE_TLS_GUIDER             -1
   #define MODE_TLS_DIRECT             2
   #define MODE_TCP_DIRECT_PLAIN       3
   #define SECURE_MODE                 MODE_TCP_DIRECT_PLAIN
   //
   #define DEFAULT_ATTRS_FREQUENCY         1//MIN
   #define DEFAULT_MQTT_HEART_FREQUENCY    60//S
   
   #define POWERON_6HOUR                   (100U * 60 * 60 * 6)//MS
   #define CLOSE_TIME_CONTINUE             (100U * 60 * 15)//MS
   #define CLOSE_TIME_MINI                 (100U * 60 * 3)//MS
   #define CLOSE_TIME_FIRSTTSG             (100U * 60 * 3)//MS
   #define FIRSTON_TIME                    (100U * 6)//MS
   
   
   // #define POWERON_6HOUR                    4000//(100U * 60 * 60 * 6)//MS
   // #define CLOSE_TIME_CONTINUE             3000//(100U * 60 * 15)//MS
   // #define CLOSE_TIME_MINI                  1000//(100U * 60 * 3)//MS
   // #define CLOSE_TIME_FIRSTTSG              (100U * 60 * 3)//MS
   // #define FIRSTON_TIME                 (100U * 6)//MS
   #endif
   
