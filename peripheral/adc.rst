============
ADC控制器
============


    BX2400共有1个ADC转换器，具有6路复用通道和一个battery monitor，可以测量来自6路外模拟电压信号，以及VDD_BAT的电压。

***************
【基本特性】
***************
ADC控制器的主要特性如下表所示：

- 转换精度可以达到8-bit相当。
- 6路ADC数据采集，均为独立的模拟IO口，不占用数字IO口。
- 支持Battory Monitor和Touch。




***************
【程序操作】
***************




第一步：配置参数
=============================

配置参数在app_adc_utils.h头文件中进行参数的配置：

1：将 ADC_GPADC_SINGLE_END_MODE_EN 配置为1

2：将 ADC_GPADC_DIFFERENTIAL_MODE_EN 配置为0

3：如果版本为SDK2.1或更高，则忽略本条目。
如果版本低于SDK2.1，需要配置 ADC_GPADC_RO_TRIM 这个宏。
如果使用的内置FLASH的芯片，需要配置为 ADC_TRIM_SIP_FLASH 。
如果使用的外置FLASH的芯片，需要配置为 ADC_TRIM_EXT_FLASH 。



第二步：初始化ADC
==============================

ADC初始化代码为app_adc_util_init()函数。该函数在main函数(裸跑模式下)，或者osapp_task函数(FreeRTOS模式下)，已经调用完毕，用户无需额外进行调用。



第三步：读取ADC操作。
==============================

读取ADC通道数据
----------------------------

读取函数为 uint32_t app_adc_gpadc_single_end(uint8_t channel)

参数含义：channel即通道号，取值范围0~5，对应P30~P35的模拟IO口。需要注意实际芯片中是否有对应的IO口。

返回值含义：返回值就是电压值，单位为mV。



读取battery monitor数据
----------------------------

读取函数为 uint32_t app_adc_battery(void)

返回值含义：返回值就是电压值，单位为uV。








