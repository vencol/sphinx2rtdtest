
.. _program_listing_file_G__code_esp_code_freezer_main_freezerdrv_drv_adc.c:

Program Listing for File drv_adc.c
==================================

|exhale_lsh| :ref:`Return to documentation for file <file_G__code_esp_code_freezer_main_freezerdrv_drv_adc.c>` (``G:\code\esp\code\freezer\main\freezerdrv\drv_adc.c``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /* ADC1 Example
   
      This example code is in the Public Domain (or CC0 licensed, at your option.)
   
      Unless required by applicable law or agreed to in writing, this
      software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
      CONDITIONS OF ANY KIND, either express or implied.
   */
   #include <stdio.h>
   #include <string.h>
   #include "freertos/FreeRTOS.h"
   #include "freertos/task.h"
   #include "driver/gpio.h"
   #include "driver/adc.h"
   #include "esp_adc_cal.h"
   #include "drv_adc.h"
   //3139mv 1350
   #define DEFAULT_VREF            1550        //Use adc2_vref_to_gpio() to obtain a better estimate
   #define NO_OF_SAMPLES           64          //Multisampling
   #define INPUT_VOLTAGE_LEVEL     (ADC_ATTEN_DB_11)     
   #define ADC_BIT_WIDTH           (ADC_WIDTH_BIT_12)  
   static const unsigned short ntc_table[NTC_TABLE_LEN] = {//-40 ~ 40 - 60 3988,3936
       3996, 3960, 3926, 3890, 3858, 3824, 3788, 3754, 3716, 3676, 
       3638, 3600, 3552, 3514, 3474, 3434, 3392, 3350, 3312, 3268, 
       3226, 3184, 3142, 3100, 3052, 3006, 2960, 2916, 2870, 2824, 
       2778, 2730, 2682, 2634, 2584, 2532, 2480, 2432, 2382, 2332, 
       2284, 2230, 2180, 2128, 2078, 2024, 1972, 1922, 1872, 1822, 
       1774, 1724, 1676, 1626, 1582, 1530, 1482, 1438, 1394, 1350, 
       1306, 1262, 1222, 1182, 1142, 1104, 1066, 1028,  990,  952, 
        918,  886,  852,  820,  790,  760,  732,  704,  678,  652, 
        626,  602,  578,  556,  534,  510,  488,  466,  448,  430, 
        412,  394,  376,  360,  344,  330,  314,  300,  286,  272, 
        258,  244,  232,  222,  210,  200,  190,  180,  170,  160, 
        150,  140,  132,  124,  116,  108
   
       // 3988,3936,3888,3838,3792,3740,3690,3640,3588,3536,
       // 3484,3426,3372,3318,3260,3204,3148,3090,3032,2972,
       // 2914,2850,2788,2730,2668,2608,2544,2484,2422,2360,
       // 2292,2230,2168,2106,2048,1984,1924,1866,1806,1748,
       // 1686,1632,1574,1522,1468,1416,1366,1312,1266,1218,
       // 1170,1122,1078,1034, 994, 954, 916, 880, 842, 806,
       //  772, 738, 706, 678, 648, 622, 594, 560, 534, 510,
       //  486, 466, 444, 422, 400, 380, 360, 342, 324, 308, 60
   };
   
   #pragma pack(1)
   typedef struct {
     unsigned int      adcmax      : 12;
     unsigned int      adcmin      : 12;
     unsigned int      adcChannel  : 8;
     unsigned short    adcReaded;
     unsigned short    adcValue;
     unsigned short    lastValue;
     unsigned char     adcCounter;
     unsigned char     adcCoutTarget;
   } adc_freezer_t;
   #pragma pack()
   static adc_freezer_t adc_freezer[MAX_ADC_CHANNEL];
   
   
   static signed short findNtcTable(const unsigned short *table,unsigned short tablen,unsigned short dat)//
   {  
       unsigned short st=0,m=0,ed=tablen-1; 
       if(dat <= table[ed]) 
           return NTC_RANGE_ERROR;  
       else if(dat >= table[st]) 
           return NTC_RANGE_ERROR;  
       while(st < ed)  
       {  
           m = (st+ed)/2 ;  
           if(dat < table[m] && dat >= table[m+1]) 
               break ;  
           if(dat < table[m]) 
               st = m ;  //ed = m ;                    
           else 
               ed = m ;//st = m ;   
       }  
       if(st > ed ) 
           return NTC_ERROR;   
       return m;  
   }  
   
   
   // static esp_adc_cal_characteristics_t *adc_chars;
   // static void checkEfuse(void)
   // {
   //     //Check TP is burned into eFuse
   //     if (esp_adc_cal_check_efuse(ESP_ADC_CAL_VAL_EFUSE_TP) == ESP_OK) {
   //         printf("eFuse Two Point: Supported\n");
   //     } else {
   //         printf("eFuse Two Point: NOT supported\n");
   //     }
   
   //     //Check Vref is burned into eFuse
   //     if (esp_adc_cal_check_efuse(ESP_ADC_CAL_VAL_EFUSE_VREF) == ESP_OK) {
   //         printf("eFuse Vref: Supported\n");
   //     } else {
   //         printf("eFuse Vref: NOT supported\n");
   //     }
   // }
   
   // static void printCharValType(esp_adc_cal_value_t val_type)
   // {
   //     if (val_type == ESP_ADC_CAL_VAL_EFUSE_TP) {
   //         printf("Characterized using Two Point Value\n");
   //     } else if (val_type == ESP_ADC_CAL_VAL_EFUSE_VREF) {
   //         printf("Characterized using eFuse Vref\n");
   //     } else {
   //         printf("Characterized using Default Vref\n");
   //     }
   // }
   
   int setAdcOperation(unsigned char channel, char type)
   {
       if(channel >= MAX_ADC_CHANNEL)
           return 0;
       if(type == NTC_CONVER_GET){
       int ret=0;
   // printf("adcCounter: %d\t adcReaded: %d\t adcValue: %d\t\n", adc_freezer[channel].adcCounter, adc_freezer[channel].adcReaded, adc_freezer[channel].adcValue);
           if(adc_freezer[channel].adcCounter == 0 && adc_freezer[channel].adcReaded && adc_freezer[channel].adcValue){
               ret = findNtcTable(ntc_table, NTC_TABLE_LEN, adc_freezer[channel].adcValue);
   // printf("adcValue : %d pos: %d\n", adc_freezer[channel].adcValue, ret);
               if(ret == NTC_RANGE_ERROR || ret == NTC_ERROR)
                   return ret;
               // else if(ret + 2 == NTC_TABLE_LEN)   
               //     ret = 400 + (ntc_table[NTC_TABLE_LEN - 2] - adc_freezer[channel].adcValue) * 200 / (ntc_table[NTC_TABLE_LEN - 2] - ntc_table[NTC_TABLE_LEN - 1]);
               else
                   ret = ret * 10 + (ntc_table[ret] - adc_freezer[channel].adcValue) * 10 / (ntc_table[ret] - ntc_table[ret+1]) + NTC_CONVER_OFFSET * 10;
               return ret;
           }
           else
               return NTC_ERROR;
       }
       else if(type == NTC_CONVER_START){
           adc_freezer[channel].adcCounter     = 1;
           adc_freezer[channel].adcReaded      = 0;
           adc_freezer[channel].adcmax         = 0;
           adc_freezer[channel].adcmin         = 0;
       }
       else if(type == NTC_CONVER_STOP){
           adc_freezer[channel].adcCounter     = 0;
           adc_freezer[channel].adcReaded      = 0;
           adc_freezer[channel].adcmax         = 0;
           adc_freezer[channel].adcmin         = 0;
       }
       return 0;
   }
   void updateAdcValue(void)
   {
       static unsigned char pertime=0;
       int i=-1, adctemp=0;
       if(++pertime > 20)//20*10ms
       {
           pertime = 0;
           i = NTC2_ADC_STRUCT;
       }
       else if(pertime == 10)
           i = NTC1_ADC_STRUCT;
       if(i != -1)
       {
           if(adc_freezer[i].adcCounter)
           {
               adctemp = adc1_get_raw( (adc1_channel_t)adc_freezer[i].adcChannel );
               if(adc_freezer[i].adcmax < adctemp)
                   adc_freezer[i].adcmax = adctemp;
               if(adc_freezer[i].adcmin > adctemp || adc_freezer[i].adcCounter == 1)//the min is first
                   adc_freezer[i].adcmin = adctemp;
               adc_freezer[i].adcReaded += adctemp;
   // printf("adcCounter : %d lastValue : %d adcReaded: %d adctemp: %d adcmax: %d adcmin: %d\n", adc_freezer[i].adcCounter, adc_freezer[i].lastValue, 
       // adc_freezer[i].adcReaded, adctemp, adc_freezer[i].adcmax, adc_freezer[i].adcmin);
               if(++adc_freezer[i].adcCounter > adc_freezer[i].adcCoutTarget)
               {
                   // adc_freezer[i].adcCounter   = 1;//0;
                   adc_freezer[i].adcCounter   = 0;//0;
                   adc_freezer[i].adcReaded = adc_freezer[i].adcReaded - adc_freezer[i].adcmax - adc_freezer[i].adcmin;
                   adctemp = adc_freezer[i].adcReaded / (adc_freezer[i].adcCoutTarget - 2);
                   if(adctemp < ntc_table[0] && adctemp > ntc_table[NTC_TABLE_LEN-1]){
                       adc_freezer[i].adcValue = (adc_freezer[i].lastValue + adctemp) / 2;
                       adc_freezer[i].lastValue = adctemp;
                   }
                   else
                       adc_freezer[i].adcValue = adctemp;
                   
                   // uint32_t voltage = esp_adc_cal_raw_to_voltage(adc_freezer[i].adcValue, adc_chars);
                   // printf("ch : %d Raw: %d\tVoltage: %dmV\n", adc_freezer[i].adcChannel, adc_freezer[i].adcValue, voltage);
                   if(adc_freezer[i].adcValue == 0){
                       adc_freezer[i].adcReaded = 1;
                       adc_freezer[i].adcValue = 1;
                   }
               }
           }
       }
   }
   void drvAdcInit(void)
   {
       //Check if Two Point or Vref are burned into eFuse
       // checkEfuse();
   
       // //Characterize ADC
       // adc_chars = calloc(1, sizeof(esp_adc_cal_characteristics_t));
       // esp_adc_cal_value_t val_type = esp_adc_cal_characterize(ADC_UNIT_1, INPUT_VOLTAGE_LEVEL, ADC_BIT_WIDTH, DEFAULT_VREF, adc_chars);
       // printCharValType(val_type);
   
       //Configure ADC
       adc1_config_width(ADC_BIT_WIDTH);
       memset(adc_freezer, 0, sizeof(adc_freezer_t) * MAX_ADC_CHANNEL);
   
       adc_freezer[NTC1_ADC_STRUCT].adcChannel     = NTC1_ADC_CHANNEL;
       adc_freezer[NTC1_ADC_STRUCT].adcCoutTarget  = NTC1_ADC_DEFAULT_COUNTER;
       adc1_config_channel_atten(adc_freezer[NTC1_ADC_STRUCT].adcChannel, INPUT_VOLTAGE_LEVEL);
   
       adc_freezer[NTC2_ADC_STRUCT].adcChannel     = NTC2_ADC_CHANNEL;
       adc_freezer[NTC2_ADC_STRUCT].adcCoutTarget  = NTC2_ADC_DEFAULT_COUNTER;
       adc1_config_channel_atten(adc_freezer[NTC2_ADC_STRUCT].adcChannel, INPUT_VOLTAGE_LEVEL);
   
   }
