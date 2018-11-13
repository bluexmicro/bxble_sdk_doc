Application Configuration Guide
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*该配置既适用于 新版本sdk ( sdk-1.1及其以上)*

* 用户在开发应用程序时，在开发之初需要在bx_app_config.h头文件中确定一些系统配置、外设配置以及应用配置，这个头文件在SDK的app目录的config目录下：

  ..  image:: img/application_config_guide1.png

  bx_app_config.h是一个sdk下管理所有示例应用配置的头文件，该文件简化了用户需要到多个配置头文件去修改配置的过程，例如其中一个应用的配置部分：

  ..  code:: c

		#ifdef B_APP_BLE_OSAPP_TEMPLATE
			#define OSAPP_TEMPLATE                                              1
			#define CFG_APP_DIS                                                 1
			#define CFG_PRF_DISS                                                1
			#define CFG_PRF_BXOTAS                                              1
		#endif/*B_APP_BLE_OSAPP_TEMPLATE*/

* 	应用需要配置的常见参数如下：

    * 用户Profile的定义。用户可以根据自己的需求删减不需要的Profile，也可以增加自定义的Profile。

      **默认 profile 全部是 0，用户应用是需要配置 1**	
		
	..  code:: c
	
		//#define CFG_PRF_PXPM                            1
		//#define CFG_PRF_PXPR                            1
		//#define CFG_PRF_FMPL                            1
		//#define CFG_PRF_FMPT                            1
		//#define CFG_PRF_HTPC                            1
		//#define CFG_PRF_HTPT                            1
		//#define CFG_PRF_DISC                            1
		//#define CFG_PRF_DISS                            1
		//#define CFG_PRF_BLPC                            1
		//#define CFG_PRF_BLPS                            1
		//#define CFG_PRF_HRPC                            1
		//#define CFG_PRF_HRPS                            1
		//#define CFG_PRF_TIPC                            1
		//#define CFG_PRF_TIPS                            1
		//#define CFG_PRF_SCPPC                           1
		//#define CFG_PRF_SCPPS                           1
		//#define CFG_PRF_BASC                            1
		//#define CFG_PRF_BASS                            1
		//#define CFG_PRF_HOGPD                           1
		//#define CFG_PRF_HOGPBH                          1
		//#define CFG_PRF_HOGPRH                          1
		//#define CFG_PRF_GLPC                            1
		//#define CFG_PRF_GLPS                            1
		//#define CFG_PRF_RSCPC                           1
		//#define CFG_PRF_RSCPS                           1
		//#define CFG_PRF_CSCPC                           1
		//#define CFG_PRF_CSCPS                           1
		//#define CFG_PRF_CPPC                            1
		//#define CFG_PRF_CPPS                            1
		//#define CFG_PRF_LANC                            1
		//#define CFG_PRF_LANS                            1
		//#define CFG_PRF_ANPC                            1
		//#define CFG_PRF_ANPS                            1
		//#define CFG_PRF_PASPC                           1
		//#define CFG_PRF_PASPS                           1
		//#define CFG_PRF_BXOTAS                          1
		

	* 用户系统参数配置 

	   **默认参数配置如下**	


	..  code:: c

		#define HW_BX_VERSION                                       00
		#define HW_ECC_PRESENT                                      1
		#define CFG_FREERTOS_SUPPORT                                1
		#define CFG_RF_APOLLO                                       1
		#define CFG_ON_CHIP                                         1
		#define CFG_SYS_LOG                                         1
		#define CFG_DYNAMIC_UPDATE                                  1
		#define ENABLE_ADV_PAYLOD_31BYTE_PATCH                      0
		#define PATCH_SKIP_H4TL_READ_START                          0
		#define BX_VERF                                             0
		#define VBAT_MILLIVOLT                                      3300
		#define VDD_AWO_SLEEP_MILLIVOLT                             800 // 950/900/850/800
		#define VDD_SRAM_SLEEP_MILLIVOLT                            650 // 950/900/850/800/750/700/650/600
		#define FLASH_XIP                                           1
		#define LOCAL_NVDS                                          1
		#define MAIN_CLOCK                                          32000000 //16000000  //96000000
		/*------------- NVDS ---------- */
		#define BX_DEV_NAME                                         {'A','P','O','L','L','O'}
		#define BX_DEV_ADDR                                         {0x99,0x22,0x33,0x44,0x55,0x66}
		#define DEEP_SLEEP_ENABLE                                   {1}
		#define EXT_WAKE_UP_ENABLE                                  {1}
		/*------------- Debug ---------- */
		#define DEBUGGER_ATTACHED                                   1
		#define FREERTOS_WAKEUP_DELAY                               900
		#define XTAL_STARTUP_TIME                                   10
		#define LDO_3V1_OUTPUT_SLEEP_RET                            1
		#define LDO_1V8_OUTPUT_SLEEP_RET                            1
		#define VDD_1V8_SLEEP_LDO1                                  1
		#define DIG_VOLTAGE_CTRL_BY_RF_REG                          1
		#define RUN_WITHOUT_SLEEP                                   0
		#define BATTERY_VOLTAGE_UPDATE_SECONDS                      10
		#define TEMPERATURE_UPDATE_SECONDS                          10
		#define BYPASS_VOLTAGE                                      3400

		
		
*  用户应用修改默认配置，或者增加应用配置，应该只在bx_app_config.h 文件中修改，不涉及其他文件。
   
   *  例如，在 OSAPP_TEMPLATE 例程中，修改设备名称和 MAC 地址：
   
	   ..  code:: c

			#ifdef B_APP_BLE_OSAPP_TEMPLATE
				#define OSAPP_TEMPLATE                                              1
				#define CFG_APP_DIS                                                 1
				#define CFG_PRF_DISS                                                1
				#define CFG_PRF_BXOTAS                                              1
				#define BX_DEV_NAME                                                 {'0','S','1','1','1','1'}
				#define BX_DEV_ADDR                                                 {0x99,0x11,0x11,0x11,0x11,0x11}
			#endif/*B_APP_BLE_OSAPP_TEMPLATE*/
		
   *  例如，在 OSAPP_TEMPLATE 例程中， 增加用户用到的profile：
   
	   ..  code:: c

			#ifdef B_APP_BLE_OSAPP_TEMPLATE
				#define OSAPP_TEMPLATE                                              1
				#define CFG_APP_DIS                                                 1
				#define CFG_PRF_DISS                                                1
				#define CFG_PRF_BXOTAS                                              1
				#define CFG_PRF_PXPM                            					1
				#define CFG_PRF_PXPR                            					1
			#endif/*B_APP_BLE_OSAPP_TEMPLATE*/
			
   *  例如，在 OSAPP_TEMPLATE 例程中， 修改睡眠配置：
   
	   ..  code:: c

			#ifdef B_APP_BLE_OSAPP_TEMPLATE
				#define OSAPP_TEMPLATE                                              1
				#define CFG_APP_DIS                                                 1
				#define CFG_PRF_DISS                                                1
				#define CFG_PRF_BXOTAS                                              1
				#define DEEP_SLEEP_ENABLE                                          {0}
			#endif/*B_APP_BLE_OSAPP_TEMPLATE*/			


			
* bx_app_config.h 配置文件，可以配置的参数主要分布在以下几个文件中
		
  -   bx_ip_config.h
  
	文件位置在 sdk/ip 目录下
    这个头文件中包含关于协议栈的配置定义，例如是否支持NVDS，BLE的角色定义，最大支持连接数等。由于这里的部分宏定义会直接影响ROM内容，因此用户不需要修改该头文件定义，也不需要关心其内容。


  -   bx_pcb_config.h

  
    文件位置在 sdk/plf/apollo_00 目录下
    这个头文件中主要包含了IC的IO定义。Apollo的IO分为4个Pad组，每组的电压可以独立配置为1.8V/3.3V。用户需要根据自己的需求，将需要的外设配置到符合要求的IO Pad上。

    -   IO电压

        枚举常量VDD_1V8/VDD_3V分别表示IO Pad电压配置为1.8V和3.3V

    -   IO属性

        IO有若干属性，分别定义在枚举常量中：

        NORMAL_MODE_IE: 该IO在IC处于唤醒状态下时，需要打开IE(Input Enable)。IE是硬件的一个功能模块，使能后外部信号才能输入到IC中，因此该属性通常用于配置为输入的IO。

        SLEEP_MODE_IE: 该IO在IC处于睡眠状态下时，需要打开IE。这种配置通常用于外部中断唤醒的IO。

        UTILITY_IO_EN: 通用IO，无IE和retention等其他额外功能要求。

        SLEEP_RET_OUT_EN: 该IO在IC处于睡眠状态时需要维持睡前的状态。比较典型的例子，是IC充当SPI Master角色时，时钟线需要配置该属性。

        SLEEP_RET_OUT_H: 该IO在IC处于睡眠状态时需要维持高电平状态。比较典型的例子，是Uart的TX IO。

        其他IO，配置为GENERAL_PURPOSE_IO即可。

        在Apollo SDK中，已经有一个默认的IO配置，用户只需要根据自己的需求去修改这个配置即可。

  -   bx_sys_config.h

  
	文件位置在 sdk/plf/apollo_00 目录下
	
    该头文件里包含了系统相关的大多数配置，其中用户需要了解的有：

    -   CFG_HW_ECC

        表示使用硬件实现的ECC。考虑到Apollo内置CPU为Cortex-M0+，并不能执行复杂的算法运算，因此此选项通常需要打开。

    -   CFG_FREERTOS_SUPPORT

        表示在FREERTOS的框架下开发客户应用程序。

    -   CFG_SYS_LOG

        表示打开应用通过RTT打印信息的功能。在调试阶段该宏通常需要打开，在某些特殊状况下可以关闭以节省RAM(通常几个KB)。

    -   VBAT_MILLIVOLT

        表示用户程序运行时的IC供电电压。定义这个宏的原因是IO输出电压会和IC供电电压有关，系统需要根据供电电压对IO输出电压做调整。

    -   MAIN_CLOCK

        表示AHB时钟。必须为16000000的整数倍，最高96000000.

    -   BX_DEV_NAME

        表示调试阶段设备蓝牙名称。量产时IC是烧入Flash里的。

    -   BX_DEV_ADDR

        表示调试阶段设备蓝牙地址。和蓝牙名称一样，量产时IC是烧入Flash里的。
        
    -   DEEP_SLEEP_ENABLE

        表示用户程序是否可以进入低功耗睡眠模式。

		
* 对于用户而言，这些是用户需要了解的配置选项，其他选项保持默认值即可。

