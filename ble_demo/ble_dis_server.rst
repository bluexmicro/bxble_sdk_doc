====================================
ble dis server
====================================

功能简介
==========

    dis server实现了设备信息服务和ota服务，server提供的设备信息服务包括了很多和设备相关的信息，比如厂商名字、串号、固件版本号等，对端设备可以访问这些信息，bxota service用于ota升级，需要配合安卓apk使用。

ble 添加 service
======================

    dis server和uart server添加server的方式略有不同，dis server调用osapp_add_dis_server_task和osapp_add_bxotas_task函数，通过profile添加sever。在osapp dis server中，通过这种方式添加设备信息服务和ota服务。

.. code:: c

    static int32_t osapp_add_dis_server_task()
    {
        struct diss_db_cfg* db_cfg;
        // Allocate the DISS_CREATE_DB_REQ
        struct gapm_profile_task_add_cmd *req = AHI_MSG_ALLOC_DYN(GAPM_PROFILE_TASK_ADD_CMD,TASK_ID_GAPM, gapm_profile_task_add_cmd, sizeof(struct 
        diss_db_cfg));
        // Fill message
        req->operation = GAPM_PROFILE_TASK_ADD;
        req->sec_lvl = PERM(SVC_AUTH, NO_AUTH);
        req->prf_task_id = TASK_ID_DISS;
        req->app_task = TASK_AHI;
        req->start_hdl = 0;

        // Set parameters 
        db_cfg = (struct diss_db_cfg* ) req->param;
        db_cfg->features = DIS_ALL_FEAT_SUP;
        return os_ahi_msg_send(req,portMAX_DELAY);

      }

    static int32_t osapp_add_bxotas_task()
    {
        struct gapm_profile_task_add_cmd *req=AHI_MSG_ALLOC(GAPM_PROFILE_TASK_ADD_CMD, TASK_ID_GAPM, gapm_profile_task_add_cmd);
        req->operation = GAPM_PROFILE_TASK_ADD;
        req->sec_lvl = PERM(SVC_AUTH,NO_AUTH);
        req->prf_task_id = TASK_ID_BXOTAS;
        req->app_task = TASK_AHI;
        req->start_hdl = 0;
        return os_ahi_msg_send(req,portMAX_DELAY);
    }

服务的具体添加过程由profile完成，比如服务的声明，特征声明，注册到协议栈等。

ble 协议栈和应用协议栈的信息交互
==================================
  
**1. 协议栈和profile的交互**

.. code:: c

    const struct ke_msg_handler diss_default_state[] =
    {
        {DISS_SET_VALUE_REQ,      (ke_msg_func_t)diss_set_value_req_handler},
        {GATTC_READ_REQ_IND,      (ke_msg_func_t)gattc_read_req_ind_handler},
        {DISS_VALUE_CFM,          (ke_msg_func_t)diss_value_cfm_handler},
    };
    const struct ke_msg_handler bxotas_default_state[] =
    {
        {GATTC_WRITE_REQ_IND,      (ke_msg_func_t)gattc_write_req_ind_handler},
        {GATTC_READ_REQ_IND,      (ke_msg_func_t)gattc_read_req_ind_handler},
        {BXOTAS_FIRMWARE_DEST_CMD, (ke_msg_func_t)bxotas_firmware_dest_cmd_handler},
        {BXOTAS_START_CFM,    (ke_msg_func_t)bxotas_start_cfm_handler},

    };


**2. profile和应用的交互**

.. code:: c

    static const osapp_msg_handler_table_t handler_table[]=
    {
                    {DISS_VALUE_REQ_IND,     (osapp_msg_handler_t)osapp_diss_value_req_ind_handler},
                    {GAPC_DISCONNECT_IND,       (osapp_msg_handler_t)osapp_gapc_disconnect_ind_handler},
                    {GAPM_PROFILE_ADDED_IND,(osapp_msg_handler_t)osapp_gapm_profile_added_ind_handler},
                    {GAPC_GET_DEV_INFO_REQ_IND,(osapp_msg_handler_t)osapp_gapc_get_dev_info_req_ind_handler},
                    {GAPM_CMP_EVT,(osapp_msg_handler_t)osapp_gapm_cmp_evt_handler},
                    {BXOTAS_START_REQ_IND,(osapp_msg_handler_t)osapp_bxotas_start_req_ind_handler},
                    {BXOTAS_FINISH_IND,(osapp_msg_handler_t)osapp_bxotas_finish_ind_handler},
    };


profile是app和协议栈的中间层，有了profile，app和协议栈的交互容易得多，app只需要发送一条profile task add命令，profile就可以帮助app完成很多事，比如构建profile、处理来自对端设备的消息，然后再将处理的结果返回给app。

