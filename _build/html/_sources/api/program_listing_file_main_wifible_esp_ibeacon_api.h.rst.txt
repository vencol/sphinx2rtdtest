
.. _program_listing_file_main_wifible_esp_ibeacon_api.h:

Program Listing for File esp_ibeacon_api.h
==========================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_wifible_esp_ibeacon_api.h>` (``main\wifible\esp_ibeacon_api.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /*
      This example code is in the Public Domain (or CC0 licensed, at your option.)
   
      Unless required by applicable law or agreed to in writing, this
      software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
      CONDITIONS OF ANY KIND, either express or implied.
   */
   
   
   
   /****************************************************************************
   *
   * This file is for iBeacon definitions. It supports both iBeacon sender and receiver
   * which is distinguished by macros IBEACON_SENDER and IBEACON_RECEIVER,
   *
   * iBeacon is a trademark of Apple Inc. Before building devices which use iBeacon technology,
   * visit https://developer.apple.com/ibeacon/ to obtain a license.
   *
   ****************************************************************************/
   
   #include <stdint.h>
   #include <string.h>
   #include <stdbool.h>
   #include <stdio.h>
   
   #include "esp_gap_ble_api.h"
   #include "esp_gattc_api.h"
   
   
   /* Because current ESP IDF version doesn't support scan and adv simultaneously,
    * so iBeacon sender and receiver should not run simultaneously */
   #define IBEACON_SENDER      0
   #define IBEACON_RECEIVER    1
   #define IBEACON_MODE CONFIG_IBEACON_MODE
   
   /* Major and Minor part are stored in big endian mode in iBeacon packet,
    * need to use this macro to transfer while creating or processing
    * iBeacon data */
   #define ENDIAN_CHANGE_U16(x) ((((x)&0xFF00)>>8) + (((x)&0xFF)<<8))
   
   /* Espressif WeChat official account can be found using WeChat "Yao Yi Yao Zhou Bian", 
    * if device advertises using ESP defined UUID. 
    * Please refer to http://zb.weixin.qq.com for further information. */
   // #define ESP_UUID    {0xFD, 0xA5, 0x06, 0x93, 0xA4, 0xE2, 0x4F, 0xB1, 0xAF, 0xCF, 0xC6, 0xEB, 0x07, 0x64, 0x78, 0x25}
   // #define ESP_MAJOR   10167
   // #define ESP_MINOR   61958
   #define ESP_UUID    {0xAB, 0x81, 0x90, 0xD5, 0xD1, 0x1E, 0x49, 0x41, 0xAC, 0xC4, 0x42, 0xF3, 0x05, 0x10, 0xB4, 0x08}
   #define ESP_MAJOR   10099
   #define ESP_MINOR   52844
   
   
   typedef struct {
       uint8_t flags[3];
       uint8_t length;
       uint8_t type;
       uint16_t company_id;
       uint16_t beacon_type;
   }__attribute__((packed)) esp_ble_ibeacon_head_t;
   
   typedef struct {
       uint8_t proximity_uuid[16];
       uint16_t major;
       uint16_t minor;
       int8_t measured_power;
   }__attribute__((packed)) esp_ble_ibeacon_vendor_t;
   
   
   typedef struct {
       esp_ble_ibeacon_head_t ibeacon_head;
       esp_ble_ibeacon_vendor_t ibeacon_vendor;
   }__attribute__((packed)) esp_ble_ibeacon_t;
   
   
   /* For iBeacon packet format, please refer to Apple "Proximity Beacon Specification" doc */
   /* Constant part of iBeacon data */
   extern esp_ble_ibeacon_head_t ibeacon_common_head;
   
   bool espBleIsIbeaconPacket (uint8_t *adv_data, uint8_t adv_data_len);
   
   esp_err_t esp_ble_config_ibeacon_data (esp_ble_ibeacon_vendor_t *vendor_config, esp_ble_ibeacon_t *ibeacon_adv_data);
