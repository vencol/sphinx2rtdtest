
.. _program_listing_file_G__code_esp_code_freezer_main_freezerdrv_drv_nvs_flash.h:

Program Listing for File drv_nvs_flash.h
========================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_freezerdrv_drv_nvs_flash.h>` (``G:\code\esp\code\freezer\main\freezerdrv\drv_nvs_flash.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __DRV_NVS_FLASH__
   #define __DRV_NVS_FLASH__
   
   //nvs flash
   #define LINKKIT_FIRMWARE_VERSION        "0.0.1"
   #define IOTX_FIRMWARE_VERSION_LEN       (32)
   #define IOTX_PRODUCT_KEY_LEN            (20)
   #define IOTX_DEVICE_NAME_LEN            (32)
   #define IOTX_DEVICE_SECRET_LEN          (64)
   #define IOTX_DEVICE_ID_LEN              (64)
   #define IOTX_PRODUCT_SECRET_LEN         (64)
   #define IOTX_FIRMWARE_VER_LEN           (32)
   #define IOTX_MQTT_DOMAIN_LEN            (64)
   #define IOTX_MQTT_PORT_LEN              (8)
   int halGetIotMqttDomain(char domain[IOTX_MQTT_DOMAIN_LEN + 1]);
   int halGetIotMqttPort(char port[IOTX_MQTT_PORT_LEN + 1]);
   int halGetProductKey(char product_key[IOTX_PRODUCT_KEY_LEN + 1]);
   int halGetDeviceName(char device_name[IOTX_DEVICE_NAME_LEN + 1]);
   int halGetDeviceSecret(char device_secret[IOTX_DEVICE_SECRET_LEN + 1]);
   int halGetFirmwareVersion(char *version);
   int halSetDeviceParam(char *device_param, int len);
   
   int halSetIotMqttDomain(char *mqttdomain);
   int halSetIotMqttPort(char *mqttport);
   int halSetDeviceName(char *device_name);
   int halSetDeviceSecret(char *device_name);
   int halSetProductKey(char *device_name);
   int halSetProductSecret(char *device_name);
   int halGetDeviceParam(char *device_param, int len);
   
   int halSetNumIotMqttPort(int port);
   int halSetNumDeviceName(unsigned long long device_id);
   
   #endif
   
