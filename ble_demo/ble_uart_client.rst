====================================
ble uart client
====================================

功能简介
==========

    uart client接合uart server实现两个蓝牙设备间的透传，uart client和uart server连接成功后，client等待server发送数据，当client检测到server发送的数据时，将数据回写给server。

设置对端地扯
======================

.. code:: c

    const static struct gap_bdaddr slave_add = 
    {
       .addr.addr = SLAVE_ADDR,
       .addr_type = 0,
    };

slave_add为要连接的对端设备地扯和地扯类型，即uart server的地扯和地扯类型。

ble 协议栈和应用协议栈的信息交互
==================================
.. code:: c

    static osapp_msg_handler_table_t osapp_svc_handler_table[] =
    {
        {APM_CMP_EVT,(osapp_msg_handler_t)osapp_gapm_cmp_evt_handler},
        {GAPM_ADV_REPORT_IND, (osapp_msg_handler_t)osapp_gapm_scan_adv_report_ind_handler},
        {GATTC_EVENT_IND, (osapp_msg_handler_t)osapp_event_ind_handler},
        {GATTC_SDP_SVC_IND, (osapp_msg_handler_t)osapp_sdp_disc_ind_handler},
        {GATTC_CMP_EVT,(osapp_msg_handler_t)osapp_gattc_cmp_evt_handler},
    };

当uart client收到uart server notify过来的数据时，协议栈通过GATTC_EVENT_IND通知应用程序，然后调用osapp_event_ind_handler函数，在该函数里可以获取uart server发过来的数据。

uart client发送数据接口
=========================

.. code:: c

    static void osapp_gatt_write(uint8_t op, uint8_t hdl, uint8_t conn_id, uint8_t length, const uint8_t* pdat)// write no respose
    {
        static uint16_t seq_num = 0;
        struct gattc_write_cmd* cmd  = AHI_MSG_ALLOC_DYN(GATTC_WRITE_CMD, KE_BUILD_ID(TASK_ID_GATTC, conn_id), gattc_write_cmd, length);
        memset(cmd, 0, (sizeof(struct gattc_write_cmd)+length));
        cmd->operation = op;
        cmd->auto_execute = 1;
        cmd->seq_num = seq_num++;
        cmd->handle = hdl;
        cmd->offset = 0;
        cmd->length = length;
        cmd->cursor = 0;
        memcpy(cmd->value, pdat, length);
        os_ahi_msg_send(cmd,portMAX_DELAY); 
    }

当连接成功后，并且也已经成功发现对端设备上的服务后，就可以调用该函数发送数据给uart server。






