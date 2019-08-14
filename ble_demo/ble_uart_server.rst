====================================
ble uart server
====================================

功能简介
==========

    uart server主要实现了一个透传功能，sever接收来自串口的数据，并把数据发送到对端设备，对端设备也可以将数据通过蓝牙写入server，server收到数据后，将数据通过串口输出。

ble 自定义 service
======================

    自定义service的本质就是构造一个service，即构造服务描述、特征声明，特征值，特征描述符，造完成之后再将这个服务注册到协议栈。

**1. 定义自己的service 描述**

.. code:: c

	static const struct gattm_svc_desc bx_simple_svc_desc = {
			.start_hdl = 0,
			.task_id = TASK_ID_AHI,
			.perm = PERM(SVC_MI,DISABLE)|PERM(SVC_EKS,DISABLE)|\
				PERM(SVC_AUTH,NO_AUTH)|PERM(SVC_UUID_LEN,UUID_128),PERM_VAL(SVC_SECONDARY,0),
			.nb_att = BX_SIMPLES_ATT_NUM,
			.uuid = BX_SIMPLE_SVC_UUID_128,
	};

**2. 定义自己的att 属性列表**

.. code:: c

    struct gattm_att_desc const uart_svc_att_db[UART_SVC_ATT_NUM] = {
            [UART_SVC_IDX_RX_CHAR] = {
                .uuid = ATT_DECL_CHAR_ARRAY,
                .perm = PERM(RD,ENABLE),
                .max_len = 0,
                .ext_perm= PERM(UUID_LEN,UUID_16),
            },
            [UART_SVC_IDX_RX_VAL] = {
                .uuid = UART_SVC_RX_CHAR_UUID_128,
                .perm = PERM(WRITE_REQ,ENABLE)|PERM(WRITE_COMMAND,ENABLE)|PERM(WP,NO_AUTH),
                .max_len = UART_SVC_RX_BUF_SIZE,
                .ext_perm = PERM(UUID_LEN,UUID_128)|PERM(RI,ENABLE),
            },
            [UART_SVC_IDX_TX_CHAR] = {
                .uuid = ATT_DECL_CHAR_ARRAY,
                .perm = PERM(RD,ENABLE),
                .max_len = 0,
                .ext_perm = PERM(UUID_LEN,UUID_16),
            },
            [UART_SVC_IDX_TX_VAL] = {

                .uuid = UART_SVC_TX_CHAR_UUID_128,
                .perm = PERM(NTF,ENABLE),
                .max_len = UART_SVC_TX_BUF_SIZE,
                .ext_perm = PERM(UUID_LEN,UUID_128)|PERM(RI,ENABLE),
            },
            [UART_SVC_IDX_TX_NTF_CFG] = {
              .uuid = ATT_DESC_CLIENT_CHAR_CFG_ARRAY,
              .perm = PERM(RD,ENABLE)|PERM(WRITE_REQ,ENABLE),
               .max_len = 0,
               .ext_perm = PERM(UUID_LEN,UUID_16),
            },
    };

**3. 注册service的svc 、database以及回调函数**

.. code:: c

    osapp_svc_helper_t uart_server_svc_helper = 
    {
        .svc_desc = &uart_svc_desc,
        .att_desc = uart_svc_att_db,
        .att_num = UART_SVC_ATT_NUM,
        .read = uart_server_read_req_ind,
        .write = uart_server_write_req_ind,
    };

    通过osapp_add_svc_req_helper将svc注册到协议栈中，svc添加完成后，开始广播。

ble 协议栈和应用协议栈的信息交互
==================================
.. code:: c

    static osapp_msg_handler_table_t osapp_svc_handler_table[] =
    {
        {GATTM_ADD_SVC_RSP,(osapp_msg_handler_t)osapp_add_svc_rsp_helper_handler}, 
        {GATTC_WRITE_REQ_IND,(osapp_msg_handler_t)osapp_write_req_ind_helper_handler},
        {GATTC_READ_REQ_IND,(osapp_msg_handler_t)osapp_read_req_ind_helper_handler},
        {GATTC_ATT_INFO_REQ_IND,(osapp_msg_handler_t)osapp_att_info_req_ind_helper_handler},
        {GAPM_CMP_EVT,(osapp_msg_handler_t)osapp_gapm_cmp_evt_handler},
    };

当和对端设备连接成功后，当对端设备写sever上的特征时，app会收到GATTC_WRITE_REQ_IND消息，然后调用osapp_write_req_ind_helper_handler处理。当server收到串口的数据时，也会将数据nofity到对端设备。