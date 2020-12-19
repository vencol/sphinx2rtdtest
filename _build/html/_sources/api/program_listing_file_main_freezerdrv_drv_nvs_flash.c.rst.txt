
.. _program_listing_file_main_freezerdrv_drv_nvs_flash.c:

Program Listing for File drv_nvs_flash.c
========================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_freezerdrv_drv_nvs_flash.c>` (``main\freezerdrv\drv_nvs_flash.c``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /*
    * ESPRESSIF MIT License
    *
    * Copyright (c) 2019 <ESPRESSIF SYSTEMS (SHANGHAI) PTE LTD>
    *
    * Permission is hereby granted for use on all ESPRESSIF SYSTEMS products, in which case,
    * it is free of charge, to any person obtaining a copy of this software and associated
    * documentation files (the "Software"), to deal in the Software without restriction, including
    * without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense,
    * and/or sell copies of the Software, and to permit persons to whom the Software is furnished
    * to do so, subject to the following conditions:
    *
    * The above copyright notice and this permission notice shall be included in all copies or
    * substantial portions of the Software.
    *
    * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
    * FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
    * COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
    * IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
    * CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
    *
    */
   
   #include <stdio.h>
   #include <string.h>
   
   
   #include "esp_err.h"
   #include "esp_log.h"
   
   #include "nvs_flash.h"
   #include "nvs.h"
   #include "drv_nvs_flash.h"
   
   
   #define MFG_PARTITION_NAME "nvs"
   #define NVS_PRODUCT "aliyun-key"
   #define FLASH_KEY_DEVICE_PARAM      "DeviceParam"
   #define FLASH_KEY_DEVICE_NAME       "DeviceName"
   #define FLASH_KEY_DEVICE_SECRET     "DeviceSecret"
   #define FLASH_KEY_PRODUCT_KEY       "ProductKey"
   #define FLASH_KEY_PRODUCT_SECRET    "ProductSecret"
   #define FLASH_KEY_MQTT_DOMAIN       "MqttDomain"
   #define FLASH_KEY_MQTT_PORT         "MqttPort"
   
   
   static const char *TAG = "drv_nvs_flash";
   
   static bool s_part_init_flag;
   
   static esp_err_t halProductParamInit(void)
   {
       esp_err_t ret = ESP_OK;
   
       do {
           if (s_part_init_flag == false) {
               if ((ret = nvs_flash_init_partition(MFG_PARTITION_NAME)) != ESP_OK) {
                   ESP_LOGE(TAG, "NVS Flash init %s failed, Please check that you have flashed fctry partition!!!", MFG_PARTITION_NAME);
                   break;
               }
   
               s_part_init_flag = true;
           }
       } while (0);
   
       return ret;
   }
   
   static int halGetProductParam(char *param_name, const char *param_name_str, int len)
   {
       esp_err_t ret = ESP_OK;
       size_t read_len = 0;
       nvs_handle handle;
   
       do {
           if (halProductParamInit() != ESP_OK) {
               break;
           }
   
           if (param_name == NULL) {
               ESP_LOGE(TAG, "%s param %s NULL", __func__, param_name);
               break;
           }
   
           ret = nvs_open_from_partition(MFG_PARTITION_NAME, NVS_PRODUCT, NVS_READONLY, &handle);
   
           if (ret != ESP_OK) {
               ESP_LOGE(TAG, "%s nvs_open failed with %x", __func__, ret);
               break;
           }
   
           if(len <= 0)    {
               ret = nvs_get_str(handle, param_name_str, NULL, (size_t *)&read_len);
   
               if (ret != ESP_OK) {
                   ESP_LOGE(TAG, "%s nvs_get_str get %s failed with %x", __func__, param_name_str, ret);
                   break;
               }
   
               ret = nvs_get_str(handle, param_name_str, param_name, (size_t *)&read_len);
   
               if (ret != ESP_OK) {
                   ESP_LOGE(TAG, "%s nvs_get_str get %s failed with %x", __func__, param_name_str, ret);
               } else {
                   ESP_LOGV(TAG, "%s %s %s", __func__, param_name_str, param_name);
               }
           }
           else    {
               read_len = len;
               ret = nvs_get_blob(handle, param_name_str, param_name, (size_t *)&read_len);
   
               if (ret != ESP_OK || read_len != len) {
                   ESP_LOGE(TAG, "%s nvs_get_str get %s failed with %x or len", __func__, param_name_str, ret);
               } else {
                   ESP_LOGV(TAG, "%s %s %s", __func__, param_name_str, param_name);
               }
           }
           
   
       } while (0);
   
       nvs_close(handle);
       if (ret != ESP_OK)
           read_len = 0;
       return read_len;
   }
   int halGetDeviceParam(char *device_param, int len)
   {
       if (len <= 0)
           return -1;
       return halGetProductParam(device_param, FLASH_KEY_DEVICE_PARAM, len);
   }
   int halGetDeviceName(char device_name[IOTX_DEVICE_NAME_LEN + 1])
   {
       return halGetProductParam(device_name, FLASH_KEY_DEVICE_NAME, 0);
   }
   
   int halGetDeviceSecret(char device_secret[IOTX_DEVICE_SECRET_LEN + 1])
   {
       return halGetProductParam(device_secret, FLASH_KEY_DEVICE_SECRET, 0);
   }
   
   int halGetProductKey(char product_key[IOTX_PRODUCT_KEY_LEN + 1])
   {
       return halGetProductParam(product_key, FLASH_KEY_PRODUCT_KEY, 0);
   }
   
   int halGetProductSecret(char product_secret[IOTX_PRODUCT_SECRET_LEN + 1])
   {
       return halGetProductParam(product_secret, FLASH_KEY_PRODUCT_SECRET, 0);
   }
   
   int halGetIotMqttDomain(char domain[IOTX_MQTT_DOMAIN_LEN + 1])
   {
       return halGetProductParam(domain, FLASH_KEY_MQTT_DOMAIN, 0);
   }
   
   int halGetIotMqttPort(char port[IOTX_MQTT_PORT_LEN + 1])
   {
       return halGetProductParam(port, FLASH_KEY_MQTT_PORT, 0);
   }
   
   int halGetFirmwareVersion(char *version)
   {
       if (!version) {
           ESP_LOGE(TAG, "%s version is NULL", __func__);
           return 0;
       }
   
       memset(version, 0, IOTX_FIRMWARE_VER_LEN);
       int len = strlen(CONFIG_LINKKIT_FIRMWARE_VERSION);
       if (len > IOTX_FIRMWARE_VER_LEN) {
           len = 0;
       } else {
           memcpy(version, CONFIG_LINKKIT_FIRMWARE_VERSION, len);
       }
   
       return len;
   }
   
   static int halSetProductParam(char *param_name, const char *param_name_str, int len)
   {
       esp_err_t ret = ESP_OK;
       size_t write_len = 0;
       nvs_handle handle;
   
       do {
           if (halProductParamInit() != ESP_OK) {
               break;
           }
   
           if (param_name == NULL) {
               ESP_LOGE(TAG, "%s param %s NULL", __func__, param_name);
               break;
           }
   
           ret = nvs_open_from_partition(MFG_PARTITION_NAME, NVS_PRODUCT, NVS_READWRITE, &handle);
   
           if (ret != ESP_OK) {
               ESP_LOGE(TAG, "%s nvs_open failed with %x", __func__, ret);
               break;
           }
   
           ESP_LOGI(TAG, "%s %s:[%s] len:%d", __func__, param_name_str, param_name, len);
           if(len <= 0)    {
               ret = nvs_set_str(handle, param_name_str, param_name);
   
               if (ret != ESP_OK) {
                   ESP_LOGE(TAG, "%s nvs_set_str set %s failed with %x", __func__, param_name_str, ret);
               } else {
                   write_len = strlen(param_name);
                   ESP_LOGV(TAG, "%s %s %s", __func__, param_name_str, param_name);
               }
           }
           else    {
               write_len = len;
               ret = nvs_set_blob(handle, param_name_str, param_name, write_len);
   
               if (ret != ESP_OK) {
                   ESP_LOGE(TAG, "%s nvs_set_str set %s failed with %x or len", __func__, param_name_str, ret);
               } else {
                   ESP_LOGV(TAG, "%s %s %s", __func__, param_name_str, param_name);
               }
           }
   
       } while (0);
   
       nvs_close(handle);
       if (ret != ESP_OK)
           write_len = 0;
   
       return write_len;
   }
   
   int halSetDeviceParam(char *device_param, int len)
   {
       if (len <= 0)
           return -1;
       return halSetProductParam(device_param, FLASH_KEY_DEVICE_PARAM, len);
   }
   
   int halSetDeviceName(char *device_name)
   {
       return halSetProductParam(device_name, FLASH_KEY_DEVICE_NAME, 0);
   }
   
   int halSetDeviceSecret(char *device_secret)
   {
       return halSetProductParam(device_secret, FLASH_KEY_DEVICE_SECRET, 0);
   }
   
   int halSetProductKey(char *product_key)
   {
       return halSetProductParam(product_key, FLASH_KEY_PRODUCT_KEY, 0);
   }
   
   int halSetProductSecret(char *product_secret)
   {
       return halSetProductParam(product_secret, FLASH_KEY_PRODUCT_SECRET, 0);
   }
   
   int halSetIotMqttDomain(char *mqttdomain)
   {
       return halSetProductParam(mqttdomain, FLASH_KEY_MQTT_DOMAIN, 0);
   }
   
   int halSetIotMqttPort(char *mqttport)
   {
       return halSetProductParam(mqttport, FLASH_KEY_MQTT_PORT, 0);
   }
   
   int halSetNumDeviceName(unsigned long long device_id)
   {
       char device_name[IOTX_DEVICE_NAME_LEN+1];
       snprintf(device_name, IOTX_DEVICE_NAME_LEN, "%lld", device_id);
       return halSetDeviceName(device_name);
   }
   int halSetNumIotMqttPort(int port)
   {
       char mqttport[IOTX_MQTT_PORT_LEN+1];
       snprintf(mqttport, IOTX_MQTT_PORT_LEN, "%d", port);
       return halSetIotMqttPort(mqttport);
   }
   
   
   
   
   
