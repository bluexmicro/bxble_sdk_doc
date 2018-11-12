==============================================
BLE Mesh SDK 配置说明
==============================================

配置步骤
==============================================
* 1. 打开广播31字节特性

  相关文件：  **bx_sys_config.h**

  相关代码：

.. code:: c

   #define ENABLE_ADV_PAYLOD_31BYTE_PATCH

* 2. 关闭设备休眠

  相关文件：  **bx_sys_config.h**

  相关代码：

.. code:: c

    #define DEEP_SLEEP_ENABLE {0}

* 3. 去除 ble profiles  config

  相关文件：  **bx_prf_config.h**

  相关代码：

.. code:: c

	 /*
	#define CFG_PRF_PXPM  
	#define CFG_PRF_PXPR  
	#define CFG_PRF_FMPL  
	#define CFG_PRF_FMPT  
	#define CFG_PRF_HTPC  
	#define CFG_PRF_HTPT  
	#define CFG_PRF_DISC  
	#define CFG_PRF_DISS  
	#define CFG_PRF_BLPC  
	#define CFG_PRF_BLPS  
	#define CFG_PRF_HRPC  
	#define CFG_PRF_HRPS  
	#define CFG_PRF_TIPC  
	#define CFG_PRF_TIPS  
	#define CFG_PRF_SCPPC  
	#define CFG_PRF_SCPPS  
	#define CFG_PRF_BASC  
	#define CFG_PRF_BASS  
	#define CFG_PRF_HOGPD  
	#define CFG_PRF_HOGPBH  
	#define CFG_PRF_HOGPRH  
	#define CFG_PRF_GLPC  
	#define CFG_PRF_GLPS  
	#define CFG_PRF_RSCPC  
	#define CFG_PRF_RSCPS  
	#define CFG_PRF_CSCPC  
	#define CFG_PRF_CSCPS  
	#define CFG_PRF_CPPC  
	#define CFG_PRF_CPPS  
	#define CFG_PRF_LANC  
	#define CFG_PRF_LANS  
	#define CFG_PRF_ANPC  
	#define CFG_PRF_ANPS  
	#define CFG_PRF_PASPC  
	#define CFG_PRF_PASPS  
	#define CFG_PRF_BXOTAS
	*/


