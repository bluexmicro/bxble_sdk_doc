=======================
ble broadcaster demo
=======================


示例说明
=======================

* 如何设置 ble broadcaster 角色  

	* 参考:	 `ble 角色设置`_

* 如何处理 ble 协议栈和应用协议栈的信息交互  

	* 参考:	 `ble 协议栈和应用协议栈的信息交互`_

* 如何更新广播内容  

	* 参考:	 `ble 更新广播内容`_


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
********************************

.. code:: c

	void osapp_ble_stack_run(void)
	{
		 osapp_ble_stack_data_init();
		 osapp_set_dev_config(GAP_ROLE_BROADCASTER,GAPM_CFG_ADDR_PUBLIC,GAPM_PAIRING_LEGACY,BLE_PERIPHERAL_MAX_MTU);
		 LOG(LOG_LVL_INFO,"osapp_ble_stack_run\n");
	}

**1. 初始化内部数据**
------------------------------------------------------

.. code:: c

	static void osapp_ble_stack_data_init(void)
	{
		broadcaster_env.Timer =  xTimerCreate("ADVTimer",APP_ADV_UPDATE_PERIOD,pdTRUE,(void *) 0,osapp_advdata_change_handler);
		BX_ASSERT(broadcaster_env.Timer!=NULL);

		broadcaster_env.test_num = 0;
	}
	
**2. 设置设备的 角色设置**
------------------------------------------------------

.. code:: c

	osapp_set_dev_config(GAP_ROLE_BROADCASTER,GAPM_CFG_ADDR_PUBLIC,GAPM_PAIRING_LEGACY,BLE_PERIPHERAL_MAX_MTU);

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

**3. 协议栈开始完整运行**
------------------------------------------------------

。。。。


_`ble 协议栈和应用协议栈的信息交互`
==============================================

实现消息交互的处理函数
********************************

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
	};

	const osapp_msg_handler_info_t handler_info = ARRAY_INFO(handler_table);

	
_`ble 更新广播内容`
==============================================

广播开启后，内部开启了一个1秒的软件定时器，定时更新广播内容
****************************************************************

.. code:: c

	static int32_t osapp_update_advertise_data(uint8_t index)
	{
		struct gapm_update_advertise_data_cmd *cmd = AHI_MSG_ALLOC(GAPM_UPDATE_ADVERTISE_DATA_CMD,TASK_ID_GAPM, gapm_update_advertise_data_cmd);

		cmd->operation = GAPM_UPDATE_ADVERTISE_DATA;
		cmd->adv_data_len = 0;

		memcpy(&cmd->adv_data[cmd->adv_data_len],
			   OSAPP_BX_ADV_DATA_UUID, OSAPP_BX_ADV_DATA_UUID_LEN);
		cmd->adv_data_len += OSAPP_BX_ADV_DATA_UUID_LEN;

		cmd->adv_data[cmd->adv_data_len] = OSAPP_DATA_TOTAL_LEN-2-OSAPP_BX_ADV_DATA_UUID_LEN+1;
		cmd->adv_data[cmd->adv_data_len+1] = '\x08';

		index %= 4;//(0~3)
		index +=1;//(1~4)

		switch (index)
		{
			case 1:
				LOG(LOG_LVL_INFO,"change1\n");
				memcpy(&cmd->adv_data[6],OSAPP_DEVICE_NAME_CHG1,OSAPP_DEVICE_NAME_CHG1_LEN);
				cmd->adv_data_len = OSAPP_DATA_TOTAL_LEN;
				break;
			case 2:
				LOG(LOG_LVL_INFO,"change2\n");
				memcpy(&cmd->adv_data[6],OSAPP_DEVICE_NAME_CHG2,OSAPP_DEVICE_NAME_CHG2_LEN);
				cmd->adv_data_len = OSAPP_DATA_TOTAL_LEN;
				break;
			case 3:
				LOG(LOG_LVL_INFO,"change3\n");
				memcpy(&cmd->adv_data[6],OSAPP_DEVICE_NAME_CHG3,OSAPP_DEVICE_NAME_CHG3_LEN);
				cmd->adv_data_len = OSAPP_DATA_TOTAL_LEN;
				break;
			case 4:
				LOG(LOG_LVL_INFO,"change4\n");
				memcpy(&cmd->adv_data[6],OSAPP_DEVICE_NAME_CHG4,OSAPP_DEVICE_NAME_CHG4_LEN);
				cmd->adv_data_len = OSAPP_DATA_TOTAL_LEN;
				break;
			default:
				break;
		}


		cmd->scan_rsp_data_len  = OSAPP_SCNRSP_DATA_CHG_LEN;
		memcpy(&cmd->scan_rsp_data[0],OSAPP_SCNRSP_DATA_CHG,cmd->scan_rsp_data_len);

		return osapp_msg_build_send(cmd,sizeof(struct gapm_update_advertise_data_cmd));

	}
