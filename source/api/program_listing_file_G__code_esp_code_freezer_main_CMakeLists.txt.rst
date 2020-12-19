
.. _program_listing_file_G__code_esp_code_freezer_main_CMakeLists.txt:

Program Listing for File CMakeLists.txt
=======================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_CMakeLists.txt>` (``G:\code\esp\code\freezer\main\CMakeLists.txt``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   
   
   #set(COMPONENT_PRIV_INCLUDEDIRS)
   #set(COMPONENT_ADD_INCLUDEDIRS)
   #set(COMPONENT_SRCDIRS)
   #set(COMPONENT_REQUIRES)
   #set(COMPONENT_PRIV_REQUIRES )
   #set(COMPONENT_SRCS)
   #set (EXTRA_COMPONENT_DIRS ".")
   
   set(COMPONENT_PRIV_INCLUDEDIRS  .
                                   freezerdrv
                                   atprocess
                                   wifible
                                   iot
   )
   set(COMPONENT_ADD_INCLUDEDIRS   .
   )
   
   set(COMPONENT_SRCDIRS   .
                           freezerdrv 
                           atprocess  
                           wifible 
                           iot 
   )     
          
   set(COMPONENT_REQUIRES  "bt"  
                           "json" 
                           "nvs_flash"  
                           "esp_adc_cal"
                           "app_update"
   )
   set(COMPONENT_PRIV_REQUIRES )  
   set(COMPONENT_SRCS)
   
   register_component()
    
