==============================================
simple tmjl ctl anl demo
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
本示例功能主要实现 SIG 标准的 light_ctl_server, generic_onoff_server 和天猫精灵的 HSL 模型。
可以用于灯等设备模型。本示例实例化了一个element，该element包括一个 light ctl server, 
generic onoff server 和 vendor server 模型。初始化部分可以参考examples 目录下 node_setup.c 文件里面的
mesh_app_init_user函数说明，开发者可以非常容易添加更多的model。每个model初始化需要开发者初始化相关的控制接口，
在例子程序中各种 user_XXX_0_evt_cb 分别作为开发者接口，通过控制灯的亮灭颜色来进行示例。例子程序中的
generic_transition_server_0 model 是设置 default transtion time,当发送的 ctl 灯命令中不带有
transition time 和 delay时。ctl server 绑定了 lightness server, level server 和 onoff server,
如果发送lightness, level 和 onoff 的命令也会改变该灯的亮度和开关，系统也会将关键事件通知到开发者， 
开发者完成自己的关键事件处理函数即可，参考user_config_server_evt_cb 函数的实现，并在初始化进行注册。
另外，为了对系统进行控制，在element0 里面也初始化了SIG 的 config server model以便进行入网等相关的系统控制操作。

该示例主要体现的功能点如下：
********************************

* 设备支持 advertising，经 adv 通过天猫精灵连接快速配置入网。

* 设备支持mesh proxy，可以通过手机，经gatt连接快速配置入网。


* 节点上有一个light ctl server，可以通过手机或者天猫精灵单独控制任一 server。


* 节点支持分组，可以分组控制。


* 节点支持relay可控，可以通过 config 命令配置，便于部署。


_`示例运行概要`
===================

硬件环境
********************************
该示例运行在 BLE module 开发板上（开发板的详细信息，参考硬件支持包），用到硬件外设如下：
_______________________________________________________________________________________________

* 恢复出场设置

  快速重启灯板3次，灯每次运行起来之后灯运行的时间要在6s以内，会发现灯闪一下，设备重启并重新初始化，该操作会丢弃所有之前配置，设备变成unprovision 状态

* 指示灯

  * led1 :
       * 熄灭：
            generic onoff server 收到 onoff 值为0, 然后会将设置的 lightness 值变为 0, 如果别的消息只改 lightness 的值， 该值的最小值为 0x200；
       * 不同亮度和颜色
            light hsl server 收到的 lightness 的值不为 0，并且和 hue于 saturation 的值转换成
            RGB 值然后显示成不同的颜色。
  * led2 :
       * 熄灭：
            generic onoff server 收到 onoff 值为0, 然后会将设置的 lightness 值变为 0, 如果只改 lightness 的值， 该值的最小值为 0x200；
       * 不同亮度和色温
            light ctl server 收到的 lightness 的值不为 0, 然后将 lightness 和 temperature 对应的值转换成亮度和色温。

软件环境
********************************

* 设备端运行 ble mesh sdk 的 examples 目录下 simple_tmjl_ctl_anl示例。
* 用手机端或者天猫精灵入网调试，支持任意符合 mesh 标准的命令。

软件运行流程
********************************
**1. 用天猫精灵入网时需要注意的地方**
    * 在编译出来的 hex 的 0x80200 位置填入设备的 MAC 地址，该地址是需要申请的，如下所示：
        0xdd,0x07,0x03,0xca,0xd2,0x38
    * 在编译出来的 hex 的 0x802010 位置填上 USER UNPROV STATIC AUTH VAL, 该值必须要是用申请的三元组通过hash 256 生成前 16 个字节，如下所示: 
        0x70,0xc2,0xda,0x36,0x3a,0x88,0xd1,0xad,0x3a,0x4e,0xba,0x2d,0x8f,0x6d,0x16,0x4f
    * 在编译出来的 hex 的 0x802020 位置上填上 USER UNPROV BEACON UUID 宏前面字节必须是注册的 company ID, Product ID 和当前设备的 MAC 地址组成,具体组成如下所示：
        0xa8,0x01,\
        0x51,\
        0x80,0x0B,0x00,0x00,\
        0xdd,0x07,0x03,0xca,0xd2,0x38,\
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
        simple_tmjl_ctl_anl_init();

        LOG(LOG_LVL_INFO,"mesh_user_main_init\n");
    }

例程初始状态
********************************
设备正常上电后：
  * 入网前:
       * 非清网重启: LED1 灭，LED2 亮度 100%， 色温 50%;
       * 清网重启: LED1 灭， LED2 亮度 0 和 100% 每个 1s 闪烁， 色温 50% 亮;
  * 入网后 :
       * LED1 灭， LED2 亮度 100%， 色温 50% 亮;


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
        uint8_t value[16];
        uint8_t value_len = 16;
        #if 1
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
        //    memcpy(m_unprov_user.beacon.dev_uuid,bd_addr,GAP_BD_ADDR_LEN);
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
          case CONFIG_SERVER_EVT_APPKEY_ADD:
          {
              uint8_t status = 0;
              bind_appkey_to_model(&scene_server_0.model.base, 0, &status);
              bind_appkey_to_model(&scene_setup_server_0.model.base, 0, &status);
              bind_appkey_to_model(&generic_onpowerup_server_0.model.base, 0, &status);
              bind_appkey_to_model(&generic_onpowerup_setup_server_0.model.base, 0, &status);
              bind_appkey_to_model(&custom_vendor_server_0.model.base, 0, &status);
              bind_appkey_to_model(&custom_vendor_client_0.model.base, 0, &status);
              bind_appkey_to_model(&generic_level_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_ctl_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_ctl_setup_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_ctl_temperature_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_lightness_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_lightness_setup_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_hsl_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_hsl_setup_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_hsl_hue_server_0.model.base, 0, &status);
              bind_appkey_to_model(&light_hsl_saturation_server_0.model.base, 0, &status);
              bind_appkey_to_model(&health_server_0.model.base, 0, &status);
          }
          break;
          case CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_ADD:
          {
              config_model_subscription_add(scene_server_0.model.base.elmt, &scene_server_0.model.base, 0xc000);
              config_model_subscription_add(scene_setup_server_0.model.base.elmt, &scene_setup_server_0.model.base, 0xc000);
              config_model_subscription_add(generic_onpowerup_server_0.model.base.elmt, &generic_onpowerup_server_0.model.base, 0xc000);
              config_model_subscription_add(generic_onpowerup_setup_server_0.model.base.elmt, &generic_onpowerup_setup_server_0.model.base, 0xc000);
              config_model_subscription_add(custom_vendor_server_0.model.base.elmt, &custom_vendor_server_0.model.base, 0xc000);
              config_model_subscription_add(custom_vendor_client_0.model.base.elmt, &custom_vendor_client_0.model.base, 0xc000);
              config_model_subscription_add(generic_level_server_0.model.base.elmt, &generic_level_server_0.model.base, 0xc000);
              config_model_subscription_add(light_ctl_server_0.model.base.elmt, &light_ctl_server_0.model.base, 0xc000);
              config_model_subscription_add(light_ctl_setup_server_0.model.base.elmt, &light_ctl_setup_server_0.model.base, 0xc000);
              config_model_subscription_add(light_ctl_temperature_server_0.model.base.elmt, &light_ctl_temperature_server_0.model.base, 0xc000);
              config_model_subscription_add(light_lightness_server_0.model.base.elmt, &light_lightness_server_0.model.base, 0xc000);
              config_model_subscription_add(light_lightness_setup_server_0.model.base.elmt, &light_lightness_setup_server_0.model.base, 0xc000);
              config_model_subscription_add(light_hsl_server_0.model.base.elmt, &light_hsl_server_0.model.base, 0xc000);
              config_model_subscription_add(light_hsl_setup_server_0.model.base.elmt, &light_hsl_setup_server_0.model.base, 0xc000);
              config_model_subscription_add(light_hsl_hue_server_0.model.base.elmt, &light_hsl_hue_server_0.model.base, 0xc000);
              config_model_subscription_add(light_hsl_saturation_server_0.model.base.elmt, &light_hsl_saturation_server_0.model.base, 0xc000);
              config_model_subscription_add(health_server_0.model.base.elmt, &health_server_0.model.base, 0xc000);
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
