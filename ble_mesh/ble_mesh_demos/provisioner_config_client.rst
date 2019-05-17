==============================================
provisioner config client demo
==============================================


示例说明
==============================================
* 示例功能简介

	* 参考:	 `示例功能简介`_

* 如何运行示例代码  

	* 参考:	 `示例运行概要`_

* 如何设置 ble mesh 角色  

	* 参考:	 `ble mesh 角色设置`_

* 如何处理 ble mesh 协议栈和应用协议栈的信息交互  

	* 参考:	 `ble mesh 协议栈和应用协议栈的信息交互`_


_`示例功能简介`
==================

该示例主要体现的功能点如下：
********************************


* 设备支持PB-ADV client,且可以支持多server同时入网。


* 满足SIG 对 Provision client的 要求。


* 可以直接入网 generic onoff server 的例程。




_`示例运行概要`
===================

硬件环境
********************************
该示例运行在 BLE Dongle 开发板上（开发板的详细信息，参考硬件支持包），用到硬件外设如下：
_______________________________________________________________________________________________

* 按键

  botton 3  button 4 同时按下 ，心跳灯会快闪，设备重启并重新初始化。
  

* 指示灯

  * led1 : 
  	 * 绿灯   
                * 熄灭， 保留；
                * 熄灭， 保留；
  	 * 蓝灯   
                * 熄灭， 保留；
                * 常亮， 保留；
	 * 红灯  
                * 熄灭， 保留；
                * 常亮， 保留；
  * led2 : 
  	 * 绿灯   
                * 闪烁， 设备正常工作；
                * 常亮/长灭， 设备异常；
  	 * 蓝灯   
                * 熄灭， 保留；
                * 熄灭， 保留；
	 * 红灯  
                * 熄灭， 保留；
                * 常亮， 保留；

软件环境
********************************
* 设备端运行 ble mesh sdk 的 examples 目录下 provisioner_config_client示例。
* 同时需要另外一个设备运行 unprovsion device 程序，请注意该设备的UUID，以及未入网设备支持的相关OOB能力。
  本例程对应的角色为provisioner，通过业务栈自动发现设备，并通知到 相关的接口函数，用户只要实现接口就可以入网控制设备。

软件运行流程
********************************

**1. 用户自己函数入口**

   在 mesh_user_main.c 中， mesh_user_main_init()初始化自己数据。（需要注意：不能阻塞）
   
**2. 开启mesh协议栈调度**

   在用户函数执行完后，系统自动开启ble协议栈调度。

**3. 示例代码**

.. code:: c

	void mesh_user_main_init(void)
	{
		///user data init
	    provisioner_config_client_init();

		LOG(LOG_LVL_INFO,"mesh_user_main_init\n");
	}
    
.. code:: c
    /** Init common server/client model*/
    //init a client model
   #define INIT_CLIENT_MODEL(model_name , model_id , sig_model)                     \
        mesh_model_init(&model_name.model.base, model_id, sig_model,                        \
                APPKEY_BOUND_NETKEY_MAX_NUM,model_name##_bound_key_buf);                    \
        model_publish_subscribe_bind(&model_name.model.base , &model_name##_publish_state,  \
                model_name##_subscription_list, ARRAY_LEN(model_name##_subscription_list), NULL); 

例程初始状态
********************************
设备正常上电后： 
  * led1 : 
  	 * 绿灯   
                * 熄灭， 保留；
  	 * 蓝灯   
                * 熄灭， 保留；
	 * 红灯  
                * 熄灭， 保留；
  * led2 : 
  	 * 绿灯   
                * 闪烁， 设备正常工作；
  	 * 蓝灯   
                * 熄灭， 保留；
	 * 红灯  
                * 熄灭， 保留；



_`ble mesh 角色设置`
===================================================================================================================

设置流程
********************************

.. code:: c

	static void user_role_init(void)
	{
	    //1.role init
	    provision_init(MESH_ROLE_UNPROV_DEVICE, mesh_provisioner_evt_cb);
	    //2. data init
	    provisioner_data_init();
        //3. set client own uniaddr
        init_elmt_addr(USER_CLENT_UNICAST_ADDRESS);
	}


**1. 定义协议栈内部事件通知回调函数**

.. code:: c

	/* provision device event callback function */
    static void mesh_provisioner_evt_cb(mesh_prov_evt_type_t type , mesh_prov_evt_param_t param)
    {
    LOG(LOG_LVL_INFO,"mesh_provisioner_evt_cb type : %d\n",type);
    switch(type)
    {
        case  PROV_EVT_BEACON :
        {
            //action link open : 这个是协议栈收到unprov beacon通知用户，用户可以判断uuid是否为需要入网的设备，下面的判断是简单的例程
            if(!memcmp(param.prov.param.p_beacon->dev_uuid + GAP_BD_ADDR_LEN,beacon_value +GAP_BD_ADDR_LEN,MESH_DEVICE_UUID_LENGTH-GAP_BD_ADDR_LEN))
            {
                start_provision_dev(param.prov.param.p_beacon->dev_uuid);
            }
            else
            {
                  LOG(LOG_LVL_INFO,"invalid device");
                  log_hex_data(param.prov.param.p_beacon->dev_uuid, MESH_DEVICE_UUID_LENGTH);
            }
        }
        break;
        // 收到LINK ACK的通知，可以不处理
        case  PROV_EVT_LINK_ACK ://(NO ACTION)
        {
             
        }
        break;
        // 收到capabilities的通知，provisioner可以根据这个记录用户的相关能力，选择合适的
        case  PROV_EVT_CAPABILITIES :
        {
        
        }
        break;
        // 这个需要provisioner 根据在PROV_EVT_CAPABILITIES 事件收到的 capabilities与自身能力来构建start报文
        case PROV_EVT_REQUEST_START:
        {    
              admin_provisioner_set_start_pdu(param);            
        }
        break;
        // 通知输入对方的public key 
        case  PROV_EVT_READ_PEER_PUBLIC_KEY_OOB : //alert input dialog
        {
            //action read peer public key
            read_public_key(param);
        }
        break;
        // 这个事件是请求 provisioner output auth info，以方便再 unprov device输入
        case  PROV_EVT_AUTH_DISPLAY_NUMBER : //provisioner expose random number (NO ACTION)
        {
             LOG(LOG_LVL_INFO,"ooutput auth = ");
             log_hex_data(param.prov.param.p_output_val, AUTHVALUE_LEN);
             make_light_blink(param.prov.param.p_output_val[AUTHVALUE_LEN -1]);

        }
        break;
        // 这个事件是请求 provisioner 输入 auth info
        case  PROV_EVT_AUTH_INPUT_NUMBER : //alert input dialog
        {
             LOG(LOG_LVL_INFO,"input auth = \n");
             make_user_attention();
             memcpy(dev_uuid,param.prov.dev_uuid,AUTHVALUE_LEN);
        }
        break;
        // 通知入网完成
        case  PROV_EVT_PROVISION_DONE :  //(NO ACTION)
        {
            
        }
        break;
        default:break;
    }
}




**2. 设置角色，注册事件回调**

.. code:: c

	provision_init(MESH_ROLE_UNPROV_DEVICE, mesh_provisioner_evt_cb);

	
**3. 初始化角色相关的数据**

.. code:: c

   static void provisioner_data_init(void)
   {
    volatile mesh_prov_evt_param_t evt_param;

    uint8_t  bd_addr[GAP_BD_ADDR_LEN];

    //get bd_addr
    mesh_core_params_t core_param;
    core_param.mac_address = bd_addr;
    mesh_core_params_get(MESH_CORE_PARAM_MAC_ADDRESS,&core_param);

    //1. Method of configuring network access
    evt_param.prov.param.method = PROVISION_BY_ADV;
    provision_config(PROV_SET_PROVISION_METHOD,evt_param);
    //2. distribution data
    //send message will use the first netkey.
    //See @access_tx_pdu_set -> @get_netkey_by_dst_addr -> @dm_netkey_get_first_handle
      evt_param.prov.param.p_distribution = &m_prov_user.distribution;
      provision_config(PROV_SET_DISTRIBUTION_DATA,evt_param);

      
    //4. PROV_SET_INVITE_DURATION
    evt_param.prov.param.attention_duration = USER_ATTENTION_DURATION;
    provision_config(PROV_SET_INVITE_DURATION,evt_param);
    //5. private key
    memcpy(m_prov_user.prov_private_key,bd_addr,GAP_BD_ADDR_LEN);
    evt_param.prov.param.p_prov_private_key = m_prov_user.prov_private_key;
    provision_config(PROV_SET_PRIVATE_KEY,evt_param);
}

**4. 协议栈开始完整运行**

监听协议栈事件。。。。


_`ble mesh 协议栈和应用协议栈的信息交互`
==============================================

实现消息交互的处理函数
********************************

  相关的API函数，可以参考	\app\freertos\mesh\provision\api\provision_api.c



根据收到的事件，做相应处理或回复
********************************

.. code:: c

	//协议->用户
    typedef enum
    {
    /*******PROVISIONER*******/
    PROV_EVT_BEACON,
    PROV_EVT_LINK_ACK,    //(NO ACTION)
    PROV_EVT_CAPABILITIES,
    PROV_EVT_REQUEST_START,
    PROV_EVT_READ_PEER_PUBLIC_KEY_OOB,
    PROV_EVT_AUTH_DISPLAY_NUMBER,//provisioner expose random number (NO ACTION)
    PROV_EVT_AUTH_INPUT_NUMBER,   //alert input dialog
    PROV_EVT_PROVISION_DONE,    //(NO ACTION)

    /*******UNPROV DEVICE*******/
    UNPROV_EVT_INVITE_MAKE_ATTENTION,//(NO ACTION)
    UNPROV_EVT_EXPOSE_PUBLIC_KEY, //(NO ACTION)
    UNPROV_EVT_AUTH_INPUT_NUMBER,//alert input dialog
    UNPROV_EVT_AUTH_DISPLAY_NUMBER,//unprov_device expose random number //(NO ACTION)
    UNPROV_EVT_PROVISION_DONE, //(NO ACTION)
    } mesh_prov_evt_type_t;

	//用户->协议栈（回复）
	typedef enum
	{
	    /*******PROVISIONER*******/
	    //PROV_EVT_AUTH_INPUT_NUMBER
	    PROV_ACTION_AUTH_INPUT_NUMBER_DONE,//input random number done
	    //PROV_EVT_READ_PEER_PUBLIC_KEY_OOB
	    PROV_ACTION_READ_PEER_PUBLIC_KEY_OOB_DONE,
	    //PROV_EVT_BEACON
	    PROV_ACTION_SET_LINK_OPEN,
	    //PROV_EVT_CAPABILITIES
	    PROV_ACTION_SEND_START_PDU,

	    /*******UNPROV DEVICE*******/
	    //UNPROV_EVT_AUTH_INPUT_NUMBER
	    UNPROV_ACTION_AUTH_INPUT_NUMBER_DONE,//input random number done
	} mesh_prov_action_type_t;

	void provision_action_send (mesh_prov_action_type_t type , mesh_prov_evt_param_t param);

