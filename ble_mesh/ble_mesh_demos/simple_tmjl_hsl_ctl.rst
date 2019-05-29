==============================================
simple tmjl hsl ctl demo
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
本示例功能主要实现 SIG 标准的 light_hsl_server, light_ctl_server, generic_onoff_server 和天猫精灵的 HSL 模型。
可以用于灯等设备模型。本示例实例化了一个element，每个element包括一个 hsl server model，light ctl server, 
generic onoff server 和 vendor server 模型。初始化部分可以参考examples 目录下 mesh_app.c 文件里面的
mesh_app_init_user函数说明，开发者可以非常容易添加更多的model。每个model初始化需要开发者初始化相关的控制接口，
在例子程序中各种 user_XXX_0_evt_cb 分别作为开发者接口，通过控制灯的亮灭颜色来进行示例。例子程序中的
generic_transition_server_0 model 是设置 default transtion time,当发送的 hsl,ctl 灯命令中不带有
transition time 和 delay时。ctl server 和 hsl server 绑定了 lightness server, level server 和 onoff server,
如果发送lightness, level 和 onoff 的命令也会改变该灯的亮度和开关，系统也会将关键事件通知到开发者， 
开发者完成自己的关键事件处理函数即可，参考user_config_server_evt_cb 函数的实现，并在初始化进行注册。
另外，为了对系统进行控制，在element0 里面也初始化了SIG 的 config server model以便进行入网等相关的系统控制操作。

该示例主要体现的功能点如下：
********************************

* 设备支持 advertising，经 adv 通过天猫精灵连接快速配置入网。

* 设备支持mesh proxy，可以通过手机，经gatt连接快速配置入网。


* 节点上有两个light hsl server，可以通过手机单独控制任一 server。


* 节点支持分组，可以分组控制。


* 节点支持relay可控，可以手动打开或关闭relay，便于部署。


* 节点支持proxy server beacon 可控，可以手动打开或关闭该beacon，便于部署。


_`示例运行概要`
===================

硬件环境
********************************
该示例运行在 BLE Dongle 开发板上（开发板的详细信息，参考硬件支持包），用到硬件外设如下：
_______________________________________________________________________________________________

* 按键

  botton 3  button 4 同时按下 ，直到蓝灯闪一下，设备重启并重新初始化，该操作会丢弃所有之前配置，设备变成unprovision 状态

* 指示灯

  * led1 :
       * 熄灭：
            light hsl server **1** 设置的 lightness 值为 0；
       * 不同亮度和颜色
            light hsl server **1** 设置的 lightness 的值为不为 0 的值，并且和 hue于 saturation 的值转换成
            RGB 值然后显示成不同的颜色。
  * led2 :
       * 蓝灯
                * 默认熄灭， 系统 reset 时闪有一下；
       * 绿灯
       * 红灯
                * 熄灭:
                    * light ctl server 设置亮度的 lightness 值设置为 0。
                * 不同亮度:
                    * light ctl server 设置亮度的 lightness 为 0 的不同亮度,
                      并且与 temperature 和 lightness 的值有关。

软件环境
********************************

* 设备端运行 ble mesh sdk 的 examples 目录下 simple_tmjl_hsl_ctl示例。
* 用手机端或者天猫精灵入网调试，支持任意符合 mesh 标准的命令。

软件运行流程
********************************
**1. 用天猫精灵入网时需要注意的地方**
    * USER_UNPROV_STATIC_AUTH_VAL 宏必须要是用三元组生成前 16 个字节，如下所示: 
        #define USER_UNPROV_STATIC_AUTH_VAL       0x70,0xc2,0xda,0x36,0x3a,0x88,0xd1,0xad,0x3a,0x4e,0xba,0x2d,0x8f,0x6d,0x16,0x4f
    * USER_UNPROV_BEACON_UUID 宏前面字节必须是注册的 company ID, Product ID 和当前设备的 MAC 地址组成,具体组成如下所示：
        #define  USER_UNPROV_BEACON_UUID          0xa8,0x01,\
                                                  0x51,\
                                                  0x80,0x0B,0x00,0x00,\
                                                  0xdd,0x07,0x03,0xca,0xd2,0x38,\
                                                  0x00,0x00,0x00  
    * BX_DEV_ADDR为该设备的 MAC 地址，该地址是需要申请的，如下所示：
        #define BX_DEV_ADDR {0xdd,0x07,0x03,0xca,0xd2,0x38}

**2. 用户自己函数入口**

   在 mesh_user_main.c 中， mesh_user_main_init()初始化自己数据。（需要注意：不能阻塞）

**3. 开启 mesh 协议栈调度**

   在用户函数执行完后，系统自动开启ble协议栈调度。

**4. 示例代码**

.. code:: c

    void mesh_user_main_init(void)
    {
        ///user data init
        simple_tmjl_hsl_ctl_init();

        LOG(LOG_LVL_INFO,"mesh_user_main_init\n");
    }

例程初始状态
********************************
设备正常上电后：
  * 入网前:
       * 所有的灯都是熄灭的。
  * 入网后 :
       * 所有的灯都是你关闭之前灯显示的颜色和亮度，如果关闭电源之前，灯是关闭状态，
       *     等再次上电都是亮的状态，并且是关电源之前亮的最后一次状态。



_`ble mesh 角色设置`
===================================================================================================================

设置流程
********************************
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

    static void unprov_data_init(void)
    {
        volatile mesh_prov_evt_param_t evt_param;

        uint8_t  bd_addr[GAP_BD_ADDR_LEN];

        //get bd_addr
        mesh_core_params_t core_param;
        core_param.mac_address = bd_addr;
        mesh_core_params_get(MESH_CORE_PARAM_MAC_ADDRESS,&core_param);

        //1. Method of configuring network access
        evt_param.unprov.method = PROVISION_BY_GATT;
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
        memcpy(m_unprov_user.beacon.dev_uuid,bd_addr,GAP_BD_ADDR_LEN);
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
      /** A new application key was added. */
      CONFIG_SERVER_EVT_APPKEY_ADD,
      /** An existing application key was updated. */
      CONFIG_SERVER_EVT_APPKEY_UPDATE,
      /** The publication paremeters for a given model was set. */
      CONFIG_SERVER_EVT_MODEL_PUBLICATION_SET,
      /** The given application key was deleted. */
      CONFIG_SERVER_EVT_APPKEY_DELETE,
      /** Secure network beacon parameters was set. */
      CONFIG_SERVER_EVT_BEACON_SET,
      /** A new default TTL value was set. */
      CONFIG_SERVER_EVT_DEFAULT_TTL_SET,
      /** Friendship parameters was set (not supported). */
      CONFIG_SERVER_EVT_FRIEND_SET,
      /** GATT proxy parameters was set (not supported). */
      CONFIG_SERVER_EVT_GATT_PROXY_SET,
      /** Key refresh phase was set. */
      CONFIG_SERVER_EVT_KEY_REFRESH_PHASE_SET,
      /** Publication to a virtual address for a given model was set. */
      CONFIG_SERVER_EVT_MODEL_PUBLICATION_VIRTUAL_ADDRESS_SET,
      /** A subscription was added to the given model. */
      CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_ADD,
      /** A subscription was deleted from the given model. */
      CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_DELETE,
      /** All subscriptions was deleted for the given model. */
      CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_DELETE_ALL,
      /** All subscriptions was overwritten by a new subscription for the given model. */
      CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_OVERWRITE,
      /** A subscription to a virtual address was added to the given model. */
      CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_VIRTUAL_ADDRESS_ADD,
      /** A subscription to a virtual address was removed from the given model. */
      CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_VIRTUAL_ADDRESS_DELETE,
      /** All subscriptions was overwritten by a new subscription to a virtual address for the given model. */
      CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_VIRTUAL_ADDRESS_OVERWRITE,
      /** Core network transmission parameters was set. */
      CONFIG_SERVER_EVT_NETWORK_TRANSMIT_SET,
      /** Core relay parameters was set. */
      CONFIG_SERVER_EVT_RELAY_SET,
      /** Low power node poll timeout was set (not supported). */
      CONFIG_SERVER_EVT_LOW_POWER_NODE_POLLTIMEOUT_SET,
      /** Heartbeat publication parameters was set. */
      CONFIG_SERVER_EVT_HEARTBEAT_PUBLICATION_SET,
      /** Heartbeat subscription parameters was set. */
      CONFIG_SERVER_EVT_HEARTBEAT_SUBSCRIPTION_SET,
      /** The given model was bound to a new application key. */
      CONFIG_SERVER_EVT_MODEL_APP_BIND,
      /** The given model was unbound from an application key. */
      CONFIG_SERVER_EVT_MODEL_APP_UNBIND,
      /** A new network key was added. */
      CONFIG_SERVER_EVT_NETKEY_ADD,
      /** A network key was deleted. */
      CONFIG_SERVER_EVT_NETKEY_DELETE,
      /** A network key was updated. */
      CONFIG_SERVER_EVT_NETKEY_UPDATE,
      /** The Node Identity was set (not supported). */
      CONFIG_SERVER_EVT_NODE_IDENTITY_SET,
      /** The node was reset, i.e., all mesh state cleared. */
      CONFIG_SERVER_EVT_NODE_RESET,
  }config_server_evt_type_t;

