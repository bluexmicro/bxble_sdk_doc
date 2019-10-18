====================================
ble demo执行流程
====================================

**1. main函数**

    在 main.c 中，soc_initialize完成对soc的初始化，rtos_task_init初如化ble协议栈，ble 开始运行。

**2. app初始化**

    在应用层，用户需要关心的是user_init函数，在每个peripheral demo中都有这个函数，比如在uart server中，该函数位于osapp_uart_server.c中。该函数配置ble的角色和地扯以及绑定初始化、注册回调函数。

**3. 设置蓝牙地扯**

       在user_init中，通过调用osapp_utils_set_dev_init设置角色为GAP_ROLE_PERIPHERAL,地扯类型（PUBLIC/STATIC）,当地扯为PUBLIC时，设备从flash 0X2000地扯获取ble address（在程序烧写时，可通过改变0x802000后面连续6个字节指定地扯，当地扯类型为STATIC时，地扯随机生成。

**4. 复位协议栈**

    当user_init执行完成后，会收到来自协议栈的第一条消息，即GAPM_DEVICE_READY_IND，然后app发送GAPM_RESET_CMD使协议栈复位（该过程在osapp_utils.c中完成）。

**5. 配置设备参数**

    协议栈完成复位后，app会收一条GAPM_CMP_EVT消息，调用相应的回调函数osapp_gapm_cmp_evt_handler，app调用osapp_set_dev_config初配置一些必要参数（比如蓝牙地扯、MTU等）。

.. code:: c

    static void osapp_set_dev_config(void)
    {

        // Set Device configuration
        struct gapm_set_dev_config_cmd* cmd = AHI_MSG_ALLOC(GAPM_SET_DEV_CONFIG_CMD,TASK_ID_GAPM,gapm_set_dev_config_cmd);
        cmd->operation = GAPM_SET_DEV_CONFIG;
        cmd->role      = l_utils.role;
        //privacy configuration
        cmd->renew_dur = GAP_TMR_PRIV_ADDR_INT;

        uint8_t addr_len = GAP_BD_ADDR_LEN;
        uint8_t irk_len = GAP_KEY_LEN;
        bool nvds_update = false;
        if(nvds_get(NVDS_TAG_STATIC_DEV_ADDR,&addr_len,cmd->addr.addr)!=NVDS_OK)
        {
            osapp_utils_random_generate(cmd->addr.addr,addr_len);//random addr;
            UTILS_SET_BD_ADDR_STATIC(cmd->addr.addr);//set mask
            nvds_put(NVDS_TAG_STATIC_DEV_ADDR,addr_len,cmd->addr.addr);
            nvds_update = true;
        }
        if(nvds_get(NVDS_TAG_LOCAL_IRK,&irk_len,cmd->irk.key)!=NVDS_OK)
        {
            osapp_utils_random_generate(cmd->irk.key, irk_len);//random addr;
            nvds_put(NVDS_TAG_LOCAL_IRK,irk_len,cmd->irk.key);\
            nvds_update = true;
        }
        if(nvds_update)
        {
            nvds_write_through();
        }
        cmd->addr_type = l_utils.addr_type;
        //security configuration
        cmd->pairing_mode = GAPM_PAIRING_LEGACY|GAPM_PAIRING_SEC_CON ;
        //attribute database configuration
        cmd->gap_start_hdl = 0;
        cmd->gatt_start_hdl = 0;
        cmd->att_cfg = GAPM_MASK_ATT_SVC_CHG_EN | GAPM_MASK_ATT_SLV_PREF_CON_PAR_EN;
        //LE Data Length Extension configuration
        cmd->sugg_max_tx_octets = BLE_MAX_OCTETS;
        cmd->sugg_max_tx_time   = BLE_MAX_TIME;
        //L2CAP Configuration
        cmd->max_mps = GAP_MAX_LE_MTU;
        cmd->max_mtu = user_max_mtu;
        cmd->max_nb_lecb = 0;
        //LE Audio Mode Supported
        cmd->audio_cfg = 0;//GAPM_MASK_AUDIO_AM0_SUP;
        //LE PHY Management
        cmd->tx_pref_rates = GAP_RATE_LE_1MBPS | GAP_RATE_LE_2MBPS;
        cmd->rx_pref_rates = GAP_RATE_LE_1MBPS | GAP_RATE_LE_2MBPS;
        os_ahi_msg_send(cmd, portMAX_DELAY);
    }

**6. service添加**

    每个demo都有不同的service,添加service的方法也不一样，可以通过构造service的database的方式添加，也可以调用现有的profile的方式添加服务。


