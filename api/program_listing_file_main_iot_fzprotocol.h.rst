
.. _program_listing_file_main_iot_fzprotocol.h:

Program Listing for File fzprotocol.h
=====================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_iot_fzprotocol.h>` (``main\iot\fzprotocol.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   #ifndef __FZ_PROTOCOL__
   #define __FZ_PROTOCOL__
   
   
   
   //send
   void fzReportRegisterInfo(void);
   void fzReportAttrsInfo(char step);
   void fzReportAlarmInfo(void);
   void fzReportShadowUpdate(int vid);
   void fzRespondAlarmQuery(const char* messageId, unsigned int respondbit);
   void fzRespondAttrsQuery(const char* messageId, unsigned int respondbit);
   void fzRespondShadowUpdate(unsigned int setbit, int vid);
   
   
   //rec
   void fzReceiveShadowMsg(char * rbuf, int len);
   void fzReceiveRrpcMsg(char * rbuf, int len, const char* messageId);
   void fzReceiveUpgradeMsg(char * rbuf, int len);
   
   
   #endif
   
