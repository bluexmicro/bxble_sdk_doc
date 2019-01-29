=======================
ble bond manage
=======================

模块简介
=======================
* 该模块主要实现了ble配对绑定的交互信息处理，以及绑定过程需要存储到flash上的数据存储。
* 该模块用的了 osapp_utils 模块的一些API，用于获取设备的 ble地址/地址类型和irk。
* 整个bond manage功能模块涉及到和ble协议栈交互的信息句柄如下：

.. code:: c

	static osapp_msg_handler_table_t const handler_table[]=
	{
		//gapm
				///GAPM event complete
				{GAPM_CMP_EVT,(osapp_msg_handler_t)osapp_gapm_cmp_evt_handler},
				//gapm addr solved ind
				{GAPM_ADDR_SOLVED_IND,(osapp_msg_handler_t)osapp_gapm_addr_solved_ind_handler},
		//gapc
				///This is the generic complete event for GAP operations. All operation triggers this event when operation is finished.
				{GAPC_CMP_EVT,(osapp_msg_handler_t)osapp_gapc_cmp_evt_handler},
				///connection indicate: receive connect request from master
				{GAPC_CONNECTION_REQ_IND,(osapp_msg_handler_t)osapp_gapc_conn_req_ind_handler},
				///connection lost indicate handler
				{GAPC_DISCONNECT_IND,(osapp_msg_handler_t)osapp_gapc_disconnect_ind_handler},
				///Event Triggered on master side when slave request to have a certain level of authentication.
				{GAPC_SECURITY_IND,(osapp_msg_handler_t)osapp_gapc_security_ind_handler},
				///Event Triggered during a bonding procedure in order to get:
					///Slave pairing information
					///Pairing temporary key (TK)
					///Key to provide to peer device during key exchange.
					///This event shall be followed by a GAPC_BOND_CFM message with same request code value.
				{GAPC_BOND_REQ_IND,(osapp_msg_handler_t)osapp_gapc_bond_req_ind_handler},
				///Event triggered when bonding information is available such as:
					///Status of the pairing (succeed or failed)
					///Key exchanged by peer device.
				{GAPC_BOND_IND,(osapp_msg_handler_t)osapp_gapc_bond_ind_handler},
				///Event Triggered during encryption procedure on slave device in order to retrieve LTK according to random
					///number and encryption diversifier value.
					///This event shall be followed by a GAPC_ENCRYPT_CFM message.
				{GAPC_ENCRYPT_REQ_IND,(osapp_msg_handler_t)osapp_gapc_encrypt_req_ind_handler},
				///Event triggered when encryption procedure succeed, it contains the link authentication level provided
					///during connection confirmation (see GAPC_CONNECTION_CFM)
				{GAPC_ENCRYPT_IND,(osapp_msg_handler_t)osapp_gapc_encrypt_ind_handler},

	};

*注意：【实际的消息句柄以发布的协议栈为准】*

文件说明
=======================
* 整个模块相关文件的目录位置：
	sdk/app/freertos/utils/bond/
	
* 用户主要访问的文件

  * bond_manage.h    协议栈消息交互接口（应用收到的事件和应用要处理的动作）
  * bond_save.h      协议栈绑定信息交互接口（应用从flash 读取信息和保存信息到flash）

示例说明
=======================

* 如何使用bond manage   

	* 参考:	 `bond manage 应用`_
	
* bond manage 模块详细介绍  

	* 参考:	 `模块详细介绍`_
	
* bond manage 模块常用参数配置  

	* 参考:	 `常用参数配置`_
	
	
_`bond manage 应用`
=======================

**1. 定义用户自己的 user_init() 函数**

.. code:: c

	void user_init()
	{
		osapp_utils_set_dev_init(GAP_ROLE_PERIPHERAL,GAPM_CFG_ADDR_PUBLIC);//设置设备的角色属性
		osapp_bond_manage_init();//bond manage 初始化
		ahi_handler_register(&handler_info);//注册用户自己的消息处理句柄列表
	}

如上述代码：
	* 先进行设备的角色初始化（osapp_utils 会自动设置设备的地址和irk）
	* bond manage init 进行绑定参数的初始化配置
	* 注册用户自己的消息处理句柄列表
	
**2. 实现bond manage的消息事件回调**

.. code:: c

	static void bond_manage_evt_cb(bond_manage_evt_t evt,bond_manage_evt_param_t *p_param)
	{
		LOG(LOG_LVL_INFO,"bond_manage_evt_cb evt: %d\n",evt);

			switch(evt)
			{
				case BOND_MG_EVT_CONNECTED :
					{
						bond_mg_evt_param_connected_t connected = p_param->connected;

						ntf_src_enable= ANCC_FALSE;
						conn_hdl = connected.conn_idx;
						osapp_ancc_enable_req(connected.conn_idx);

						//process send ancs data src
						if(connected.result == BOND_MG_BONDED)
						{
							LOG(LOG_LVL_INFO,"BOND_MG_EVT_CONNECTED bond id: %d ,conn_idx: %d\n",connected.bond_id,connected.conn_idx);
							process_send_ancs_data_src_procdure(SEND_ANCS_DATA_SRC_RECONNECT_COMPLETE);
						}
					}
					break;
				case BOND_MG_EVT_DISPLAY_PW :
					{
						bond_mg_evt_param_display_pw_t display_pw = p_param->display_pw;

						LOG(LOG_LVL_INFO,"BOND_MG_EVT_DISPLAY_PW key: %06d ,conn_idx: %d\n",display_pw.key,display_pw.conn_idx);
					}
					break;
				case BOND_MG_EVT_NC_PW :
					{
						bond_mg_evt_param_nc_pw_t nc_pw = p_param->nc_pw;
						bond_manage_action_param_t   action;

						LOG(LOG_LVL_INFO,"BOND_MG_EVT_NC_PW key: %06d ,conn_idx: %d\n",nc_pw.key,nc_pw.conn_idx);

						//TODO:judge peer display nc pw is same to local.
						action.nc_right.conn_idx = nc_pw.conn_idx;
						action.nc_right.result   = BOND_MG_ACTION_NC_RIGHT;//BOND_MG_ACTION_NC_WRONG

						BX_ASSERT(bond_manage_action_set(BOND_MG_ACTION_NC_PW,&action) == BLE_BOND_SUCCESS);
					}
					break;
				case BOND_MG_EVT_INPUT_PW :
					{
						bond_mg_evt_param_input_pw_t input_pw = p_param->input_pw;
	//                    bond_manage_action_param_t   action;

						LOG(LOG_LVL_INFO,"BOND_MG_EVT_INPUT_PW conn_idx: %d\n",input_pw.conn_idx);

						//TODO:input peer display password.
	//                    action.input_key.conn_idx = input_pw.conn_idx;
	//                    action.input_key.key = 0;//todo:input key
	//
	//                    BX_ASSERT(bond_manage_action_set(BOND_MG_ACTION_KEY_INPUT_PW,&action) == BLE_BOND_SUCCESS);
					}
					break;
				case BOND_MG_EVT_PAIR_RESULT :
					{
						bond_mg_evt_param_pair_result_t pair_result = p_param->pair_result;

						LOG(LOG_LVL_INFO,"BOND_MG_EVT_PAIR_RESULT conn_idx: %d\n",pair_result.conn_idx);
						LOG(LOG_LVL_INFO,"BOND_MG_EVT_PAIR_RESULT bond_id: %d\n",pair_result.bond_id);

						if(p_param->pair_result.success == BOND_MG_PAIRING_SUCCEED)
						{
							LOG(LOG_LVL_INFO,"BOND_MG_PAIRING_SUCCEED \n");
							osapp_ancc_data_src_enable();
						}
						else
						{
							LOG(LOG_LVL_INFO,"BOND_MG_PAIRING_FAILED reason= %d \n",pair_result.u.reason);

							if(pair_result.u.reason == SMP_ERROR_REM_UNSPECIFIED_REASON)
							{
								//if you send a ecurity request in paired device , iOS7 will occur this error , should ignore.
								LOG(LOG_LVL_INFO,"SMP_ERROR_REM_UNSPECIFIED_REASON.(iOS7 ignore)\n");
							}
							else
							{
								osapp_disconnect(pair_result.conn_idx);
							}
						}
					}
					break;
				case BOND_MG_EVT_ENCRYPY_RESULT :
					{
						bond_mg_evt_param_encrypt_result_t  encrypt_result = p_param->encrypt_result;

						LOG(LOG_LVL_INFO,"BOND_MG_EVT_ENCRYPY_RESULT conn_idx: %d\n",encrypt_result.conn_idx);
						LOG(LOG_LVL_INFO,"BOND_MG_EVT_ENCRYPY_RESULT bond_id: %d\n",encrypt_result.bond_id);
						LOG(LOG_LVL_INFO,"BOND_MG_EVT_ENCRYPY_RESULT auth_level: %d\n",encrypt_result.auth_level);
					}
					break;
				case BOND_MG_EVT_LEGACY_OOB :
					{
						bond_mg_evt_param_legacy_oob_t legacy_oob = p_param->legacy_oob;

						LOG(LOG_LVL_INFO,"BOND_MG_EVT_LEGACY_OOB conn_idx: %d\n",legacy_oob.conn_idx);
						osapp_utils_log_hex_data(legacy_oob.key,GAP_KEY_LEN);
					}
					break;
				default:break;
			}

	}

	
**3. 初始化绑定参数**

.. code:: c

	static void osapp_bond_manage_init(void)
	{
		bond_manage_dev_cfg_t cfg=
		{
			  .evt = bond_manage_evt_cb,
			  .pair_mode = BOND_MG_PAIR_MODE_INITIATE,
			  .pairing_feat = {
				  /// IO capabilities (@see gap_io_cap)
				  .iocap = GAP_IO_CAP_DISPLAY_ONLY,//GAP_IO_CAP_NO_INPUT_NO_OUTPUT,
				  /// OOB information (@see gap_oob)
				  .oob = GAP_OOB_AUTH_DATA_NOT_PRESENT,
				  /// Authentication (@see gap_auth)
				  /// Note in BT 4.1 the Auth Field is extended to include 'Key Notification' and
				  /// and 'Secure Connections'.
				  .auth = GAP_AUTH_REQ_MITM_BOND,
				  /// Encryption key size (7 to 16)
				  .key_size = 16,
				  ///Initiator key distribution (@see gap_kdist)
				  .ikey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
				  ///Responder key distribution (@see gap_kdist)
				  .rkey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
				  /// Device security requirements (minimum security level). (@see gap_sec_req)
				  .sec_req = GAP_SEC1_AUTH_PAIR_ENC,
			  },
		};

		BX_ASSERT(bond_manage_init(&cfg)==BLE_BOND_SUCCESS);

	//    bond_manage_action_param_t   action;
	//
	//    action.static_password = 123456;
	//    BX_ASSERT(bond_manage_action_set(BOND_MG_ACTION_STATIC_PW,&action) == BLE_BOND_SUCCESS);
	}

*如果设备端展示密码，并且需要每次展示相同的密码需要设置 BOND_MG_ACTION_STATIC_PW （默认是随机生成密码）*

	
	
_`模块详细介绍`
=======================

**1. 应用收到的事件类型及其参数说明**

.. code:: c

	/// bond manage event type
	typedef enum
	{
		//flash recover complete,ble connected.
		BOND_MG_EVT_CONNECTED     = 0,
		// display password num
		BOND_MG_EVT_DISPLAY_PW,
		// Numeric Comparison password//dispaly and input
		BOND_MG_EVT_NC_PW,
		// input password num
		BOND_MG_EVT_INPUT_PW,
		//Legacy oob info
		BOND_MG_EVT_LEGACY_OOB,
		//pairing result
		BOND_MG_EVT_PAIR_RESULT,
		//encrypt result
		BOND_MG_EVT_ENCRYPY_RESULT,
	}bond_manage_evt_t;

	typedef union
	{
		bond_mg_evt_param_connected_t       connected;
		bond_mg_evt_param_display_pw_t      display_pw;
		bond_mg_evt_param_nc_pw_t           nc_pw;
		bond_mg_evt_param_input_pw_t        input_pw;
		bond_mg_evt_param_pair_result_t     pair_result;
		bond_mg_evt_param_encrypt_result_t  encrypt_result;
		bond_mg_evt_param_legacy_oob_t      legacy_oob;
	}bond_manage_evt_param_t;
	
	
如上述代码：

* BOND_MG_EVT_CONNECTED -> bond_mg_evt_param_connected_t 

		* 当设备建立连接后，该模块会自动去flash中查询连接的设备是否已经绑定，如果已经绑定，绑定信息的访问句柄是什么。
		* 后续所有的绑定信息都是通过该（bond_node_id_t  bond_id）句柄访问 ；如果没有绑定，则是无效句柄 BOND_MG_INVALID_ID 。
		* 数据结构如下
	
.. code:: c
	
	///bond manage event connected struct
	typedef struct
	{
		///bond evt result
		bool            result;//bond result  
		bond_node_id_t  bond_id;
		uint8_t         conn_idx;//connect index
	}bond_mg_evt_param_connected_t;

	
* BOND_MG_EVT_DISPLAY_PW -> bond_mg_evt_param_display_pw_t

		* 当设备收到配对请求，并且需要密码展示时（参考 iocap 配置），请求用户展示密码。
		* 用户密码是 uint32_t  范围 （0~999999），用户需要转换成6个字符展示（000000-999999）
		* 数据结构如下

.. code:: c
	
	///bond manage event display_pw struct
	typedef struct
	{
		uint32_t        key;//display password
		uint8_t         conn_idx;//connect index
	}bond_mg_evt_param_display_pw_t;

* BOND_MG_EVT_NC_PW -> bond_mg_evt_param_nc_pw_t

		* 当设备收到配对请求，并且需要数字比对时（参考 iocap 配置），请求用户展示数字，并且确认和对方设备是否一致。
		* 用户密码是 uint32_t  范围 （0~999999），用户需要转换成6个字符展示（000000-999999）
		* 用户需要在收到该事件后，通过 bond action 进行确认和对方设备是否一致。
		* 数据结构如下

.. code:: c
	
	///bond manage event  Numeric Comparison password struct
	typedef struct
	{
		uint32_t        key;//Numeric Comparison password
		uint8_t         conn_idx;//connect index
	}bond_mg_evt_param_nc_pw_t;

	case BOND_MG_EVT_NC_PW :
	{
		bond_mg_evt_param_nc_pw_t nc_pw = p_param->nc_pw;
		bond_manage_action_param_t   action;

		LOG(LOG_LVL_INFO,"BOND_MG_EVT_NC_PW key: %06d ,conn_idx: %d\n",nc_pw.key,nc_pw.conn_idx);

		//TODO:judge peer display nc pw is same to local.
		action.nc_right.conn_idx = nc_pw.conn_idx;
		action.nc_right.result   = BOND_MG_ACTION_NC_RIGHT;//BOND_MG_ACTION_NC_WRONG

		BX_ASSERT(bond_manage_action_set(BOND_MG_ACTION_NC_PW,&action) == BLE_BOND_SUCCESS);
	}
	break;

* BOND_MG_EVT_INPUT_PW -> bond_mg_evt_param_input_pw_t

		* 当设备收到配对请求，并且需要输入密码时，请求用户输入密码。
		* 用户密码是 uint32_t  范围 （0~999999），对应6个字符展示（000000-999999），用户需要把字符换算成整数
		* 用户需要在收到该事件后，通过 bond action 进行确认输入的密码。
		* 数据结构如下

.. code:: c
	
	///bond manage event input_pw struct
	typedef struct
	{
		uint8_t         conn_idx;//connect index
	}bond_mg_evt_param_input_pw_t;

	case BOND_MG_EVT_INPUT_PW :
	{
		bond_mg_evt_param_input_pw_t input_pw = p_param->input_pw;
		//bond_manage_action_param_t   action;

		LOG(LOG_LVL_INFO,"BOND_MG_EVT_INPUT_PW conn_idx: %d\n",input_pw.conn_idx);

		//TODO:input peer display password.
		//action.input_key.conn_idx = input_pw.conn_idx;
		//action.input_key.key = 0;//todo:input key
		//
		//BX_ASSERT(bond_manage_action_set(BOND_MG_ACTION_KEY_INPUT_PW,&action) == BLE_BOND_SUCCESS);
	}
	break;
	
* BOND_MG_EVT_PAIR_RESULT -> bond_mg_evt_param_pair_result_t

		* 当设备配对结束后，通知用户配对结果。
		* 配对成功，通知用户 配对的安全配置；失败，通知用户失败的原因。
		* 后续所有的绑定信息都是通过该（bond_node_id_t  bond_id）句柄访问 ；如果没有绑定，则是无效句柄 BOND_MG_INVALID_ID 。
		* 数据结构如下

.. code:: c
	
	///bond manage event pair result struct
	typedef struct
	{
		uint8_t            success;
		union{
			uint8_t auth;
			uint8_t reason;
		}u;
		bond_node_id_t  bond_id;
		uint8_t         conn_idx;//connect index
	}bond_mg_evt_param_pair_result_t;

	case BOND_MG_EVT_PAIR_RESULT :
	{
		bond_mg_evt_param_pair_result_t pair_result = p_param->pair_result;

		LOG(LOG_LVL_INFO,"BOND_MG_EVT_PAIR_RESULT conn_idx: %d\n",pair_result.conn_idx);
		LOG(LOG_LVL_INFO,"BOND_MG_EVT_PAIR_RESULT bond_id: %d\n",pair_result.bond_id);

		if(p_param->pair_result.success == BOND_MG_PAIRING_SUCCEED)
		{
			LOG(LOG_LVL_INFO,"BOND_MG_PAIRING_SUCCEED \n");
			osapp_ancc_data_src_enable();
		}
		else
		{
			LOG(LOG_LVL_INFO,"BOND_MG_PAIRING_FAILED reason= %d \n",pair_result.u.reason);

			if(pair_result.u.reason == SMP_ERROR_REM_UNSPECIFIED_REASON)
			{
				//if you send a ecurity request in paired device , iOS7 will occur this error , should ignore.
				LOG(LOG_LVL_INFO,"SMP_ERROR_REM_UNSPECIFIED_REASON.(iOS7 ignore)\n");
			}
			else
			{
				osapp_disconnect(pair_result.conn_idx);
			}
		}
	}
	break;
	
* BOND_MG_EVT_ENCRYPY_RESULT -> bond_mg_evt_param_encrypt_result_t

		* 当设备已经绑定过，重新连接后，通知用户重新连接加密结果。
		* 加密连接的安全配置。
		* 后续所有的绑定信息都是通过该（bond_node_id_t  bond_id）句柄访问 ；如果没有绑定，则是无效句柄 BOND_MG_INVALID_ID 。
		* 数据结构如下

.. code:: c
	
	///bond manage event encrypt result struct
	typedef struct
	{
		uint8_t         auth_level;///// Authentication Requirements @ref enum gap_auth
		bond_node_id_t  bond_id;
		uint8_t         conn_idx;//connect index
	}bond_mg_evt_param_encrypt_result_t;

	case BOND_MG_EVT_ENCRYPY_RESULT :
	{
		bond_mg_evt_param_encrypt_result_t  encrypt_result = p_param->encrypt_result;

		LOG(LOG_LVL_INFO,"BOND_MG_EVT_ENCRYPY_RESULT conn_idx: %d\n",encrypt_result.conn_idx);
		LOG(LOG_LVL_INFO,"BOND_MG_EVT_ENCRYPY_RESULT bond_id: %d\n",encrypt_result.bond_id);
		LOG(LOG_LVL_INFO,"BOND_MG_EVT_ENCRYPY_RESULT auth_level: %d\n",encrypt_result.auth_level);
	}
	break;
	
* ... （TODO）



**2. bond manage 配置参数说明**

.. code:: c
	
	/* @brief  bond manage config data typedef */
	///bond manage init device config data struct
	typedef struct
	{
		///Callback function to send the event of bond manage
		bond_manage_evt_cb_t        evt;
		///bond manage pair mode config
		bond_manage_pair_mode_t     pair_mode;
		/// Pairing Features (request = GAPC_PAIRING_RSP)
		struct gapc_pairing         pairing_feat;
	}bond_manage_dev_cfg_t;
	
如上述代码：
	* 首先需要设置用户事件接收函数
		* 在这个事件接收函数里，用户接收 模块发送的事件消息，并做出必要回复
	* 设置配对的模式
		* BOND_MG_PAIR_MODE_NO_PAIR 设备不支持配对，收到配对请求后，模块自动回复拒绝。
		*  BOND_MG_PAIR_MODE_WAIT_FOR_REQ 设备建立连接后，不会主动发起配对请求，等待对方发起配对请求，或者用户自己发送配对请求。
		* BOND_MG_PAIR_MODE_INITIATE 设备建立连接后，自动发起配对请求。

.. code:: c

	/// bond manage pair mode config
	typedef enum
	{
		/// Pairing is not allowed.
		BOND_MG_PAIR_MODE_NO_PAIR = 0x00,
		/// Wait for a pairing request or slave security request,not auto send.
		BOND_MG_PAIR_MODE_WAIT_FOR_REQ = 0x01,
		/// When the link connected, don't wait, the bond manage auto initiate a pairing request or slave security request.
		BOND_MG_PAIR_MODE_INITIATE = 0x02,
	}bond_manage_pair_mode_t;
	
* 设置配对的特性
		
.. code:: c
	
	/// Pairing parameters
	struct gapc_pairing
	{
		/// IO capabilities (@see gap_io_cap)
		uint8_t iocap;
		/// OOB information (@see gap_oob)
		uint8_t oob;
		/// Authentication (@see gap_auth)
		/// Note in BT 4.1 the Auth Field is extended to include 'Key Notification' and
		/// and 'Secure Connections'.
		uint8_t auth;
		/// Encryption key size (7 to 16)
		uint8_t key_size;
		///Initiator key distribution (@see gap_kdist)
		uint8_t ikey_dist;
		///Responder key distribution (@see gap_kdist)
		uint8_t rkey_dist;

		/// Device security requirements (minimum security level). (@see gap_sec_req)
		uint8_t sec_req;
	};

	
_`常用参数配置`
=======================

* Just Works (LE Legacy pairing)
		
.. code:: c
	
    bond_manage_dev_cfg_t cfg=
    {
          .evt = bond_manage_evt_cb,
          .pair_mode = BOND_MG_PAIR_MODE_INITIATE,
          .pairing_feat = {
              /// IO capabilities (@see gap_io_cap)
              .iocap = GAP_IO_CAP_NO_INPUT_NO_OUTPUT,
              /// OOB information (@see gap_oob)
              .oob = GAP_OOB_AUTH_DATA_NOT_PRESENT,
              /// Authentication (@see gap_auth)
              /// Note in BT 4.1 the Auth Field is extended to include 'Key Notification' and
              /// and 'Secure Connections'.
              .auth = GAP_AUTH_REQ_NO_MITM_BOND,
              /// Encryption key size (7 to 16)
              .key_size = 16,
              ///Initiator key distribution (@see gap_kdist)
              .ikey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
              ///Responder key distribution (@see gap_kdist)
              .rkey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
              /// Device security requirements (minimum security level). (@see gap_sec_req)
              .sec_req = GAP_SEC1_NOAUTH_PAIR_ENC,
          },
    };

* Just Works (LE Secure connections pairing)
		
.. code:: c
	
    bond_manage_dev_cfg_t cfg=
    {
          .evt = bond_manage_evt_cb,
          .pair_mode = BOND_MG_PAIR_MODE_INITIATE,
          .pairing_feat = {
              /// IO capabilities (@see gap_io_cap)
              .iocap = GAP_IO_CAP_NO_INPUT_NO_OUTPUT,
              /// OOB information (@see gap_oob)
              .oob = GAP_OOB_AUTH_DATA_NOT_PRESENT,
              /// Authentication (@see gap_auth)
              /// Note in BT 4.1 the Auth Field is extended to include 'Key Notification' and
              /// and 'Secure Connections'.
              .auth = GAP_AUTH_REQ_NO_MITM_BOND,
              /// Encryption key size (7 to 16)
              .key_size = 16,
              ///Initiator key distribution (@see gap_kdist)
              .ikey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
              ///Responder key distribution (@see gap_kdist)
              .rkey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
              /// Device security requirements (minimum security level). (@see gap_sec_req)
              .sec_req = GAP_SEC1_SEC_CON_PAIR_ENC,
          },
    };
	
* Numeric Comparison (LE Secure connections pairing)
		
.. code:: c
	
    bond_manage_dev_cfg_t cfg=
    {
          .evt = bond_manage_evt_cb,
          .pair_mode = BOND_MG_PAIR_MODE_INITIATE,
          .pairing_feat = {
              /// IO capabilities (@see gap_io_cap)
              .iocap = GAP_IO_CAP_DISPLAY_YES_NO,
              /// OOB information (@see gap_oob)
              .oob = GAP_OOB_AUTH_DATA_NOT_PRESENT,
              /// Authentication (@see gap_auth)
              /// Note in BT 4.1 the Auth Field is extended to include 'Key Notification' and
              /// and 'Secure Connections'.
              .auth = GAP_AUTH_REQ_SEC_CON_BOND,
              /// Encryption key size (7 to 16)
              .key_size = 16,
              ///Initiator key distribution (@see gap_kdist)
              .ikey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
              ///Responder key distribution (@see gap_kdist)
              .rkey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
              /// Device security requirements (minimum security level). (@see gap_sec_req)
              .sec_req = GAP_SEC1_SEC_CON_PAIR_ENC,
          },
    };

* Passkey Entry	(LE legacy pairing)

		
.. code:: c
	
    bond_manage_dev_cfg_t cfg=
    {
          .evt = bond_manage_evt_cb,
          .pair_mode = BOND_MG_PAIR_MODE_INITIATE,
          .pairing_feat = {
              /// IO capabilities (@see gap_io_cap)
              .iocap = GAP_IO_CAP_KB_ONLY,
              /// OOB information (@see gap_oob)
              .oob = GAP_OOB_AUTH_DATA_NOT_PRESENT,
              /// Authentication (@see gap_auth)
              /// Note in BT 4.1 the Auth Field is extended to include 'Key Notification' and
              /// and 'Secure Connections'.
              .auth = GAP_AUTH_REQ_MITM_BOND,
              /// Encryption key size (7 to 16)
              .key_size = 16,
              ///Initiator key distribution (@see gap_kdist)
              .ikey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
              ///Responder key distribution (@see gap_kdist)
              .rkey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
              /// Device security requirements (minimum security level). (@see gap_sec_req)
              .sec_req = GAP_SEC1_AUTH_PAIR_ENC,
          },
    };
	
* Passkey Entry	(LE Secure connections pairing)
		
.. code:: c
	
    bond_manage_dev_cfg_t cfg=
    {
          .evt = bond_manage_evt_cb,
          .pair_mode = BOND_MG_PAIR_MODE_INITIATE,
          .pairing_feat = {
              /// IO capabilities (@see gap_io_cap)
              .iocap = GAP_IO_CAP_KB_ONLY,
              /// OOB information (@see gap_oob)
              .oob = GAP_OOB_AUTH_DATA_NOT_PRESENT,
              /// Authentication (@see gap_auth)
              /// Note in BT 4.1 the Auth Field is extended to include 'Key Notification' and
              /// and 'Secure Connections'.
              .auth = GAP_AUTH_REQ_SEC_CON_BOND,
              /// Encryption key size (7 to 16)
              .key_size = 16,
              ///Initiator key distribution (@see gap_kdist)
              .ikey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
              ///Responder key distribution (@see gap_kdist)
              .rkey_dist = GAP_KDIST_ENCKEY | GAP_KDIST_IDKEY ,
              /// Device security requirements (minimum security level). (@see gap_sec_req)
              .sec_req = GAP_SEC1_SEC_CON_PAIR_ENC,
          },
    };
