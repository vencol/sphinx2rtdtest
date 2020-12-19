
.. _program_listing_file_G__code_esp_code_freezer_main_app_main.c:

Program Listing for File app_main.c
===================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_app_main.c>` (``G:\code\esp\code\freezer\main\app_main.c``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   
   #include <string.h>
   #include "esp_log.h"
   #include "iotmain.h"
   static const char *TAG = "freezer";
   
   #include "esp_ota_ops.h"
   static esp_ota_handle_t update_handle = 0 ;
   int appOtaReceive(int step, char* buf, int buflen)
   {
     static int filelen = 0;
     if(buf == NULL || buflen <= 0) {
       ESP_LOGE(TAG, "the ota param error");
       update_handle = 0;
       return -1;
     }
     esp_err_t err;
     // const esp_partition_t *running = esp_ota_get_running_partition();
     const esp_partition_t *update_partition = esp_ota_get_next_update_partition(NULL);
     if(step > 0)  {
       if(step == 1) {
         esp_app_desc_t new_app_info;
         if (buflen > sizeof(esp_image_header_t) + sizeof(esp_image_segment_header_t) + sizeof(esp_app_desc_t)) {
           // check current version with downloading
           esp_app_desc_t running_app_info;
           memcpy(&new_app_info, &buf[sizeof(esp_image_header_t) + sizeof(esp_image_segment_header_t)], sizeof(esp_app_desc_t));
           ESP_LOGI(TAG, "New firmware version: [%s] and running version is [%s]", new_app_info.version, CONFIG_APP_PROJECT_VER);
           // if (memcmp(new_app_info.version, running_app_info.version, sizeof(new_app_info.version)) == 0) {
           //   ESP_LOGW(TAG, "Current running version is the same as a new. We will not continue the update.");
           //   update_handle = 0;
           //   return -1;
           // }
           filelen = 0;
         }
         else  {
           ESP_LOGE(TAG, "the first receive error");
           update_handle = 0;
           return -1;
         }
         err = esp_ota_begin(update_partition, OTA_SIZE_UNKNOWN, &update_handle);
         if (err != ESP_OK) { 
           ESP_LOGE(TAG, "the first ota begin error");
           update_handle = 0;
           return -1;
         }
         ESP_LOGI(TAG, "esp_ota_begin succeeded");
       }
       
       err = esp_ota_write( update_handle, (const void *)buf, buflen);
       if (err != ESP_OK) {
         ESP_LOGE(TAG, "ota write error");
         update_handle = 0;
         return -1;
       }
       filelen += buflen;
       ESP_LOGI(TAG, "esp_ota_write filelen:%d", filelen);
     }
     else if(step == 0){
       update_handle = 0;
     }
     else if(step == -1){
       err = esp_ota_write( update_handle, (const void *)buf, buflen);
       if (err != ESP_OK) {
         ESP_LOGE(TAG, "last ota write error");
         update_handle = 0;
         return -1;
       }
       filelen += buflen;
       ESP_LOGI(TAG, "esp_ota_write filelen:%d", filelen);
       err = esp_ota_end(update_handle);
       if (err != ESP_OK) {
         if (err == ESP_ERR_OTA_VALIDATE_FAILED) {
           ESP_LOGE(TAG, "Image validation failed, image is corrupted");
         }
         ESP_LOGE(TAG, "esp_ota_end failed (%s)! filelen:%d", esp_err_to_name(err), filelen);
         update_handle = 0;
         return -1;
       }
       err = esp_ota_set_boot_partition(update_partition);
       if (err != ESP_OK) {
         ESP_LOGE(TAG, "esp_ota_set_boot_partition failed (%s)!", esp_err_to_name(err));
         update_handle = 0;
         return -1;
       }
       ESP_LOGI(TAG, "esp_ota_write last filelen:%d", filelen);
       ESP_LOGI(TAG, "Prepare to restart system!");
       // esp_restart();
     }
     return 0;
   }
   static void appBootInit(void)
   {
     esp_app_desc_t running_app_info;
     esp_ota_img_states_t ota_state;
     const esp_partition_t *configured = esp_ota_get_boot_partition();
     const esp_partition_t *running = esp_ota_get_running_partition();
     const esp_partition_t *update_partition = esp_ota_get_next_update_partition(NULL);
     ESP_LOGI(TAG, "filesize imgheader:%d imgsegment:%d appdesc:%d total:%d ver:[%s]", sizeof(esp_image_header_t), sizeof(esp_image_segment_header_t), sizeof(esp_app_desc_t)
                             , sizeof(esp_image_header_t) + sizeof(esp_image_segment_header_t) + sizeof(esp_app_desc_t), CONFIG_APP_PROJECT_VER);
     if (esp_ota_get_partition_description(running, &running_app_info) == ESP_OK) 
       ESP_LOGI(TAG, "start from addr:0x%08X version:[%s] size:%d KB type:%d name:[%s]", running->address, running_app_info.version, running->size/1024, running->subtype, running->label);
     else
       ESP_LOGI(TAG, "start from addr:0x%08X size:%d KB type:%d name:[%s]", running->address, running->size/1024, running->subtype, running->label);
   
     if(update_partition)
       ESP_LOGI(TAG, "update_partition in addr:0x%08X size:%d KB type:%d name:[%s]", update_partition->address, update_partition->size/1024, update_partition->subtype, update_partition->label);
     if (configured != running) {
       ESP_LOGW(TAG, "Configured OTA boot partition at offset 0x%08x, but running from offset 0x%08x",configured->address, running->address);
       ESP_LOGW(TAG, "(This can happen if either the OTA boot data or preferred boot image become corrupted somehow.)");
       ESP_LOGI(TAG, "configured app in addr:0x%08X size:%d type:%d KB name:[%s]", configured->address, configured->size/1024, configured->subtype, configured->label);
     }
   
     if (esp_ota_get_state_partition(running, &ota_state) == ESP_OK) {
       if (ota_state == ESP_OTA_IMG_PENDING_VERIFY) {
         // run diagnostic function ...
         // bool diagnostic_is_ok = diagnostic();
         // if (diagnostic_is_ok) {
         //     ESP_LOGI(TAG, "Diagnostics completed successfully! Continuing execution ...");
         //     esp_ota_mark_app_valid_cancel_rollback();
         // } else {
         //     ESP_LOGE(TAG, "Diagnostics failed! Start rollback to the previous version ...");
         //     esp_ota_mark_app_invalid_rollback_and_reboot();
         // }
       }
     }
   
     
       // esp_err_t err = esp_ota_set_boot_partition(update_partition);
       // if (err != ESP_OK) 
       //   ESP_LOGE(TAG, "esp_ota_set_boot_partition failed (%s)!", esp_err_to_name(err));
       // else
       //   ESP_LOGI(TAG, "Prepare to restart system!");
   }
   
   
   
   static void appMainInit(void)
   {
       keyDisplayInit();//uart1 gpio4 gpio5 must before
       atCmdInit();
       iotMainInit();
   }
   static void appMainPro(void)
   {
     keyDisplayPro();
     atCmdPro();
     iotMainPro();
   }
   
   void app_main(void)
   {
       appBootInit();
       appMainInit();
       appMainPro();
       ESP_LOGI(TAG, "app_main end");
   }
