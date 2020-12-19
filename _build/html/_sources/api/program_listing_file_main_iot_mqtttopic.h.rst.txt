
.. _program_listing_file_main_iot_mqtttopic.h:

Program Listing for File mqtttopic.h
====================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_iot_mqtttopic.h>` (``main\iot\mqtttopic.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __MQTT_PROCESS__
   #define __MQTT_PROCESS__
   
   int subMqttTopic(unsigned char topicnum, char *flexfix);
   int unSubMqttTopic(unsigned char topicnum, char *flexfix);
   int publishMqttTopic(unsigned char topicnum, char *flexfix, char *payload);
   
   
   typedef enum {
       MQTT_TOPIC_SUB_RRPC,
       MQTT_TOPIC_SUB_SHADOW,
       MQTT_TOPIC_SUB_RPT,
       MQTT_TOPIC_SUB_UPGRADE,
   
       MQTT_TOPIC_PUB_SHADOW,
       MQTT_TOPIC_PUB_DEVINFO,
       MQTT_TOPIC_PUB_ATTRRPT,
       MQTT_TOPIC_PUB_ALARMRPT,
       MQTT_TOPIC_PUB_RRPC,
       MQTT_TOPIC_PUB_UPGRADEACK,
   
       MQTT_TOPIC_NONE
   } mqtt_topic_map_opt_t;
   
   #define MQTT_TOPIC_SUB_END      (MQTT_TOPIC_SUB_UPGRADE + 1)
   #define MQTT_TOPIC_PUB_END      (MQTT_TOPIC_PUB_ALARMRPT + 1)
   
   #endif
   
