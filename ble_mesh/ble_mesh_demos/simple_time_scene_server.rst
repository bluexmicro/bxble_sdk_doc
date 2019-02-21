==============================================
simple time scene server demo
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
本示例功能主要实现SIG 标准的 time scene server model，可以用于灯等设备模型。
本示例实例化了两个element，每个element包括一个 scene server model，初始化部分
可以参考examples 目录下 mesh_app.c 文件里面的 mesh_app_init_user函数说明，开发者
可以非常容易添加更多的model。每个model初始化需要开发者初始化相关的控制接口，在例子
程序中user_scene_0_evt_cb，user_scene_1_evt_cb分别作为两个开发者接口，可以设置当前场景，
并且恢复之前的场景，即设置时灯的颜色亮度值，保存的最大的场景值为 15。例子程序中的 
generic_transition_server_0 和 generic_transition_server_1两个 model 是设置 
default transtion time,当发送的 hsl 命令中不带有 transition time 和 delay 时。hsl server
绑定了 lightness server, level server 和 onoff server, 如果发送lightness, level 和
onoff 的命令也会改变该灯的亮度，系统也会将关键事件通知到开发者，开发者完成自己
的关键事件处理函数即可。参考user_config_server_evt_cb函数的实现，并在初始化进行注册。
另外，为了对系统进行控制，在element0 里面也初始化了SIG 的 config server model以便
进行入网等相关的系统控制操作。

该示例主要体现的功能点如下：

********************************


* 设备支持mesh proxy，可以通过手机，经gatt连接快速配置入网。


* 节点上有两个 element server，可以通过手机单独控制任一 elememt。


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

  botton 3  button 4 同时按下 ，直到绿灯闪一下，，设备重启并重新初始化，该操作会丢弃所有之前配置，设备变成unprovision 状态

* PIN

  PIN 10 11短路 ：  延迟一分钟后 关闭proxy server beacon 功能打开，LED1红灯亮

* 指示灯

  * led1 :
      * 熄灭：
            light hsl server **1** 设置的 lightness 值为 0；
       * 不同亮度和颜色
            light hsl server **1** 设置的 lightness 的值为不为 0 的值，并且和 hue于 saturation 的值转换成
            RGB 值然后显示成不同的颜色。
  * led2 :
       * 绿灯
                * 熄灭， 保留；
       * 蓝灯
                * 熄灭， 保留；
       * 红灯
                * 熄灭:
                    *light hsl server **2** 设置亮度的 lightness 值设置为0；
                * 不同亮度:
                    *light hsl server **2** 设置亮度的 lightness 为 0 的不同亮度,
                        并且还与 hue 和 saturation 有关。

软件环境
********************************
* 设备端运行 ble mesh sdk 的 examples 目录下 simple_time_scene_server示例。
* 手机端运行 任意厂商符合mesh标准的app。

软件运行流程
********************************

**1. 用户自己函数入口**

   在 mesh_user_main.c 中， mesh_user_main_init()初始化自己数据。（需要注意：不能阻塞）

**2. 开启mesh协议栈调度**

   在用户函数执行完后，系统自动开启 ble 协议栈调度。

**3. 示例代码**

.. code:: c

    void mesh_user_main_init(void)
    {
        ///user data init
        simple_time_scene_server_init();

        LOG(LOG_LVL_INFO,"mesh_user_main_init\n");
    }

例程初始状态
********************************
设备正常上电后：
  * led1 :
        * 常亮, 默认为白色的光，此时亮度为 50%， lightness 的值为 0x8000, hue 的值为0 ，saturation 的值为0；
  * led2 :
       * 绿灯
                * 熄灭， 保留；
       * 蓝灯
                * 熄灭， relay 功能默认关闭；
       * 红灯
                * 常亮， light hsl server **2** 默认设置打开亮度 为 50%,此时 lightness 的值为0x8000；



_`ble mesh 角色设置`
===================================================================================================================

设置流程
********************************

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

根据收到的事件，做相应处理或回复
********************************

.. code:: c

    //协议->用户
    typedef enum
    {
        /*******PROVISIONER*******/
        PROV_EVT_BEACON,
        PROV_EVT_CAPABILITIES,
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

