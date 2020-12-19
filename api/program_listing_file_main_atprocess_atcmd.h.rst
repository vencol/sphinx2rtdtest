
.. _program_listing_file_main_atprocess_atcmd.h:

Program Listing for File atcmd.h
================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_atprocess_atcmd.h>` (``main\atprocess\atcmd.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __AT_4GCMD__
   #define __AT_4GCMD__
   
   void atSendCmd(bool first);
   int sendAtCmd(const char* data, int len);
   int atMallocCmd(char cmdstate, char *sendcmdin, char *receiveprefixin);
   int atSendCmdChange(char cmdstate, char *sendcmd, char *receiveprefix, unsigned short timeout, unsigned short retry);
   
   //sendcmd   respond   timeout   rectry
   typedef struct {
       char      *sendcmd;
       char      *receiveprefix;
       char      *payload;
       unsigned int  timeout   : 16;//*10ms
       unsigned int  wait      : 3;
       unsigned int  retry     : 13;
   } atcmd_uri_map_t;
   typedef enum {
       AT,
       ATIPR,
       ATE0,
       ATCGMR,
       ATCPIN,
       ATCSQ,
       ATCREG,
       ATCEREG,
       ATCPSI,
       ATCNETCI1,
       ATCNETCI,
   
       AT_BASE_SPECIAL,
       ATCIMI,
       ATI,
       ATSIMEI,
       AT_BASE_MAX,
   
       ATHTTPINIT,
       ATHTTPPARA,
       ATREADMODE,
       ATHTTPACTION,
       ATHTTPHEAD,
       ATHTTPREAD,
       ATOTAPARA,//OTA
       ATOTAHEAD,
       ATOTAREAD,
       ATHTTPTERM,
       AT_HTTP_MAX,
       ATCMQTTSTART,
       ATCMQTTACCQ,
       ATCMQTTCONNECT,
       ATCMQTTDISC,
       ATCMQTTREL,
       ATCMQTTSTOP,
   
       ATCMQTTSUBTOPIC,
       ATCMQTTSUB,
       ATCMQTTUNSUBTOPIC,
       ATCMQTTUNSUB,
       ATCMQTTTOPIC,
       ATCMQTTPAYLOAD,
       ATCMQTTPUB,
       AT_MQTT_MAX,
       
   } atcmd_uri_map_opt_t;
   extern atcmd_uri_map_opt_t atcmd_current;
   extern atcmd_uri_map_t g_atcmd_uri_map_send;
   
   #define ATCMD_MIN_RETRY                 (3)
   #define ATCMD_MIN_TIMEOUT               (30)
   #define ATCMD_1S_TIMEOUT                (100)
   #define ATCMD_CONMON_TIMEOUT            (ATCMD_MIN_TIMEOUT)
   #define ATCMD_CONMON_RETRY              (ATCMD_MIN_RETRY)
   
   
   #define ATCMD_WAIT_NONE                 (0)
   #define ATCMD_WAIT_ADD                  (1)
   #define ATCMD_WAIT_INPUT                (2)
   #define ATCMD_WAIT_SPECIAL_CMP(buf, name) (memcmp(buf + 2, AT_CMD_##name##_SEND+2, AT_CMD_##name##_RESPOND_SPLEN) == 0)
   #define ATCMD_WAIT_SPECIAL(time)                do{ \
       if(g_atcmd_uri_map_send.wait == ATCMD_WAIT_NONE ) {    \
           g_atcmd_uri_map_send.wait       = ATCMD_WAIT_ADD;    \
           g_atcmd_uri_map_send.retry      = 3;    \
           g_atcmd_uri_map_send.timeout    = time;  \
       }   \
   }while(0)
   #define ATSENDCMD_CHANGE(name)              do{ \
       if( atMallocCmd(name, AT_CMD_##name##_SEND, AT_CMD_##name##_RESPOND) == 0)  \
           atSendCmdChange(name, AT_CMD_##name##_SEND, AT_CMD_##name##_RESPOND, AT_CMD_##name##_TIMEOUT, AT_CMD_##name##_RETRY); \
   }while(0)
   #define ATSENDCMD_CHANGE_NULL()             do{ \
       g_atcmd_uri_map_send.retry      = 0;    \
       g_atcmd_uri_map_send.timeout    = 0;  \
   }while(0)
   
   #define AT_CMD_RESPOND_SPMN_LEN          6 
   #define ATCMDRESPONDOKSTR               "\r\nOK\r\n"
   #define ATCMDRESPONDERRSTR              "\r\nERROR\r\n"
   #define AT_CMD_RESPOND_SPOK             "\r\n\r\nOK\r\n"
   #define ATCMDRESPONDOKSTRLEN            6   //strle(ATCMDRESPONDOKSTR)
   #define ATCMDRESPONDOK(buf, len)        (memcmp(buf+len-ATCMDRESPONDOKSTRLEN, ATCMDRESPONDOKSTR, ATCMDRESPONDOKSTRLEN) == 0)
   
   
   //AT
   #define AT_CMD_AT_SEND                  "AT\r\n"
   #define AT_CMD_AT_RESPOND               "AT"
   #define AT_CMD_AT_TIMEOUT               (ATCMD_1S_TIMEOUT * 3)
   #define AT_CMD_AT_RETRY                 (10)//wait 4g poweron and uart ready
   //ATIPR
   #define AT_CMD_ATIPR_SEND                  "AT+IPR=921600\r\n"
   #define AT_CMD_ATIPR_RESPOND               AT_CMD_RESPOND_SPOK
   #define AT_CMD_ATIPR_TIMEOUT               (ATCMD_1S_TIMEOUT)
   #define AT_CMD_ATIPR_RETRY                 (10)//wait 4g poweron and uart ready
   //ATE0
   #define AT_CMD_ATE0_SEND                  "ATE0\r\n"
   #define AT_CMD_ATE0_RESPOND               "ATE0"
   #define AT_CMD_ATE0_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATE0_RETRY                 (ATCMD_MIN_RETRY)
   //AT+CGMR
   #define AT_CMD_ATCGMR_SEND                  "AT+CGMR\r\n"
   #define AT_CMD_ATCGMR_RESPOND               AT_CMD_RESPOND_SPOK
   #define AT_CMD_ATCGMR_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCGMR_RETRY                 (ATCMD_MIN_RETRY)
   //AT+CIMI  15+2
   #define AT_CMD_ATCIMI_SEND                  "AT+CIMI\r\n"
   #define AT_CMD_ATCIMI_RESPOND               AT_CMD_RESPOND_SPOK
   #define AT_CMD_ATCIMI_TIMEOUT               ATCMD_1S_TIMEOUT
   #define AT_CMD_ATCIMI_RETRY                 (ATCMD_MIN_RETRY)
   //ATI?
   #define AT_CMD_ATI_SEND                  "ATI\r\n"
   #define AT_CMD_ATI_RESPOND               AT_CMD_RESPOND_SPOK
   #define AT_CMD_ATI_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATI_RETRY                 (ATCMD_MIN_RETRY)
   #define AT_CMD_ATI_VERSION_FLAG           "_V"
   #define AT_CMD_ATI_IMEI_FLAG              "IMEI: "
   //AT+CSQ
   #define AT_CMD_ATCSQ_SEND                  "AT+CSQ\r\n"
   #define AT_CMD_ATCSQ_RESPOND               "\r\n+CSQ: "
   #define AT_CMD_ATCSQ_TIMEOUT               ATCMD_1S_TIMEOUT
   #define AT_CMD_ATCSQ_RETRY                 (10)
   //AT+CEREG=2 replace creg
   #define AT_CMD_ATCREG_SEND                  "AT+CEREG=2\r\n"
   #define AT_CMD_ATCREG_RESPOND               "\r\n+CREG: "
   #define AT_CMD_ATCREG_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCREG_RETRY                 (ATCMD_MIN_RETRY)
   //AT+CEREG?
   #define AT_CMD_ATCEREG_SEND                  "AT+CEREG?\r\n"
   #define AT_CMD_ATCEREG_RESPOND               "\r\n+CEREG: "
   #define AT_CMD_ATCEREG_TIMEOUT               (ATCMD_1S_TIMEOUT * 3)
   #define AT_CMD_ATCEREG_RETRY                 (10)
   
   //not use
   //AT+CPIN?
   #define AT_CMD_ATCPIN_SEND                  "AT+CPIN?\r\n"
   #define AT_CMD_ATCPIN_RESPOND               "\r\n+CPIN: READY"
   #define AT_CMD_ATCPIN_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCPIN_RETRY                 (ATCMD_MIN_RETRY)
   //AT+SIMEI
   #define AT_CMD_ATSIMEI_SEND                  "AT+SIMEI?\r\n"
   #define AT_CMD_ATSIMEI_RESPOND               "\r\n+SIMEI: "
   #define AT_CMD_ATSIMEI_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATSIMEI_RETRY                 (ATCMD_MIN_RETRY)
   
   
   //AT+CNETCI1
   #define AT_CMD_ATCNETCI1_SEND                  "AT+CNETCI=1\r\n"
   #define AT_CMD_ATCNETCI1_RESPOND               AT_CMD_RESPOND_SPOK
   #define AT_CMD_ATCNETCI1_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCNETCI1_RETRY                 (ATCMD_MIN_RETRY)
   //AT+CNETCI?
   #define AT_CMD_ATCNETCI_SEND                  "AT+CNETCI?\r\n"
   #define AT_CMD_ATCNETCI_RESPOND               "\r\n+CCNETCI: "
   #define AT_CMD_ATCNETCI_TIMEOUT               ATCMD_1S_TIMEOUT
   #define AT_CMD_ATCNETCI_RETRY                 (ATCMD_MIN_RETRY)
   //AT+CPSI?
   #define AT_CMD_ATCPSI_SEND                  "AT+CPSI?\r\n"
   #define AT_CMD_ATCPSI_RESPOND               AT_CMD_RESPOND_SPOK
   #define AT_CMD_ATCPSI_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCPSI_RETRY                 (ATCMD_MIN_RETRY)
   
   
   #endif
   
