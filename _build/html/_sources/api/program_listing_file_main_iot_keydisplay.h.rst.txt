
.. _program_listing_file_main_iot_keydisplay.h:

Program Listing for File keydisplay.h
=====================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_iot_keydisplay.h>` (``main\iot\keydisplay.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __IOT_DISPLAY__
   #define __IOT_DISPLAY__
   
   
   #pragma pack(1)
   typedef struct {
       unsigned int    reskey      : 8;
   
       unsigned int    digital0    : 7;
       unsigned int    point0      : 1;
       unsigned int    digital1    : 7;
       unsigned int    point1      : 1;
       unsigned int    digital2    : 7;
       unsigned int    point2      : 1;
   
       unsigned int    compressorled: 1;
       unsigned int    wifiled     : 1;
       unsigned int    signled     : 1;
       unsigned int    lteled      : 1;
       unsigned int    save        : 2;
       unsigned int    frozengreen : 1;
       unsigned int    frozenred   : 1;
   } display_buf_t;
   #pragma pack()
   typedef enum {
       DISPLAY_NONE,
       DISPLAY_POWERON,
       DISPLAY_3STEMP,
       DISPLAY_LOCKED,
       DISPLAY_LOCALOFF,
       DISPLAY_REMOTEOFF,
       
       DISPLAY_SET_TEMP,
   
       DISPLAY_SELECT_PARAM_C1,
       DISPLAY_SELECT_PARAM_C2,
       DISPLAY_SELECT_PARAM_C3,
       DISPLAY_SELECT_PARAM_C4,
       DISPLAY_SELECT_PARAM_C5,
       DISPLAY_SELECT_PARAM_F1,
   
       DISPLAY_SET_C1,
       DISPLAY_SET_C2,
       DISPLAY_SET_C3,
       DISPLAY_SET_C4,
       DISPLAY_SET_C5,
       DISPLAY_SET_F1,
   
       DISPLAY_ERR_NONE,
       DISPLAY_ERR_NOSIM,
       DISPLAY_ERR_FROZEN,
       DISPLAY_ERR_REFRIGERA,
       DISPLAY_ERR_REGISTER,
       DISPLAY_ERR_ERRSIM,
       DISPLAY_ERR_CSQ,
       DISPLAY_ERR_MCU,
       DISPLAY_ERR_WIFI,
       DISPLAY_ERR_BLE,
       DISPLAY_ERR_SHOW_CSQ,
   
   } display_status_opt_t;
   extern unsigned char    displayStatus;
   extern signed short     displayTemp;
   
   void displayInit(void);
   void displayProcess(void);
   void displaySetRekey(unsigned char key);
   uint8_t keyDisplayUartSend(uint8_t cmd, const char *data, uint8_t len);
   
   void resetDisplay(void);
   int settingQuit(signed char type);
   int setTempDisplay(signed char type);
   int setParamInOut(signed char type);
   int setParamDisplay(signed char type);
   int selectParamDisplay(signed char type);
   int setErrDisplay(signed char type);
   void setDisplay_0(signed char settemp);
   int powerOffLocal(signed char type);
   int powerOffRemote(signed char type);
   int powerOnDisplay(signed char type);
   int canFastDisplay(void);
   void displayTempAlarm(int settime);
   
   
   #define BASE_DISPLAY_TIME         (10)//MS
   #define DISPLAY_REFRESH_TIME       (100 / BASE_DISPLAY_TIME)//MS
   #define SET_OPERATION_TIME        (8000 / BASE_DISPLAY_TIME)
   
   #define DISPLAY_FLASH_500MS       (500 / BASE_DISPLAY_TIME)
   #define DISPLAY_FLASH_1S          (1000 / BASE_DISPLAY_TIME)
   #define DISPLAY_FLASH_NONE        (10000 / BASE_DISPLAY_TIME)
   
   #define DISPLAY_CHAR_b            (0x7c)
   #define DISPLAY_CHAR_C            (0x39)
   #define DISPLAY_CHAR_d            (0x5e)
   #define DISPLAY_CHAR_E            (0x79)
   #define DISPLAY_CHAR_F            (0x71)
   #define DISPLAY_CHAR_H            (0x76)
   #define DISPLAY_CHAR_L            (0x38)
   #define DISPLAY_CHAR_r            (0x50)
   
   #define DISPLAY_TYPE_NONE         (-1)
   #define DISPLAY_TYPE_SET          (0)
   #define DISPLAY_TYPE_ON           (1)
   #define DISPLAY_TYPE_OFF          (2)
   #define DISPLAY_TYPE_ONOVER       (3)
   #define DISPLAY_TYPE_OffOVER      (4)
   #define DISPLAY_TYPE_ERROR        (0x7f)
   #define DISPLAY_TYPE_DIGITAL      (1)
   #define DISPLAY_TYPE_DECIMAL      (DISPLAY_TYPE_DIGITAL + 1)
   #define DISPLAY_TYPE_PARAM1       (DISPLAY_TYPE_DECIMAL + 1)
   #define DISPLAY_TYPE_CSQ          (DISPLAY_TYPE_PARAM1 + 1)
   #define DISPLAY_TYPE_HHH          (DISPLAY_TYPE_CSQ + 1)
   #define DISPLAY_TYPE_LLL          (DISPLAY_TYPE_HHH + 1)
   #define DISPLAY_TYPE_bd           (DISPLAY_TYPE_LLL + 1)
   #define DISPLAY_TYPE_OF           (DISPLAY_TYPE_bd + 1)
   #endif
   
