==============================================
simple light hsl ctl demo
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
本示例功能主要实现 SIG 标准的 light hsl server 和 light ctl server model，可以用于灯等设备模型。
本示例实例化了两个element，第一个element包括一个 hsl server model，第二个element包括一个 ctl server model初始化部分
可以参考examples 目录下 node_setup.c 文件里面的 mesh_app_init_user函数说明，开发者
可以非常容易添加更多的model。每个model初始化需要开发者初始化相关的控制接口，在例子
程序中user_hsl_0_evt_cb，user_ctl_1_evt_cb分别作为两个开发者接口，通过控制灯
的亮灭来进行示例。例子程序中的 generic_transition_server_0 和 generic_transition_server_1
两个 model 是设置 default transtion time,当发送的 hsl 和 ctl 命令中不带有 transition time 和 delay时。
hsl server 绑定了 lightness server, level server 和 onoff server, 如果发送lightness, level 和
onoff 的命令也会改变该灯的亮度，系统也会将关键事件通知到开发者，开发者完成自己的关键事件处理
函数即可，参考user_config_server_evt_cb 函数的实现，并在初始化进行注册。另外，为了对系统进行
控制，在element0 里面也初始化了SIG 的 config server model以便进行入网等相关的系统控制操作。

该示例主要体现的功能点如下：

********************************


* 设备支持mesh proxy，可以通过手机，经gatt连接快速配置入网。


* 节点上有一个light hsl server，一个 light ctl server 可以通过手机单独控制任一 server。


* 节点支持分组，可以分组控制。


* 节点支持relay可控，可以通过 config 命令配置，便于部署。


_`示例运行概要`
===================

硬件环境
********************************
该示例运行在 BLE Dongle 开发板上（开发板的详细信息，参考硬件支持包），用到硬件外设如下：
_______________________________________________________________________________________________

* 按键

  botton 3  button 4 同时按下 ，直到绿灯闪一下，设备重启并重新初始化，该操作会丢弃所有之前配置，设备变成unprovision 状态

* 指示灯

  * led1 :
       * 熄灭：
            light hsl server **1** 设置的 lightness 值为 0；
       * 不同亮度和颜色
            light hsl server **1** 设置的 lightness 的值为不为 0 的值，并且和 hue于 saturation 的值转换成
            RGB 值然后显示成不同的颜色。
  * led2 :
       * 绿灯
                * 闪烁，每隔一段时间亮灭一次；
       * 蓝灯
                * 熄灭:
                    *light ctl server **2** 设置亮度的 lightness 值设置为0；
                * 不同亮度:
                    *light ctl server **2** 设置亮度的 lightness 为 0 的不同亮度
       * 红灯
                * 熄灭:
                    *light ctl server **2** 设置色温的 temperature 值设置为0；
                * 不同亮度:
                    *light ctl server **2** 设置亮度的 temperature 为 0 的不同亮度

软件环境
********************************
* 设备端运行 ble mesh sdk 的 examples 目录下 simple_light_hsl_ctl示例。
* 手机端运行 任意厂商符合mesh标准的app。

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
        simple_light_hsl_ctl_init();

        LOG(LOG_LVL_INFO,"mesh_user_main_init\n");
    }

例程初始状态
********************************
设备正常上电后：
  * led1 :
       * 常亮, 默认为白色的光，此时亮度为 50%， lightness 的值为 0x8000, hue 的值为0 ，saturation 的值为0；
  * led2 :
       * 绿灯
                * 闪烁，每隔一段时间亮灭一次；；
       * 红灯
                * 熄灭

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
          }
          break;
          case CONFIG_SERVER_EVT_MODEL_SUBSCRIPTION_ADD:
          {
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
