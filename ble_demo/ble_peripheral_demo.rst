=======================
ble peripheral demo
=======================


示例说明
=======================

* 如何新建自定义 service    

	* 参考:	 `ble 自定义 service`_

* 如何设置 ble peripheral 角色  

	* 参考:	 `ble 角色设置`_

* 如何处理 ble 协议栈和应用协议栈的信息交互  

	* 参考:	 `ble 协议栈和应用协议栈的信息交互`_


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

_`ble 自定义 service`
=======================

参考ble_bx2400 simple service 创建模板
********************************************************

**1. 定义自己的service 描述**

.. code:: c

	static const struct gattm_svc_desc bx24xx_simple_svc_desc = {
			.start_hdl = 0,
			.task_id = TASK_ID_AHI,
			.perm = PERM(SVC_MI,DISABLE)|PERM(SVC_EKS,DISABLE)|\
				PERM(SVC_AUTH,NO_AUTH)|PERM(SVC_UUID_LEN,UUID_128),PERM_VAL(SVC_SECONDARY,0),
			.nb_att = BX24XX_SIMPLES_ATT_NUM,
			.uuid = BX24XX_SIMPLE_SVC_UUID_128,
	};

**2. 定义自己的att 属性列表**

.. code:: c

	static const struct gattm_att_desc bx24xx_simple_svc_att_db[] =
	{
		// Characteristic1 Declaration
		[BX24XX_SIMPLES_IDX_CHAR1_CHAR]        =   {
				.uuid = ATT_UUID16_TO_ARRAY(ATT_DECL_CHARACTERISTIC),
				.perm = PERM(RD, ENABLE),
				.max_len =0,
				.ext_perm = PERM(UUID_LEN,UUID_16),
		},
		// Characteristic1 Value
		[BX24XX_SIMPLES_IDX_CHAR1_VAL]         =   {
				.uuid = BX24XX_SIMPLE_CHAR1_UUID_128,
				.perm = PERM(WRITE_REQ,ENABLE)|PERM(WRITE_COMMAND,ENABLE)|PERM(WP,NO_AUTH)|PERM(RD, ENABLE)|PERM(RP,NO_AUTH),//write read
				.max_len = BX24XX_SIMPLE_CHAR1_MAX_LEN,
				.ext_perm = PERM(RI,ENABLE)|PERM(UUID_LEN, UUID_128),// if the char's perm has 'read',this must be set.  uuid 128
		},
		// Characteristic2 Declaration
		[BX24XX_SIMPLES_IDX_CHAR2_CHAR]        =   {
				.uuid = ATT_UUID16_TO_ARRAY(ATT_DECL_CHARACTERISTIC),
				.perm = PERM(RD, ENABLE),
				.max_len =0,
				.ext_perm = PERM(UUID_LEN,UUID_16),
		},
		// Characteristic2 Value
		[BX24XX_SIMPLES_IDX_CHAR2_VAL]         =   {
				.uuid = BX24XX_SIMPLE_CHAR2_UUID_128,
				.perm = PERM(RD, ENABLE)|PERM(RP,NO_AUTH),//read
				.max_len = BX24XX_SIMPLE_CHAR2_MAX_LEN,
				.ext_perm = PERM(RI,ENABLE)|PERM(UUID_LEN, UUID_128),// if the char's perm has 'read',this must be set.  uuid 128
		},
		// Characteristic3 Declaration
		[BX24XX_SIMPLES_IDX_CHAR3_CHAR]        =   {
				.uuid = ATT_UUID16_TO_ARRAY(ATT_DECL_CHARACTERISTIC),
				.perm = PERM(RD, ENABLE),
				.max_len =0,
				.ext_perm = PERM(UUID_LEN,UUID_16),
		},
		// Characteristic3 Value
		[BX24XX_SIMPLES_IDX_CHAR3_VAL]         =   {
				.uuid = BX24XX_SIMPLE_CHAR3_UUID_128,
				.perm = PERM(NTF, ENABLE)|PERM(NP,NO_AUTH),//notify
				.max_len = BX24XX_SIMPLE_CHAR3_MAX_LEN,
				.ext_perm = PERM(UUID_LEN, UUID_128),//uuid 128
		},
		// Client Characteristic Configuration Descriptor
		[BX24XX_SIMPLES_IDX_CHAR3_CFG]      =   {
				.uuid = ATT_UUID16_TO_ARRAY(ATT_DESC_CLIENT_CHAR_CFG),
				.perm = PERM(RD, ENABLE) |PERM(WRITE_REQ, ENABLE),
				.max_len =0,
				.ext_perm = PERM(UUID_LEN,UUID_16),
		},
	};

**3. 实现service的函数事件回调**

.. code:: c

	static const gattServiceCBs_t bx24xx_simple_callbacks = {
			.pfnReadAttrCB = ble_bx24xx_simple_read_callback,
			.pfnWriteAttrCB = ble_bx24xx_simple_write_callback,
			.pfnConnectCB = ble_bx24xx_simple_connect_callback,
			.pfnHandlerInitCB = ble_bx24xx_simple_handler_init,
	};

**4. 实现外部应用服务添加接口**

.. code:: c

	int32_t ble_bx24xx_simple_add_svc(gattServiceCBs_t *cb)
	{
		struct gattm_add_svc_req *req = AHI_MSG_ALLOC_DYN(GATTM_ADD_SVC_REQ,TASK_ID_GATTM,\
			gattm_add_svc_req,sizeof(bx24xx_simple_svc_att_db));
		struct gattm_svc_desc *svc = &req->svc_desc;
		memcpy(svc,&bx24xx_simple_svc_desc,sizeof(bx24xx_simple_svc_desc));
		memcpy(svc->atts,bx24xx_simple_svc_att_db,sizeof(bx24xx_simple_svc_att_db));

		memcpy(cb,&bx24xx_simple_callbacks,sizeof(gattServiceCBs_t));

		LOG(LOG_LVL_INFO," svc ble_bx24xx_simple_add_svc \n");
		return osapp_msg_build_send(req, sizeof(struct gattm_svc_desc)+sizeof(bx24xx_simple_svc_att_db));
	}

**5. 书写service应用的外部数据访问接口**

.. code:: c

	void ble_bx24xx_simple_char3_send_notification(uint8_t const *data,uint8_t length)
	{
		static uint16_t notify_seq_num = 0;
		struct gattc_send_evt_cmd *cmd= AHI_MSG_ALLOC_DYN(GATTC_SEND_EVT_CMD,TASK_ID_GATTC, gattc_send_evt_cmd, length);

		if(bx24xx_simple_env.is_connect == IS_CONNECTED && char3_notify_cfg)
		{
			LOG(LOG_LVL_INFO,"ble_bx24xx_simple_char3_send_notification\n");

			cmd->operation = GATTC_NOTIFY;
			cmd->seq_num = notify_seq_num++;
			cmd->handle = start_handler + BX24XX_SIMPLES_IDX_CHAR3_CFG;
			cmd->length = length;
			memcpy(cmd->value,data,length);

			osapp_ahi_msg_send(cmd, (sizeof(struct gattc_send_evt_cmd) + length), portMAX_DELAY);
		}
	}


_`ble 角色设置`
=======================

设置流程
****************************

.. code:: c

	void osapp_ble_stack_run(void)
	{
		 osapp_ble_stack_data_init();
		 osapp_set_dev_config(GAP_ROLE_PERIPHERAL,GAPM_CFG_ADDR_PUBLIC,GAPM_PAIRING_LEGACY,BLE_PERIPHERAL_MAX_MTU);
		 LOG(LOG_LVL_INFO,"osapp_ble_stack_run\n");
	}

**1. 设置perapheral 支持的自定义服务**
------------------------------------------------------

**1. 在ble_peripheral_simple.h 中，设置枚举自定义service id**

.. code:: c

	/// BX24XX Peripheral Service Table
	enum
	{
		BX24XX_SIMPLES_SERVICE_ID,


		BLE_PERIPHERAL_SERVICES_NUM,
	};

**2. 在ble_peripheral_simple.c 中，设置初始话service 信息**

.. code:: c

	static void osapp_ble_stack_data_init(void)
	{
		memset(&peripheral_env, 0 , sizeof(ble_peripheral_simple_env_t));
		peripheral_env.svc.max_num = BLE_PERIPHERAL_SERVICES_NUM;
		peripheral_env.svc.index = 0;
		peripheral_env.svc.handles[BX24XX_SIMPLES_SERVICE_ID] = ble_bx24xx_simple_add_svc;
	}
	
**2. 设置设备的 角色设置**
------------------------------------------------------

.. code:: c

	osapp_set_dev_config(GAP_ROLE_PERIPHERAL,GAPM_CFG_ADDR_PUBLIC,GAPM_PAIRING_LEGACY,BLE_PERIPHERAL_MAX_MTU);

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
