====================================
ble ancs client
====================================

功能简介
==========

    ancs client用于和苹果手机通信，苹果手机上有一个ancs服务，可以将手机上的消息推送到与之绑定的蓝牙设备。在发现苹果手机上的ancs服务前，必须要和苹果手机绑定，才能发现服务。苹果手机上的服务可能由于某些原因发生改变，这就需要绑定过的设备重新去发现ancs服务。当苹果手机的服备发生改变时， 我们希望他能通知与之绑定过的设备服务已经发生了改变，这就需要设备去使能苹果手机的service changed char（通过将cccd置1），由gattsc profile完成该过程。

ble 添加 profile
======================

.. code:: c

    static void osapp_add_ancc_task()
    {
        struct gapm_profile_task_add_cmd *req=AHI_MSG_ALLOC(GAPM_PROFILE_TASK_ADD_CMD, TASK_ID_GAPM, gapm_profile_task_add_cmd);
        req->operation = GAPM_PROFILE_TASK_ADD;
        req->sec_lvl = PERM(SVC_AUTH,NO_AUTH);
        req->prf_task_id = TASK_ID_ANCC;
        req->app_task = TASK_AHI;
        req->start_hdl = 0;
        os_ahi_msg_send(req,portMAX_DELAY);
    }

    static void osapp_add_gattsc_task(void)
    {
        struct gapm_profile_task_add_cmd *req=AHI_MSG_ALLOC(GAPM_PROFILE_TASK_ADD_CMD, TASK_ID_GAPM, gapm_profile_task_add_cmd);
        req->operation = GAPM_PROFILE_TASK_ADD;
        req->sec_lvl = PERM(SVC_AUTH,NO_AUTH);
        req->prf_task_id = TASK_ID_GATTSC;
        req->app_task = TASK_AHI;
        req->start_hdl = 0;
        os_ahi_msg_send(req,portMAX_DELAY);
    }


ble 协议栈和应用协议栈的信息交互
==================================
  
**1. 协议栈和profile的交互**

.. code:: c

    const struct ke_msg_handler ancc_default_state[] =
    {
        {GATTC_CMP_EVT,(ke_msg_func_t)gattc_cmp_evt_handler},
        {GATTC_SDP_SVC_IND,(ke_msg_func_t)gattc_sdp_svc_ind_handler},
        {ANCC_ENABLE_REQ,(ke_msg_func_t)ancc_enable_req_handler},
        {ANCC_CL_CFG_NTF_EN_CMD,(ke_msg_func_t)ancc_cl_cfg_ntf_en_cmd_handler},
        {GATTC_EVENT_IND,(ke_msg_func_t)gattc_event_ind_handler},
        {ANCC_GET_NTF_ATTS_CMD,(ke_msg_func_t)ancc_get_ntf_atts_cmd_handler},
        {ANCC_GET_APP_ATTS_CMD,(ke_msg_func_t)ancc_get_app_atts_cmd_handler},
        {ANCC_PERFORM_NTF_ACTION_CMD,(ke_msg_func_t)ancc_perform_ntf_action_cmd_handler},
    };

    const struct ke_msg_handler gattsc_default_state[] =
    {
        {GATTC_CMP_EVT,(ke_msg_func_t)gattc_cmp_evt_handler},
        {GATTC_SDP_SVC_IND,(ke_msg_func_t)gattc_sdp_svc_ind_handler},
        {GATTSC_ENABLE_REQ,(ke_msg_func_t)gattsc_enable_req_handler},
        {GATTSC_SVC_CHANGED_IND_CFG_CMD,(ke_msg_func_t)gattsc_svc_changed_ind_cfg_cmd_handler},
        {GATTC_EVENT_REQ_IND,(ke_msg_func_t)gattc_event_req_ind_handler},
    };



**2. profile和应用的交互**

.. code:: c

    static osapp_msg_handler_table_t const handler_table[]=
    {
    //gapm
        {GAPM_PROFILE_ADDED_IND,(osapp_msg_handler_t)osapp_gapm_profile_added_ind_handler},
        {GAPM_CMP_EVT,(osapp_msg_handler_t)osapp_gapm_cmp_evt_handler},
        {GAPC_DISCONNECT_IND,(osapp_msg_handler_t)osapp_gapc_disconnect_ind_handler},
        {GAPC_GET_DEV_INFO_REQ_IND,(osapp_msg_handler_t)osapp_gapc_get_dev_info_req_ind_handler},
    //profile
        {ANCC_ENABLE_RSP,(osapp_msg_handler_t)osapp_ancc_enable_rsp_handler},
        {ANCC_CMP_EVT,(osapp_msg_handler_t)osapp_ancc_cmp_evt_handler},
        {ANCC_NTF_ATT_IND,(osapp_msg_handler_t)osapp_ancc_ntf_att_ind_handler},
        {ANCC_APP_ATT_IND,(osapp_msg_handler_t)osapp_ancc_app_att_ind_handler},
        {ANCC_NTF_SRC_IND,(osapp_msg_handler_t)osapp_ancc_ntf_src_ind_handler},
    //gattsc
        {GATTSC_ENABLE_RSP,(osapp_msg_handler_t)osapp_gattsc_enable_rsp_handler},
        {GATTSC_SVC_CHANGED_IND,(osapp_msg_handler_t)osapp_gattsc_svc_chaged_handler},
        {GATTSC_CMP_EVT,(osapp_msg_handler_t)osapp_gattsc_cmp_evt_handler},
    };

profile是app和协议栈的中间层，有了profile，app和协议栈的交互容易得多，app只需要发送一条profile task add命令，profile就可以帮助app完成很多事，比如构建profile、处理来自对端设备的消息，然后再将处理的结果返回给app。对于ancs，app首先发送一条ANCC_ENABLE_REQ给profile，profile去发现苹果手机的ancs，然后将发现的ancs有关信息通信ANCC_ENABLE_RSP返回给应用层，应该层接着去使能src_data cccd和ntf_data cccd。就能够接收到来自苹果手机的消息推送。对于service changed char，app首先发送GATTSC_ENABLE_REQ到gattsc profile， gattsc profile去发现对端的gatt的service changed char,然后再去使能cccd，允许指示。当苹果手机上的服务发生变化时，会将发生变化的handle指示给与之绑定的设备