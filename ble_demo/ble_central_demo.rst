=======================
ble central demo
=======================


示例说明
=======================

* 如何设置 ble central 角色  

	* 参考:	 `ble 角色设置`_

* 如何处理 ble 协议栈和应用协议栈的信息交互  

	* 参考:	 `ble 协议栈和应用协议栈的信息交互`_
	
* 如何和peripheral 设备建立连接并通信    

	* 参考:	 `ble 和peripheral连接通信`_


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
****************************

.. code:: c

	void osapp_ble_stack_run(void)
	{
		 osapp_ble_stack_data_init();
		 osapp_set_dev_config(GAP_ROLE_CENTRAL,GAPM_CFG_ADDR_PUBLIC,GAPM_PAIRING_LEGACY,BLE_L2CAP_MAX_MTU);
		 LOG(LOG_LVL_INFO,"osapp_ble_stack_run\n");
	}

**1. 设置central 支持的 profiles**
------------------------------------------------------

**1. 在ble_central.h 中，设置枚举自定义 profile service id**

.. code:: c

	/// BX24XX Central Service Table
	enum
	{
		PRF_BX24XX_SIMPLES_SERVICE_ID,


		BLE_CENTRAL_SERVICES_NUM,
	};


**2. 在ble_central.c 中，设置初始话profiles 信息**

.. code:: c

	static void osapp_ble_stack_data_init(void)
	{
		memset(&central_env, 0 , sizeof(ble_central_env_t));
		central_env.svc.max_num = BLE_CENTRAL_SERVICES_NUM;
		central_env.svc.index = 0;
		central_env.svc.handles[PRF_BX24XX_SIMPLES_SERVICE_ID] = ble_bx24xx_simple_prf_add_svc;
	}

	
**2. 设置设备的 角色设置**
------------------------------------------------------

.. code:: c

	 osapp_set_dev_config(GAP_ROLE_CENTRAL,GAPM_CFG_ADDR_PUBLIC,GAPM_PAIRING_LEGACY,BLE_L2CAP_MAX_MTU);

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
****************************

.. code:: c

	/**
	 * @brief message and handler table. This define the connection of message and it's callback.
	 */
	static const osapp_msg_handler_table_t handler_table[]=
	{
		[0] =   {KE_MSG_DEFAULT_HANDLER,(osapp_msg_handler_t)osapp_default_msg_handler},
		   ///connection indicate: receive connect request from master
		   {GAPC_CONNECTION_REQ_IND,(osapp_msg_handler_t)osapp_gapc_conn_req_ind_handler},
		   ///connection lost indicate handler
		   {GAPC_DISCONNECT_IND,(osapp_msg_handler_t)osapp_gapc_disconnect_ind_handler},
		   ///GAPM event complete
		   {GAPM_CMP_EVT,(osapp_msg_handler_t)osapp_gapm_cmp_evt_handler},
		   ///ble power on ready and should do a reset
		   {GAPM_DEVICE_READY_IND,(osapp_msg_handler_t)osapp_device_ready_ind_handler},
		   ///trigger when master need to read device information uuid 0x1800
		   {GAPC_GET_DEV_INFO_REQ_IND,(osapp_msg_handler_t)osapp_gapc_get_dev_info_req_ind_handler},
		   ///add service complete and store service handler
		   {GATTM_ADD_SVC_RSP,(osapp_msg_handler_t)osapp_gattm_add_svc_rsp_handler},
		   ///master write data to device
		   {GATTC_WRITE_REQ_IND,(osapp_msg_handler_t)osapp_gattc_write_req_ind_handler},
		   ///master read data from device
		   {GATTC_READ_REQ_IND,(osapp_msg_handler_t)osapp_gattc_read_req_ind_handler},
		   ///gattc event has completed
		   {GATTC_CMP_EVT,(osapp_msg_handler_t)osapp_gattc_cmp_evt_handler},
	};

	const osapp_msg_handler_info_t handler_info = ARRAY_INFO(handler_table);

	
_`ble 和peripheral连接通信`
==============================================

central 和 peripheral连接通信分为以下步骤
********************************************************


**1. 设置扫描参数，扫描空中的设备**
------------------------------------------------------

.. code:: c

	static int32_t osapp_start_scan(void)
	{
		struct gapm_start_scan_cmd *cmd = AHI_MSG_ALLOC(GAPM_START_SCAN_CMD,TASK_ID_GAPM,gapm_start_scan_cmd);
		cmd->op.code = GAPM_SCAN_PASSIVE;

		cmd->mode = GAP_GEN_DISCOVERY;
		cmd->interval = 0x20;
		cmd->window = 0x20;

		return osapp_msg_build_send(cmd, sizeof(struct gapm_start_scan_cmd));
	}
	
**2. 发现符合要求的设备后，建立连接**
------------------------------------------------------

.. code:: c
	
	//停止扫描
	static void osapp_find_device(ke_msg_id_t const msgid, void const *param,ke_task_id_t const dest_id,ke_task_id_t const src_id)
	{
		struct adv_report const *report = param;

		bd_addr_t filter_addr={
				.addr = {CENTRAL_FILTER_BDADDR}
		};

		if(memcmp(filter_addr.addr ,report->adv_addr.addr , GAP_BD_ADDR_LEN) == 0)//match
		{
			LOG(LOG_LVL_INFO,"find the device :\n");
			osapp_stop_scan();
		}
	}
    //建立连接	
	static int32_t  osapp_connect(void)   //struct gap_bdaddr *addr, struct gapm_start_connection_cmd *conn_cmd
	{
		struct gap_bdaddr addr;
		struct gapm_start_connection_cmd conn_cmd;
		uint8_t  connect_addr[GAP_BD_ADDR_LEN] = {CENTRAL_FILTER_BDADDR};

		memset(&addr,0,sizeof(addr));
		memset(&conn_cmd,0,sizeof(conn_cmd));

		conn_cmd.op.code = GAPM_CONNECTION_DIRECT;
		conn_cmd.op.addr_src = GAPM_STATIC_ADDR;
		conn_cmd.scan_interval = 0x20;
		conn_cmd.scan_window = 0x20;

		conn_cmd.con_intv_max = 0x6;
		conn_cmd.con_intv_min = 0x6;
		conn_cmd.con_latency = 0;
		conn_cmd.superv_to = 0x2a;

		conn_cmd.ce_len_max = 0x0;
		conn_cmd.ce_len_min = 0x0;

		conn_cmd.nb_peers = 1;

		addr.addr_type = ADDR_PUBLIC;
		memcpy(addr.addr.addr,connect_addr,GAP_BD_ADDR_LEN);

		struct gapm_start_connection_cmd* new_cmd = AHI_MSG_ALLOC_DYN(GAPM_START_CONNECTION_CMD,
																		 TASK_ID_GAPM,
																		 gapm_start_connection_cmd, sizeof(struct gap_bdaddr));

		memcpy(new_cmd,&conn_cmd,sizeof(struct gapm_start_connection_cmd));

		memcpy(new_cmd->peers,&addr,sizeof(struct gap_bdaddr));

		return osapp_msg_build_send(new_cmd, sizeof(struct gapm_start_connection_cmd) + sizeof(struct gap_bdaddr) );
	}

**3. 连接后开始通信**
------------------------------------------------------

.. code:: c

	static void osapp_start_data_communication(void)
	{
		//do something
	}
	