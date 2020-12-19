
.. _program_listing_file_main_freezerdrv_drv_1650.c:

Program Listing for File drv_1650.c
===================================

|exhale_lsh| :ref:`Return to documentation for file <file_main_freezerdrv_drv_1650.c>` (``main\freezerdrv\drv_1650.c``)

.. |exhale_lsh| unicode:: U+021B0 .. UPWARDS ARROW WITH TIP LEFTWARDS

.. code-block:: cpp

   /* i2c - Example
   
      For other examples please check:
      https://github.com/espressif/esp-idf/tree/master/examples
   
      See README.md file to get detailed usage of this example.
   
      This example code is in the Public Domain (or CC0 licensed, at your option.)
   
      Unless required by applicable law or agreed to in writing, this
      software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
      CONDITIONS OF ANY KIND, either express or implied.
   */
   #include <stdio.h>
   #include "driver/i2c.h"
   #include "drv_1650.h"
   
   
   // drvTm1650Init();
   // tm1650SetDispaly( TM1650_DISPLAY_VALUE(LEVEL_1, TM1650_SEGMENT_8, TM1650_DISPLAY_ON) );
   // uint8_t disbuftest[4]={0x11, 0x22, 0x33, 0x44};
   // tm1650SetDispalyData(disbuftest, 4);
   #define _I2C_NUMBER(num) I2C_NUM_##num
   #define I2C_NUMBER(num) _I2C_NUMBER(num)
   
   #define I2C_MASTER_SCL_IO                   5               //19
   #define I2C_MASTER_SDA_IO                   4               //18
   #define I2C_MASTER_NUM I2C_NUMBER(1) 
   #define I2C_MASTER_FREQ_HZ                  40000        
   #define I2C_MASTER_TX_BUF_DISABLE           0                           
   #define I2C_MASTER_RX_BUF_DISABLE           0                           
   #define ESP_SLAVE_ADDR                      0x28 
   #define WRITE_BIT                           I2C_MASTER_WRITE              
   #define READ_BIT                            I2C_MASTER_READ                
   #define ACK_CHECK_EN                        0x1                        
   #define ACK_CHECK_DIS                       0x0                       
   #define ACK_VAL                             0x0                             
   #define NACK_VAL                            0x1                            
   #define TM1650_READ_KEY_ADDR                0X49                           
   #define TM1650_SET_DIASPLAY_ADDR            0X48                           
   #define TM1650_SET_DIASPLAY_DATAADDR        0X68                           
   static uint8_t LSB2MSB(uint8_t data)
   {
       // printf("data in 0x%0X", data);
       // data = data<<4 | data>>4;
       // data = (data&0x55)<<1 | (data&0xaa)>>1;
       // data = (data&0x33)<<2 | (data&0xcc)>>2;
       // printf("data in 0x%0X", data);
       return data;
   }
   static int i2cMasterSetAddrData(uint8_t addr, uint8_t data)
   {
       i2c_cmd_handle_t cmd = i2c_cmd_link_create();
       i2c_master_start(cmd);
       i2c_master_write_byte(cmd, LSB2MSB(addr), ACK_CHECK_EN);
       i2c_master_write_byte(cmd, LSB2MSB(data), ACK_CHECK_EN);
       i2c_master_stop(cmd);
       esp_err_t ret = i2c_master_cmd_begin(I2C_MASTER_NUM, cmd, 100 / portTICK_RATE_MS);
       // printf("i2cMasterSetAddrData init %d\r\n", ret);
       i2c_cmd_link_delete(cmd);
       return ret;
   }
   int tm1650SetDispaly(uint8_t data)
   {
       return i2cMasterSetAddrData(TM1650_SET_DIASPLAY_ADDR, data);
   }
   int tm1650SetDispalyData(uint8_t *data, uint8_t size)
   {
       // static char testper=0;
       // if(++testper < 100)
       //     return 0;
       // testper = 0;
       if (data == NULL || size == 0) {
           return -1;
       }
       // *(data+1) = 0;
       // printf("get ke435y 0x%0X\r\n", *(data+1));
       // tm1650ReadKey(data+1);
       // printf("get key 0x%0X\r\n", *(data+1));
       for(int i=0; i<size; i++)
       {
           if( i2cMasterSetAddrData(TM1650_SET_DIASPLAY_DATAADDR + 2*i, *(data+i)) != 0 )
               return -1;
       }
       return 0;
   }
   int tm1650ReadKey(uint8_t *key)
   {
       if (key == NULL) {
           return -1;
       }
       i2c_cmd_handle_t cmd = i2c_cmd_link_create();
       i2c_master_start(cmd);
       i2c_master_write_byte(cmd, LSB2MSB(TM1650_READ_KEY_ADDR), ACK_CHECK_EN);
       i2c_master_read(cmd, key, 1, ACK_VAL);
       i2c_master_stop(cmd);
       esp_err_t ret = i2c_master_cmd_begin(I2C_MASTER_NUM, cmd, 10 / portTICK_RATE_MS);
       i2c_cmd_link_delete(cmd);
       return ret;
   }
   
   int drvTm1650Init(void)
   {
       int ret=0;
       int i2c_master_port = I2C_MASTER_NUM;
       i2c_config_t conf;
       conf.mode = I2C_MODE_MASTER;
       conf.sda_io_num = I2C_MASTER_SDA_IO;
       conf.sda_pullup_en = GPIO_PULLUP_ENABLE;
       conf.scl_io_num = I2C_MASTER_SCL_IO;
       conf.scl_pullup_en = GPIO_PULLUP_ENABLE;
       conf.master.clk_speed = I2C_MASTER_FREQ_HZ;
       ret = i2c_param_config(i2c_master_port, &conf);
       printf("i2c_param_config init %d\r\n", ret);
       ret = i2c_driver_install(i2c_master_port, conf.mode, I2C_MASTER_RX_BUF_DISABLE, I2C_MASTER_TX_BUF_DISABLE, 0);
       printf("i2c_driver_install init %d\r\n", ret);
       return ret;
   }
