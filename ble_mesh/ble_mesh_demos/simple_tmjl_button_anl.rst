==============================================
simple tmjl button anl demo
==============================================


示例说明
==============================================
* 示例功能简介

    * 参考:     `示例功能简介`_

* 如何运行示例代码

    * 参考:     `示例运行概要`_

* 如何设置 ble mesh 角色

    * 参考:     `ble mesh 角色设置`_

* 如何处理 ble mesh 协议栈和应用协议栈的信息交互

    * 参考:     `ble mesh 协议栈和应用协议栈的信息交互`_


_`示例功能简介`
==================
本示例功能主要实现 SIG 标准的 light_ctl_client, light_lightness_client 和generic_onoff_client 和天猫精灵的开关板模型。
可以用于灯等设备模型。本示例实例化了一个element，每个element包括一个 light ctl client, 
generic onoff client 和 vendor server 模型。初始化部分可以参考examples 目录下 node_setup.c 文件里面的
mesh_app_init_user函数说明，开发者可以非常容易添加更多 client 的 model。另外，为了对系统进行控制，
并且可以控制别的灯，需要给开关板的每个 client model 配置一个 publish 地址。现在目前只是统一的往 0xC000 地址发。该model 实现了开关板的
低功耗的功能，可以调节灯的亮度，色温，开关，并且亮度和色温支持无极调光。

该示例主要体现的功能点如下：
********************************

* 设备支持 advertising，经 adv 通过天猫精灵连接快速配置入网。

* 设备支持mesh proxy，可以通过手机，经gatt连接快速配置入网。

* 节点支持分组，可以分组控制。

_`示例运行概要`
===================

硬件环境
********************************
该示例运行在 BLE 开关板开发板上（开发板的详细信息，参考硬件支持包），用到硬件外设如下：
_______________________________________________________________________________________________

* 恢复出场设置

  长按开关板的多功能键，大概三到四秒直至发现指示灯会亮灭的闪，说明设备重启并重新初始化，该操作会丢弃所有之前配置，
  设备变成unprovision 状态，而且只有在灯一直闪烁的时候灯才会发 unprovision 的包，才可以被入网，并且被入网成功之后
  指示灯才不会闪烁进入熄灭状态。

* 指示灯

  * 默认是不亮，为了省电，但是长按多功能键进入到可被入网状态之后灯会一直闪烁，直至入网成功。

软件环境
********************************

* 设备端运行 ble mesh sdk 的 examples 目录下 simple_tmjl_button_anl示例。
* 用手机端或者天猫精灵入网调试，支持任意符合 mesh 标准的命令。

软件运行流程
********************************
**1. 用天猫精灵入网时需要注意的地方**
    * 在编译出来的 hex 的 0x80200 位置填入设备的 MAC 地址，该地址是需要申请的，如下所示：
        0xb9,0x9f,0x31,0xca,0xd2,0x38
    * 在编译出来的 hex 的 0x802010 位置填上 USER UNPROV STATIC AUTH VAL, 该值必须要是用申请的三元组通过hash 256 生成前 16 个字节，如下所示: 
        0xa5,0x95,0xaf,0x15,0x7f,0x3c,0xe2,0x48,0x2b,0xfb,0x25,0xe1,0x82,0x1c,0x1b,0x4f
    * 在编译出来的 hex 的 0x802020 位置上填上 USER UNPROV BEACON UUID 宏前面字节必须是注册的 company ID, Product ID 和当前设备的 MAC 地址组成,具体组成如下所示：
        0xa8,0x01,\
        0x51,\
        0x17,0x0f,0x00,0x00,\
        0xb9,0x9f,0x31,0xca,0xd2,0x38,\
        0x00,0x00,0x00  

**2. 用户自己函数入口**

   在 mesh_user_main.c 中， mesh_user_main_init()初始化自己数据。（需要注意：不能阻塞）

**3. 开启 mesh 协议栈调度**

   在用户函数执行完后，系统自动开启ble协议栈调度。

**4. 示例代码**

.. code:: c

    void mesh_user_main_init(void)
    {
        ///user data init
        simple_tmjl_button_anl_init();

        LOG(LOG_LVL_INFO,"mesh_user_main_init\n");
    }

例程初始状态
********************************
设备正常上电后：
  * 入网前:
       * 非清网重启: 上电一瞬间指示灯会闪烁一下;
       * 清网重启: 指示灯会一直闪烁，直至被入网;
  * 入网后 :
       * 上电一瞬间指示灯会闪烁一下;


_`ble mesh 角色设置`
===================================================================================================================

.. code:: c

    static void user_role_init(void)
    {
        //1.role init
        provision_init(MESH_ROLE_UNPROV_DEVICE,mesh_unprov_evt_cb);
        //2. data init
        unprov_data_init();
    }

**1. 定义协议栈内部事件通知回调函数**

.. code:: c

    /* unprovision device event callback function */
    static void mesh_unprov_evt_cb(mesh_prov_evt_type_t type , mesh_prov_evt_param_t param)
    {
        LOG(LOG_LVL_INFO,"mesh_unprov_evt_cb type : %d\n",type);

        switch(type)
        {
            case  UNPROV_EVT_INVITE_MAKE_ATTENTION : //(NO ACTION)
            {

            }
            break;
            case  UNPROV_EVT_EXPOSE_PUBLIC_KEY :  //(NO ACTION)
            {

            }
            break;
            case  UNPROV_EVT_AUTH_INPUT_NUMBER : //alert input dialog
            {

            }
            break;
            case  UNPROV_EVT_AUTH_DISPLAY_NUMBER : //unprov_device expose random number //(NO ACTION)
            {

            }
            break;
            case  UNPROV_EVT_PROVISION_DONE :  //(NO ACTION)
            {

            }
            break;
            default:break;
        }
    }


**2. 设置角色，注册事件回调**

.. code:: c

    provision_init(MESH_ROLE_UNPROV_DEVICE,mesh_unprov_evt_cb);


**3. 初始化角色相关的数据**

.. code:: c
    #define FLASH_TAG_SAVE_AUTH_VAL 0x2010
    #define FLASH_TAG_SAVE_BEACON_UUID 0x2020
  
    static void unprov_data_init(void)
    {
        volatile mesh_prov_evt_param_t evt_param;
  
        uint8_t  bd_addr[GAP_BD_ADDR_LEN];
    #if 1
        uint8_t value[16];
        uint8_t value_len = 16; 
        if(flash_multi_read(FLASH_TAG_SAVE_AUTH_VAL, value_len, value) == NVDS_OK) {
            memcpy(m_unprov_user.static_value, value, value_len);
        }   
        //show_buf("FLASH_TAG_SAVE_AUTH_VAL", value, value_len);
        if(flash_multi_read(FLASH_TAG_SAVE_BEACON_UUID, value_len, value) == NVDS_OK) {
            memcpy(m_unprov_user.beacon.dev_uuid, value, value_len);
        }   
        //show_buf("FLASH_TAG_SAVE_BEACON_UUID", value, value_len);
    #endif
  
        //get bd_addr
        mesh_core_params_t core_param;
        core_param.mac_address = bd_addr;
        mesh_core_params_get(MESH_CORE_PARAM_MAC_ADDRESS,&core_param);
  
        //copy mac to uuid
        memcpy(m_unprov_user.beacon.dev_uuid + 7, bd_addr, GAP_BD_ADDR_LEN);
  
        //1. Method of configuring network access
        evt_param.unprov.method = MESH_UNPROV_PROVISION_METHOD;
        provision_config(UNPROV_SET_PROVISION_METHOD,evt_param);
        //2. private key
        memcpy(m_unprov_user.unprov_private_key,bd_addr,GAP_BD_ADDR_LEN);
        evt_param.unprov.p_unprov_private_key = m_unprov_user.unprov_private_key;
        provision_config(UNPROV_SET_PRIVATE_KEY,evt_param);
        //3.static auth value
        evt_param.unprov.p_static_val = m_unprov_user.static_value;
        provision_config(UNPROV_SET_AUTH_STATIC,evt_param);
        //4.dev_capabilities
        evt_param.unprov.p_dev_capabilities = &m_unprov_user.dev_capabilities;
        provision_config(UNPROV_SET_OOB_CAPS,evt_param);
        //5.adv beacon
       //memcpy(m_unprov_user.beacon.dev_uuid,bd_addr,GAP_BD_ADDR_LEN);
        evt_param.unprov.p_beacon = &m_unprov_user.beacon;
        provision_config(UNPROV_SET_BEACON,evt_param);
    }


**4. 协议栈开始完整运行**

监听协议栈事件。。。。


_`ble mesh 协议栈和应用协议栈的信息交互`
==============================================

实现消息交互的处理函数
********************************

.. code:: c

    /* provision device event callback function */
    void user_config_server_evt_cb(config_server_evt_type_t type, config_server_evt_param_t*p_param)
  {
      LOG(LOG_LVL_INFO , "user_config_server_evt_cb=%d\n",type);
  
      switch(type)
      {
          case CONFIG_SERVER_EVT_RELAY_SET :
          {
          }
          break;
          case CONFIG_SERVER_EVT_APPKEY_ADD:
          {
              uint8_t status = 0;
              bind_appkey_to_model(&tmall_model_server_0.model.base, 0, &status);
              bind_appkey_to_model(&tmall_model_client_0.model.base, 0, &status);
              bind_appkey_to_model(&health_server_0.model.base, 0, &status);
              bind_appkey_to_model(&generic_onoff_client_0.model.base, 0, &status);
              bind_appkey_to_model(&generic_level_client_0.model.base, 0, &status);
              bind_appkey_to_model(&light_lightness_client_0.model.base, 0, &status);
              bind_appkey_to_model(&light_ctl_client_0.model.base, 0, &status);
              bind_appkey_to_model(&light_hsl_client_0.model.base, 0, &status);
  
              bind_appkey_to_model(&generic_onoff_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_lightness_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_ctl_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_hsl_server_0.model.base, 0, &status);

          }
          break;
          case CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_ADD:
          {
              break;
          }
          default:break;
      }
  }

根据收到的事件，做相应处理或回复
********************************
.. code:: h

  /** Configuration server event type. */
    typedef enum
    {
        CONFIG_SERVER_EVT_APPKEY_ADD,
        CONFIG_SERVER_EVT_APPKEY_UPDATE,
        CONFIG_SERVER_EVT_MODEL_PUBLICATION_SET,
        CONFIG_SERVER_EVT_APPKEY_DELETE,
        CONFIG_SERVER_EVT_BEACON_SET,
        CONFIG_SERVER_EVT_DEFAULT_TTL_SET,
        CONFIG_SERVER_EVT_FRIEND_SET,
        CONFIG_SERVER_EVT_GATT_PROXY_SET,
        CONFIG_SERVER_EVT_KEY_REFRESH_PHASE_SET,
        CONFIG_SERVER_EVT_MODEL_PUBLICATION_VIRTUAL_ADDRESS_SET,
        CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_ADD,
        CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_DELETE,
        CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_DELETE_ALL,
        CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_OVERWRITE,
        CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_VIRTUAL_ADDRESS_ADD,
        CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_VIRTUAL_ADDRESS_DELETE,
        CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_VIRTUAL_ADDRESS_OVERWRITE,
        CONFIG_SERVER_EVT_NETWORK_TRANSMIT_SET,
        CONFIG_SERVER_EVT_RELAY_SET,
        CONFIG_SERVER_EVT_LOW_POWER_NODE_POLLTIMEOUT_SET,
        CONFIG_SERVER_EVT_HEARTBEAT_PUBLICATION_SET,
        CONFIG_SERVER_EVT_HEARTBEAT_SUBSCRIPTION_SET,
        CONFIG_SERVER_EVT_MODEL_APP_BIND,
        CONFIG_SERVER_EVT_MODEL_APP_UNBIND,
        CONFIG_SERVER_EVT_NETKEY_ADD,
        CONFIG_SERVER_EVT_NETKEY_DELETE,
        CONFIG_SERVER_EVT_NETKEY_UPDATE,
        CONFIG_SERVER_EVT_NODE_IDENTITY_SET,
        CONFIG_SERVER_EVT_NODE_RESET,
    }config_server_evt_type_t;


.. code:: c
    void config_server_evt_act(config_server_evt_type_t type , config_server_evt_param_t param);
