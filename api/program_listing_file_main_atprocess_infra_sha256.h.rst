
.. _program_listing_file_main_atprocess_infra_sha256.h:

Program Listing for File infra_sha256.h
=======================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_atprocess_infra_sha256.h>` (``main\atprocess\infra_sha256.h``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /*
    * Copyright (C) 2015-2018 Alibaba Group Holding Limited
    */
   
   #ifndef _IOTX_COMMON_SHA256_H_
   #define _IOTX_COMMON_SHA256_H_
   
   #include <stdint.h>
   
   #define SHA256_DIGEST_LENGTH            (32)
   #define SHA256_BLOCK_LENGTH             (64)
   #define SHA256_SHORT_BLOCK_LENGTH       (SHA256_BLOCK_LENGTH - 8)
   #define SHA256_DIGEST_STRING_LENGTH     (SHA256_DIGEST_LENGTH * 2 + 1)
   
   typedef struct {
       uint32_t total[2];          
       uint32_t state[8];          
       unsigned char buffer[64];   
       int is224;                  
   } iot_sha256_context;
   
   typedef union {
       char sptr[8];
       uint64_t lint;
   } u_retLen;
   
   void utilsSha256Init(iot_sha256_context *ctx);
   
   void utilsSha256Free(iot_sha256_context *ctx);
   
   
   void utilsSha256Starts(iot_sha256_context *ctx);
   
   void utilsSha256Update(iot_sha256_context *ctx, const unsigned char *input, uint32_t ilen);
   
   void utilsSha256Finish(iot_sha256_context *ctx, uint8_t output[32]);
   
   /* Internal use */
   void utilsSha256Process(iot_sha256_context *ctx, const unsigned char data[64]);
   
   void utilsSha256(const uint8_t *input, uint32_t ilen, uint8_t output[32]);
   
   void utilsHmacSha256(const uint8_t *msg, uint32_t msg_len, const uint8_t *key, uint32_t key_len, uint8_t output[32]);
   
   #endif
   
   
