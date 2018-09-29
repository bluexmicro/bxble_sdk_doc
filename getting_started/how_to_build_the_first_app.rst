How to build the first BLE & peripheral application
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

BX2400 SDK面向用户提供了基于FreeRTOS的开发接口，在此基础上可以简单而快速的构建自己的第一个BLE/外设应用程序，而无须关心BLE和外设的实现细节。

构建第一个BLE应用程序
---------------------

1. 用户与协议栈交互原理

   基于FreeRTOS的软件体系，其核心为两个任务：BLE任务和APP任务。操作系统初始化时会有两个队列，BLE任务和APP任务通过这两个队列进行消息收发和处理。细节可以参考文档Folders structure & Software architecture.

#. 用户需要关注的核心：APP任务

   基于FreeRTOS，BX2400 SDK提供了简便易用的APP任务使用接口。用户需要做的事情绝大多数被抽象为一件事：定义并实现一个handler_table. 下面以osapp_dis_server为例详细描述：

   .. image:: how_to_build_the_first_app_img1.png

   上图为osapp_dis_server里的handler_table. 从handler_table的定义可以看到，这是一个osapp_msg_handler_table_t类型的数组，而osapp_msg_handler_table_t则是一个结构体：

   .. image:: how_to_build_the_first_app_img2.png

   所以handler_table是一个数组，其中每一个元素都包含一个消息id，和一个对应的处理函数指针。用户的BLE应用程序中，考虑的主要是需要和协议栈交互哪些消息，以及每条消息所对应的处理函数。

#. 构建APP任务的通用步骤

   1) 协议栈发送的第一条消息：GAPM_DEVICE_READY_IND

      这是BLE协议栈在初始化完成后，通知APP任务的第一条消息。当APP收到这条消息时，意味着协议栈相关的软硬件初始化已经完成，等待APP任务向下发送消息。此时比较通用的做法是向协议栈发送一条reset命令，使协议栈的各个状态回复到reset值。在osapp_dis_server里的做法是调用一个osapp_reset函数：

      .. image:: how_to_build_the_first_app_img3.png

      可以看到osapp_reset函数主要负责向协议栈发送了一条GAPM_RESET_CMD，调用的接口是AHI_MSG_ALLOC。关于AHI_MSG_ALLOC，是一个分配AHI命令的宏，其第一个参数是消息ID，第二个参数是消息接收者ID，第三个参数是该消息的参数类型。每一条从APP任务发送到协议栈的消息，都是通过对这个宏的调用，之后填入正确的参数发送到协议栈的。关于消息ID、接收者ID以及消息的参数类型，在相对应的头文件中找到其定义。

      APP任务发送AHI消息到协议栈的接口是osapp_msg_build_send函数，两个参数分别为命令指针，和命令的大小。

   #) 协议栈收到GAPM_RESET_CMD消息后，会进行协议栈的reset，完成之后，会给APP发送一条GAPM_CMP_EVT消息。APP任务处理该任务的函数是osapp_gapm_cmp_evt_handler。GAPM_CMP_EVT是一条通用的协议栈处理消息，APP任务收到这个消息表示协议栈处理完成相关的命令。当这个消息上来之后，在其参数里可以获取对应的动作：
      
      .. image:: how_to_build_the_first_app_img4.png

      可以看到，在收到GAPM_RESET后，osapp_dis_server的处理是向协议栈发送了一条GAPM_SET_DEV_CONFIG_CMD的消息。封装该消息的接口函数是osapp_set_dev_config，包括三个参数，设备角色，地址类型和配对方式。参数的相关含义可以在相关头文件中找到。可以看到osapp_dis_server将设备定义为slave角色，公开地址和传统配对模式。 

   #) 协议栈收到GAPM_SET_DEV_CONFIG_CMD后，会和GAPM_RESET_CMD消息一样，向APP任务发送一条GAPM_CMP_EVT消息。当APP任务收到该消息，表明之前系统设定的动作已经完成。此时通用的处理是，根据应用的具体需求，通过GAPM_PROFILE_TASK_ADD_CMD消息，添加所需的profile。 osapp_dis_server实现了一个osapp_add_dis_server_task：
      
      .. image:: how_to_build_the_first_app_img5.png

      在osapp_add_dis_server_task里，调用了AHI_MSG_ALLOC_DYN宏来实现消息分配。AHI_MSG_ALLOC_DYN和AHI_MSG_ALLOC的功能基本相同，唯一的区别在于DYN分配的消息里，除了消息本身约定的参数外，还包含额外的自定义参数。在发送的消息gapm_profile_task_add_cmd里，prf_task_id对应着profile的ID。根据用户具体需求，这里需要填入不同的Profile ID。消息构建完成后，调用osapp_msg_build_send函数发送到协议栈。

   #) 协议栈收到GAPM_PROFILE_TASK_ADD_CMD消息后，会在BLE Kernel里添加一个内核任务，并参与调度。而在完成这一动作后，会向APP任务发送一条GAPM_CMP_EVT消息。与此同时，协议栈会向添加APP任务发送一条GAPM_PROFILE_ADDED_IND消息，在osapp_dis_server里对应的处理函数是osapp_gapm_profile_added_ind_handler：

      .. image:: how_to_build_the_first_app_img7.png
   
      通用的处理是，如果还有添加的Profile，则继续通过GAPM_PROFILE_TASK_ADD_CMD添加，否则根据之前用户定义的设备角色，可以开始相关的BLE动作。osapp_dis_server定义的角色是slave，所以这里开始请求协议栈发送广播包，调用的函数是osapp_start_advertising：
      
      .. image:: how_to_build_the_first_app_img6.png

      发送广播包需要向协议栈发送的消息是GAPM_START_ADVERTISE_CMD，这里有几个广播包配置参数需要注意：

      - op.addr_src: 地址类型。具体可以配置为哪些地址类型，需要参考蓝牙官方协议文档。

      - channel_map: 广播报发送的通道。根据蓝牙协议，广播包可以在37/38/39三个通道上发送，这里通常定义为0x7，低三位分别代表三个通道。

      - intv_min & intv_max: 广播包间隔最小值和最大值。具体对应的时间是该宏的值乘以625us，例如intv_min = intv_max = 32时，对应的实际广播包长度是32*0.625 = 20ms。

      - op.code: 广播包类型，通常表示广播包是否是定向广播包，是否具备可连接性等。具体含义需要参考蓝牙官方文档和相关头文件。

      - info.host.mode: 发送广播包的模式，通常意味着广播包可以被scan到的对象范围。具体含义需要参考蓝牙官方文档和相关头文件。

      - info.host.adv_data_len: 广播包的长度。这里是通过调用宏ADV_DATA_PACK来计算具体的广播包长度数据。根据蓝牙协议，广播包具有几个字节的包头，包含广播包类型，载荷长度等信息，而ADV_DATA_PACK的作用是根据用户程序配置的广播包内容信息自动填入符合要求的包头，同时返回包含包头在内的广播包长度信息。

   #) 协议栈收到GAPM_START_ADVERTISE_CMD消息后，会首先判断用户发送的参数是否合法。判断通过后，协议栈会向上层发送一条GAPM_CMP_EVT，通知协议栈广播包即将发出。到此为止，配置协议栈发送广播包的动作全部完成。

   #) 用户程序如果需要和其他的device建立连接，往往需要处理BLE的建立连接消息。作为slave角色的osapp_dis_server，在发送广播包的时候，如果对方设备请求建立连接，协议栈会向APP任务发送一条GAPC_CONNECTION_REQ_IND消息，APP任务收到后会通过osapp_gapc_conn_req_ind_handler函数处理该事件。osapp_dis_server的操作是向协议栈回送一条GAPC_CONNECTION_CFM消息确认连接建立。

构建第一个外设应用程序
-----------------------
      

