=======================
ble observer demo
=======================


示例说明
=======================

* 如何设置 ble observer 角色  

	* 参考:	 `ble 角色设置`_

* 如何处理 ble 协议栈和应用协议栈的信息交互  

	* 参考:	 `ble 协议栈和应用协议栈的信息交互`_

* 如何获取扫描到的广播内容  

	* 参考:	 `ble 获取广播内容`_


执行流程
=======================

**1. 用户自己函数入口**

   在 ble_user_main.c 中， osapp_user_main_init()初始化自己数据。
   
**2. 开启ble协议栈调度**

   在用户函数执行完后，调用 osapp_ble_stack_run(),开启ble协议栈调度。

**3. 示例伪代码**

.. code:: c

	void osapp_user_main_init(void)
	{
		///user data init
		{
			//do ...
	
		}
		///ble stack run
		osapp_ble_stack_run();
	
		LOG(LOG_LVL_INFO,"ble user run\n");
	}


_`ble 角色设置`
=======================

设置流程
**************************

.. code:: c

	void osapp_ble_stack_run(void)
	{
		 osapp_ble_stack_data_init();
		 osapp_set_dev_config(GAP_ROLE_OBSERVER,GAPM_CFG_ADDR_PUBLIC,GAPM_PAIRING_LEGACY,BLE_L2CAP_MAX_MTU);
		 LOG(LOG_LVL_INFO,"osapp_ble_stack_run\n");
	}

	
**1. 设置设备的 角色设置**
------------------------------------------------------

.. code:: c

	osapp_set_dev_config(GAP_ROLE_OBSERVER,GAPM_CFG_ADDR_PUBLIC,GAPM_PAIRING_LEGACY,BLE_L2CAP_MAX_MTU);

	static int32_t osapp_set_dev_config(uint8_t role,uint8_t addr_type,uint8_t pairing_mode,uint16_t max_mtu)
	{
		// Set Device configuration
		struct gapm_set_dev_config_cmd* cmd = AHI_MSG_ALLOC(GAPM_SET_DEV_CONFIG_CMD,TASK_ID_GAPM,gapm_set_dev_config_cmd);

		memset(cmd, 0 , sizeof(struct gapm_set_dev_config_cmd));

		cmd->operation = GAPM_SET_DEV_CONFIG;
		cmd->role      = role;

		// Set Data length parameters
		cmd->sugg_max_tx_octets = BLE_MIN_OCTETS;
		cmd->sugg_max_tx_time   = BLE_MIN_TIME;
		cmd->max_mtu = max_mtu;
		cmd->addr_type = addr_type;
		cmd->pairing_mode = pairing_mode;

		return osapp_msg_build_send(cmd, sizeof(struct gapm_set_dev_config_cmd));
	}

**2. 开启设备扫描配置**
------------------------------------------------------

.. code:: c

	static int32_t osapp_start_scan(void)
	{
		struct gapm_start_scan_cmd *cmd = AHI_MSG_ALLOC(GAPM_START_SCAN_CMD,TASK_ID_GAPM,gapm_start_scan_cmd);
		cmd->op.code = GAPM_SCAN_PASSIVE;

		cmd->mode = GAP_OBSERVER_MODE;
		cmd->interval = 0x20;
		cmd->window = 0x20;

		return osapp_msg_build_send(cmd, sizeof(struct gapm_start_scan_cmd));
	}

**3. 协议栈开始完整运行**
------------------------------------------------------

。。。。


_`ble 协议栈和应用协议栈的信息交互`
==============================================

实现消息交互的处理函数
******************************************

.. code:: c

	/**
	 * @brief message and handler table. This define the connection of message and it's callback.
	 */
	static const osapp_msg_handler_table_t handler_table[]=
	{
		[0] =   {KE_MSG_DEFAULT_HANDLER,(osapp_msg_handler_t)osapp_default_msg_handler},
		   ///GAPM event complete
		   {GAPM_CMP_EVT,(osapp_msg_handler_t)osapp_gapm_cmp_evt_handler},
		   ///ble power on ready and should do a reset
		   {GAPM_DEVICE_READY_IND,(osapp_msg_handler_t)osapp_device_ready_ind_handler},
		   ///trigger when master need to read device information uuid 0x1800
		   {GAPC_GET_DEV_INFO_REQ_IND,(osapp_msg_handler_t)osapp_gapc_get_dev_info_req_ind_handler},
		   ///triggered when scanning operation of selective connection establishment procedure receive advertising report information.
		   {GAPM_ADV_REPORT_IND,osapp_gapm_adv_report_ind_handler},
	};

	const osapp_msg_handler_info_t handler_info = ARRAY_INFO(handler_table);

	
_`ble 获取广播内容`
==============================================

收到广播消息后，内部通过 log 打印出了获得的广播内容
************************************************************************************

.. code:: c

	static void osapp_gapm_adv_report_ind_handler(ke_msg_id_t const msgid, void const *param,ke_task_id_t const dest_id,ke_task_id_t const src_id)
	{
		struct adv_report const *report = param;
		uint8_t i=0;

		switch(report->evt_type)
		{
		case ADV_CONN_UNDIR:
			LOG(LOG_LVL_INFO,"ADV_CONN_UNDIR\n");
			break;
		case ADV_CONN_DIR:
			LOG(LOG_LVL_INFO,"ADV_CONN_DIR\n");
			break;
		case ADV_DISC_UNDIR:
			LOG(LOG_LVL_INFO,"ADV_DISC_UNDIR\n");
			break;
		case ADV_NONCONN_UNDIR:
			LOG(LOG_LVL_INFO,"ADV_NONCONN_UNDIR\n");
			break;
		case ADV_CONN_DIR_LDC:
			LOG(LOG_LVL_INFO,"ADV_CONN_DIR_LDC\n");
			break;

		default:
			LOG(LOG_LVL_WARN,"NO THIS TYPE ADV");
			break;
		}

		LOG(LOG_LVL_INFO,"adv_addr_type : %d \n",report->adv_addr_type);//GAPM_CFG_ADDR_PUBLIC
		LOG(LOG_LVL_INFO,"adv_addr: ");
		for(i=0;i<GAP_BD_ADDR_LEN;i++)
			LOG(LOG_LVL_INFO,":0x%02X",report->adv_addr.addr[i]);//GAPM_CFG_ADDR_PUBLIC

		LOG(LOG_LVL_INFO,"\n ADV len : %d \n  ADV data ",report->data_len);
		for(uint8_t i=0;i<report->data_len;i++)
			LOG(LOG_LVL_INFO,":0x%02X",report->data[i]);

		LOG(LOG_LVL_INFO,"\n ADV RSSI : %d \n",report->rssi);
	}

打印的部分日志如下：

.. code:: c

	ADV_NONCONN_UNDIR
	adv_addr_type : 0 
	adv_addr: :0x99:0x11:0x30:0x40:0x59:0x68
	 ADV len : 23 
	  ADV data :0x16:0x2A:0x19:0x89:0xEA:0x44:0xBC:0x2C:0x18:0xE9:0x53:0x5F:0x09:0x28:0xE3:0x7B:0xED:0x1F:0xD7:0xC3:0x35:0x42:0x52
	 ADV RSSI : 186 
	ADV_NONCONN_UNDIR
	adv_addr_type : 1 
	adv_addr: :0x8F:0x11:0x99:0xC9:0xF7:0x24
	 ADV len : 31 
	  ADV data :0x1E:0xFF:0x06:0x00:0x01:0x09:0x20:0x02:0xF7:0xDD:0xE1:0x89:0x90:0x35:0x40:0x8E:0xB3:0x82:0x87:0x38:0x10:0x9A:0xED:0xB9:0x67:0x69:0xE0:0xD6:0x0C:0xDE:0x9F
	 ADV RSSI : 187 
	ADV_NONCONN_UNDIR
	adv_addr_type : 1 
	adv_addr: :0xC8:0xE6:0xA9:0x24:0xA6:0x22
	 ADV len : 31 
	  ADV data :0x1E:0xFF:0x06:0x00:0x01:0x09:0x20:0x02:0x89:0xB1:0x2A:0x4A:0xF8:0xC0:0x13:0x46:0xCA:0x68:0xEF:0xE6:0x77:0xC4:0x0A:0xCA:0x6D:0x7E:0x0F:0x39:0xD6:0x42:0xCD
	 ADV RSSI : 196 

	 