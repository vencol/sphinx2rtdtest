
.. _program_listing_file_main_atprocess_athttp.h:

Program Listing for File athttp.h
=================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_atprocess_athttp.h>` (``main\atprocess\athttp.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __AT_4GHTTPCMD__
   #define __AT_4GHTTPCMD__
   
   void atCmdStartGetIotInfo(void);
   void atPraseHttpCmd(char* recbuf, int len);
   
   #define AT_CMD_JSON_STATUS                    "status"
   #define AT_CMD_JSON_DATA                      "data"
   #define AT_CMD_JSON_DOMAIN                    "domain"
   #define AT_CMD_JSON_PRODUCTKEY                "productKey"
   #define AT_CMD_JSON_DEVICESECRET              "deviceSecret"
   
   //AT+HTTPINIT
   #define AT_CMD_ATHTTPINIT_SEND                  "AT+HTTPINIT\r\n"
   #define AT_CMD_ATHTTPINIT_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATHTTPINIT_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATHTTPINIT_RETRY                 (ATCMD_MIN_RETRY)
   //AT+HTTPPARA
   #define AT_CMD_ATHTTPPARA_SEND                  "AT+HTTPPARA=\"URL\",\"http://iot.mengniu.com.cn/iot/device/registration?imei=%lld&customerId=%s\"\r\n"
   #define AT_CMD_ATHTTPPARA_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATHTTPPARA_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATHTTPPARA_RETRY                 (ATCMD_MIN_RETRY)
   //AT+READMODE readmode
   #define AT_CMD_ATREADMODE_SEND                  "AT+HTTPPARA=\"READMODE\",1\r\n"
   #define AT_CMD_ATREADMODE_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATREADMODE_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATREADMODE_RETRY                 (ATCMD_MIN_RETRY)
   //AT+HTTPACTION
   #define AT_CMD_ATHTTPACTION_SEND                  "AT+HTTPACTION=0\r\n"
   #define AT_CMD_ATHTTPACTION_RESPOND               "+HTTPACTION: "
   #define AT_CMD_ATHTTPACTION_TIMEOUT               (ATCMD_1S_TIMEOUT * 30)
   #define AT_CMD_ATHTTPACTION_RETRY                 (ATCMD_MIN_RETRY)
   #define AT_CMD_ATHTTPACTION_RESPOND_SPLEN          11
   //AT+HTTPHEAD
   #define AT_CMD_ATHTTPHEAD_SEND                  "AT+HTTPHEAD\r\n"
   #define AT_CMD_ATHTTPHEAD_RESPOND               "+HTTPHEAD: "
   #define AT_CMD_ATHTTPHEAD_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATHTTPHEAD_RETRY                 (ATCMD_MIN_RETRY)
   //AT+HTTPREAD
   #define AT_CMD_ATHTTPREAD_SEND                  "AT+HTTPREAD=%d,%d\r\n"
   #define AT_CMD_ATHTTPREAD_RESPOND               "+HTTPREAD:"
   #define AT_CMD_ATHTTPREAD_TIMEOUT               ATCMD_1S_TIMEOUT
   #define AT_CMD_ATHTTPREAD_RETRY                 (ATCMD_MIN_RETRY)
   #define AT_CMD_ATHTTPREAD_SEND_MAXLEN           (28)
   #define AT_CMD_ATHTTPREAD_RESPOND_SPLEN         9
   //AT+HTTPTERM
   #define AT_CMD_ATHTTPTERM_SEND                  "AT+HTTPTERM\r\n"
   #define AT_CMD_ATHTTPTERM_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATHTTPTERM_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATHTTPTERM_RETRY                 (ATCMD_MIN_RETRY)
   
   
   //AT+OTAPARA
   #define AT_CMD_ATOTAPARA_SEND                  "AT+HTTPPARA=\"URL\",\"http://%s\"\r\n"
   #define AT_CMD_ATOTAPARA_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATOTAPARA_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATOTAPARA_RETRY                 (ATCMD_MIN_RETRY)
   //AT+OTAREAD
   #define AT_CMD_ATOTAREAD_SEND                  "AT+HTTPREAD=%d,%d\r\n"
   #define AT_CMD_ATOTAREAD_RESPOND               "+HTTPREAD:"
   #define AT_CMD_ATOTAREAD_TIMEOUT               10*ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATOTAREAD_RETRY                 (ATCMD_MIN_RETRY)
   #define AT_CMD_ATOTAREAD_SEND_MAXLEN           (28)
   #define AT_CMD_ATOTAREAD_RESPOND_SPLEN          9
   
   #endif
   
