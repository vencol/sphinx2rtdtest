
.. _program_listing_file_main_freezerdrv_drv_adc.h:

Program Listing for File drv_adc.h
==================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_freezerdrv_drv_adc.h>` (``main\freezerdrv\drv_adc.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __DRV_ADC__
   #define __DRV_ADC__
   
   void drvAdcInit(void);
   void updateAdcValue(void);//10ms
   int setAdcOperation(unsigned char channel, char type);
   
   #define NTC1_ADC_STRUCT                     0 
   #define NTC2_ADC_STRUCT                     1 
   #define MAX_ADC_CHANNEL                     2//
   #define NTC1_ADC_CHANNEL                    ADC1_CHANNEL_6//ADC1_GPIO34_CHANNEL
   #define NTC2_ADC_CHANNEL                    ADC1_CHANNEL_7//ADC1_GPIO35_CHANNEL
   #define NTC1_ADC_DEFAULT_COUNTER            10 
   #define NTC2_ADC_DEFAULT_COUNTER            10 
   
   
   #define NTC_CONVER_STOP                     0 
   #define NTC_CONVER_START                    1 
   #define NTC_CONVER_GET                      2 
   #define NTC_ERROR                           (-10000)
   #define NTC_RANGE_ERROR                     (-10001)
   #define NTC_TABLE_LEN                       116    
   #define NTC_CONVER_OFFSET                   (-40) 
   
   #endif
   
