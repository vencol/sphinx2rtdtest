
.. _program_listing_file_G__code_esp_code_freezer_main_atprocess_atmqtt.h:

Program Listing for File atmqtt.h
=================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_atprocess_atmqtt.h>` (``G:\code\esp\code\freezer\main\atprocess\atmqtt.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __AT_4GMQTTCMD__
   #define __AT_4GMQTTCMD__
   
   
   void atPraseMqttCmd(char* recbuf, int len);
   //mqtt
   int atMqttInfoInit(char secureMode);
   int atConnectMqtt(unsigned char timeout);
   int atDisconnectMqtt(unsigned char timeout);
   int atSubMqttTopic(unsigned char qos, char *topic);
   int atUnSubMqttTopic(unsigned char dup, char *topic);
   int atPublishMqttTopic(char *topic, char *payload);
   int atMqttCallBackInit(void (*IotMqttOperatoinCb)(unsigned char), void (*IotMqttReceiveCb)(char *, char *));
   typedef enum {
       MQTT_NONE,
       MQTT_CONNECT_FAIL,
       MQTT_CONNECT_SUCCESS,
       MQTT_CONNECT_SUB_FAIL,
       MQTT_CONNECT_SUB_SUCCESS,
       MQTT_CONNECT_UNSUB_FAIL,
       MQTT_CONNECT_UNSUB_SUCCESS,
       MQTT_CONNECT_PUB_FAIL,
       MQTT_CONNECT_PUB_SUCCESS,
       MQTT_DISCONNECT_FAIL,
       MQTT_DISCONNECT_SUCCESS,
   } mqtt_uri_map_opt_t;
   
   #define AT_CMD_MQTT_CLIENT_NUM                  0
   #define AT_CMD_MQTT_CLIENT_MAXNUM               1
   #define AT_CMD_MQTT_KEEPALIVE_TIME              60  //*s
   #define AT_CMD_MQTT_CLEAN_SESSION               1
   #define AT_CMD_MQTT_QOS                         1
   #define AT_CMD_MQTT_DUP                         0
   #define AT_CMD_MQTT_RETAIN                      0
   #define AT_CMD_MQTT_TIMEOUT                     60
   
   
   #define AT_CMD_MQTT_RECEIVE_START               "+CMQTTRXSTART:"
   #define AT_CMD_MQTT_RECEIVE_TOPIC               "+CMQTTRXTOPIC:"
   #define AT_CMD_MQTT_RECEIVE_PAYLOAD             "+CMQTTRXPAYLOAD:"
   #define AT_CMD_MQTT_RECEIVE_END                 "+CMQTTRXEND:"
   
   #define AT_CMD_MQTT_INPUT_STR                   "\r\n>"
   #define AT_CMD_MQTT_INPUTOK(buf)            (memcmp(buf, AT_CMD_MQTT_INPUT_STR, strlen(AT_CMD_MQTT_INPUT_STR)) == 0)
   #define AT_CMD_MQTT_SECURE_MAXNUM                 3
   #define AT_CMD_MQTT_RETAIN_MAXNUM                 1
   #define AT_CMD_MQTT_DUP_MAXNUM                    1
   #define AT_CMD_MQTT_QOS_MAXNUM                    2
   #define AT_CMD_MQTT_MAX_TOPIC_LEN                 128
   #define AT_CMD_MQTT_MAX_PAYLOAD_LEN               1024
   
   //AT+CMQTTSTART 
   #define AT_CMD_ATCMQTTSTART_SEND                  "AT+CMQTTSTART\r\n"
   #define AT_CMD_ATCMQTTSTART_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATCMQTTSTART_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCMQTTSTART_RETRY                 (ATCMD_MIN_RETRY)
   #define AT_CMD_ATCMQTTSTART_RESPOND_SPLEN         11
   //AT+CMQTTACCQ
   #define AT_CMD_ATCMQTTACCQ_SEND                  "AT+CMQTTACCQ=%d,\"%s\",%d\r\n"
   #define AT_CMD_ATCMQTTACCQ_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATCMQTTACCQ_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCMQTTACCQ_RETRY                 (ATCMD_MIN_RETRY)
   //AT+CMQTTCONNECT
   #define AT_CMD_ATCMQTTCONNECT_SEND                  "AT+CMQTTCONNECT=%d,\"tcp://%s.%s:%d\",%d,%d,\"%s\",\"%s\"\r\n"
   #define AT_CMD_ATCMQTTCONNECT_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATCMQTTCONNECT_TIMEOUT               (ATCMD_1S_TIMEOUT*10)
   #define AT_CMD_ATCMQTTCONNECT_RETRY                 (ATCMD_MIN_RETRY)
   #define AT_CMD_ATCMQTTCONNECT_SEND_MAXLEN           190
   #define AT_CMD_ATCMQTTCONNECT_KEEPALIVE_MAX         64800 
   #define AT_CMD_ATCMQTTCONNECT_CLEANSEESION_MAX      1
   #define AT_CMD_ATCMQTTCONNECT_RESPOND_SPLEN         13
   
   //AT+CMQTTDISC
   #define AT_CMD_ATCMQTTDISC_SEND                  "AT+CMQTTDISC=%d,%d\r\n"
   #define AT_CMD_ATCMQTTDISC_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATCMQTTDISC_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCMQTTDISC_RETRY                 (ATCMD_MIN_RETRY)
   #define AT_CMD_ATCMQTTDISC_RESPOND_SPLEN         10
   //AT+CMQTTREL
   #define AT_CMD_ATCMQTTREL_SEND                  "AT+CMQTTREL=%d\r\n"
   #define AT_CMD_ATCMQTTREL_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATCMQTTREL_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCMQTTREL_RETRY                 (ATCMD_MIN_RETRY)
   //AT+CMQTTSTOP
   #define AT_CMD_ATCMQTTSTOP_SEND                  "AT+CMQTTSTOP\r\n"
   #define AT_CMD_ATCMQTTSTOP_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATCMQTTSTOP_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCMQTTSTOP_RETRY                 (ATCMD_MIN_RETRY)
   #define AT_CMD_ATCMQTTSTOP_RESPOND_SPLEN         10
   
   //AT+CMQTTSUBTOPIC   RESPOND to send     
   #define AT_CMD_ATCMQTTSUBTOPIC_SEND                  "AT+CMQTTSUBTOPIC=%d,%d,%d\r\n"
   #define AT_CMD_ATCMQTTSUBTOPIC_RESPOND               ""
   #define AT_CMD_ATCMQTTSUBTOPIC_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCMQTTSUBTOPIC_RETRY                 (ATCMD_MIN_RETRY)
   //AT+CMQTTSUB   RESPOND to send     
   #define AT_CMD_ATCMQTTSUB_SEND                  "AT+CMQTTSUB=%d\r\n"
   #define AT_CMD_ATCMQTTSUB_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATCMQTTSUB_TIMEOUT               ATCMD_1S_TIMEOUT
   #define AT_CMD_ATCMQTTSUB_RETRY                 (ATCMD_MIN_RETRY)
   #define AT_CMD_ATCMQTTSUB_RESPOND_SPLEN         9
   
   //AT+CMQTTUNSUBTOPIC   RESPOND to send     
   #define AT_CMD_ATCMQTTUNSUBTOPIC_SEND                  "AT+CMQTTUNSUBTOPIC=%d,%d,%d\r\n"
   #define AT_CMD_ATCMQTTUNSUBTOPIC_RESPOND               ""
   #define AT_CMD_ATCMQTTUNSUBTOPIC_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCMQTTUNSUBTOPIC_RETRY                 (ATCMD_MIN_RETRY)
   //AT+CMQTTUNSUB   RESPOND to send     
   #define AT_CMD_ATCMQTTUNSUB_SEND                  "AT+CMQTTUNSUB=%d\r\n"
   #define AT_CMD_ATCMQTTUNSUB_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATCMQTTUNSUB_TIMEOUT               ATCMD_1S_TIMEOUT
   #define AT_CMD_ATCMQTTUNSUB_RETRY                 (ATCMD_MIN_RETRY)
   #define AT_CMD_ATCMQTTUNSUB_RESPOND_SPLEN           9
   
   //AT+CMQTTTOPIC   RESPOND to send     
   #define AT_CMD_ATCMQTTTOPIC_SEND                  "AT+CMQTTTOPIC=%d,%d\r\n"
   #define AT_CMD_ATCMQTTTOPIC_RESPOND               ""
   #define AT_CMD_ATCMQTTTOPIC_TIMEOUT               (2*ATCMD_MIN_TIMEOUT)
   #define AT_CMD_ATCMQTTTOPIC_RETRY                 (ATCMD_MIN_RETRY)
   //AT+CMQTTPAYLOAD   RESPOND to send     
   #define AT_CMD_ATCMQTTPAYLOAD_SEND                  "AT+CMQTTPAYLOAD=%d,%d\r\n"
   #define AT_CMD_ATCMQTTPAYLOAD_RESPOND               ""
   #define AT_CMD_ATCMQTTPAYLOAD_TIMEOUT               ATCMD_MIN_TIMEOUT
   #define AT_CMD_ATCMQTTPAYLOAD_RETRY                 (ATCMD_MIN_RETRY)
   //AT+CMQTTPUB   RESPOND to send     
   #define AT_CMD_ATCMQTTPUB_SEND                  "AT+CMQTTPUB=%d,%d,%d,%d,%d\r\n"
   #define AT_CMD_ATCMQTTPUB_RESPOND               ATCMDRESPONDOKSTR
   #define AT_CMD_ATCMQTTPUB_TIMEOUT               ATCMD_1S_TIMEOUT
   #define AT_CMD_ATCMQTTPUB_RETRY                 (ATCMD_MIN_RETRY)
   #define AT_CMD_ATCMQTTPUB_RESPOND_SPLEN         9
   #define AT_CMD_ATCMQTTPUB_TIMEOUT_MINI          60
   #define AT_CMD_ATCMQTTPUB_TIMEOUT_MAX           180
   
   #endif
   
