Folders structure & Software architecture
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1. 目录结构
	 
	 Apollo SDK的根目录结构如下:
	 
   .. image:: folder_structure_software_architecture_img1.png

   下面将依次介绍每个文件夹中的内容和结构。
   
   1. freertos
   
      .. image:: folder_structure_software_architecture_img2.png

      freertos下主要包含了FreeRTOS的源码，以及开发的一些demo。
   
      app: 包含Apollo的一些demo。例如osapp_dis_server/osapp_dis_client是dis server/client的应用，分别在板子上跑起来手机可以对联；osapp_uart_server是串口透传的应用。用户自己开发的BLE相关的应用程序，通常也在这个目录下。

   #. ip
   
      .. image:: folder_structure_software_architecture_img3.png

      ip下是关于ceva IP的头文件和部分源码。关于BLE的协议栈源码，已经固话在ROM里，用户无需关心具体实现细节。
      
   #. KeilPro
      
      KeilPro文件夹下全部是Keil相关的配置文件，没有源代码。
      
   #. modules
   
      .. image:: folder_structure_software_architecture_img4.png   

      modules下包含一些系统相关的功能模块。用户可能需要关心的有：
      
      1. ecc_p256

				 这里包含ECC(Elliptic Curve Cryptography)的软件实现源码，以及硬件调用接口。用户应用中如果包含自定义的加密，其中涉及ECC算法，则需要了解此处内容。

      #. nvds

				 NVDS(Non-Volatile Data Storage)给用户提供了一个在Flash和RAM上存储自定义数据的接口。除了设备名称/蓝牙地址/睡醒配置等，用户自定义的应用相关的数据也可以存储在NVDS中。关于NVDS的细节，参见文档 |NVDS spec|.
				 
   #. plf/bx_debug
				 
			.. image:: folder_structure_software_architecture_img5.png				 

      plf/debug文件夹下需要用户了解的是log文件夹，其中包含了Segger RTT的源码，已经Apollo对用户封装的打印信息接口。关于RTT的具体内容，参见文档 |Debug Environment & Tools|.
			
   #. plf/apollo_00
	 
	    .. image:: folder_structure_software_architecture_img6.png
	    
	    plf/apollo_00下包含的Apollo系统的主要功能。其中用户需要关心的是main文件夹，这里包含了arch_main.c文件，其中有main函数的入口。在main调用之前的系统初始化函数system_init函数也同样在arch_main.c中。
	    
	    关于其他文件夹，这里也做一个简单的介绍，希望帮助有深入了解需求的用户。
			
      1. bootloader
      
         bootloader下包含了boot_rom/boot_ram/boot_ram_download，分别表示ROM中的boot工程，RAM中的boot工程，以及从Uart boot的工程。Apollo的Boot包含从ROM到RAM的两部分，详见文档 |Boot introduction|.
      
      #. config
      
         链接时需要用到的ROM符号表。
         
      #. arch/boot
      
         boot时的startup汇编文件。

      #. build
      
         ble相关的头文件。
         
      #. jump_table

         ROM调用RAM函数的跳转文件。

      #. patch_list
      
         Apollo里使用的Patch文件。
         
      #. peripheral_integration
      
         外设驱动中与平台相关的文件。这里的外设和Apollo平台紧密相关，但和用户调用没有直接关系。

      #. sleep
      
         睡眠唤醒文件。

      #. system_integration
      
         系统功能性配置文件，比如时钟配置/modem/patch/软中断等，用户基本不需要关心。

      #. peripheral
      
         系统驱动靠近用户层面的源代码。
         
   #. bx_config.h等头文件

      这些头文件有一部分需要用户关心。一些系统和应用相关的配置都包含在这些文件中。具体信息参见文档 |Top header files configuration|.
      
#. 软件架构

   Apollo集成了FreeRTOS，可以以带FreeRTOS/不带FreeRTOS两种方式支持用户应用开发。

   .. image:: folder_structure_software_architecture_img7.png

   1. w/o FreeRTOS
   
      没有FreeRTOS的情况下，系统的基本框架是一个while循环，所有任务，包括协议栈上层的Host，下层的Linklayer，Host与Linklayer之间的HCI，Profile以及用户应用都是同等优先级，事件处理无先后之分。

      在没有FreeRTOS的前提下，一些功能需要额外实现，例如定时器，malloc/free堆管理等。同时考虑到任务之间无优先级区分，所以推荐用户在FreeRTOS的框架下开发自己的应用程序。
   
   #. with FreeRTOS
   
      FreeRTOS是一种世界范围内广泛使用的嵌入式实时操作系统，十几年间与众多世界知名IC公司合作，使得从某种角度而言，FreeRTOS已经成为一种嵌入式微处理器的潜在标准。更多FreeRTOS的信息，可以参考 (https://www.freertos.org/)
      
      基于FreeRTOS的软件架构由如下任务：

      1. BLE任务
      
         BLE任务包含了所有蓝牙相关的事件处理，通常具有最高优先级。这样处理的好处，是使得BLE相关的事件到来时，会第一时间处理到。例如，在BLE收到空中数据后，会通过中断给BLE任务一个通知，此时不管其他任务状态如何，BLE任务都会被立刻处理到，从而保证蓝牙数据收发不会被延迟。
      
      #. APP任务
      
         APP任务通常包含了用户的应用程序处理，具有次高优先级。用户程序能够正常运行的前提是BLE数据收发正常，因此用户任务通常优先级低于BLE任务。另一方面，用户的程序也希望延迟小，可以在最短的时间内处理到，所以往往高于Idle任务。
      
      #. Idle任务

         Idle任务包含一些优先级最低的事件处理，例如定时器计时，以及WFI。Idle任务具有最低优先级，只有在BLE任务和APP任务都无事件要处理的时候，Idle任务才能获取CPU的处理权限。这样即便定时器事件没有及时处理，之后进入Idle后也有容错处理。同时进入WFI本身就要求软件内部其他任务都处于空闲状态。
         
      对于大多数用户而言，需要关心的是APP任务的构建，以及APP任务与BLE任务之间的通信接口，保证在最短的事件内快速构建一个BLE应用。关于构建BLE应用的详情，请参考文档 |How to build the first BLE application/peripheral application|.      

#. 软件流程

   在应用场景下，Apollo软件通常是从Flash启动，之后分两步将Flash里的软件代码搬移到RAM中，并跳转到RAM中运行。这里只讨论软件被搬移到RAM中后运行的流程，关于Boot流程，请参考文档 |Boot Procedure|.

   当软件跳转到RAM中运行后，大致经历了如下几个步骤：

   #. Reset_handler() in startup_apollo_00.s

      RAM中运行软件的入口就是Reset_handler()，其具体实现是在startup_apollo_00.s中。Reset_handler()里做了两件事：一是调用System_init()，这个函数在arch_main.c中，对于用户来说需要关心的是这里做了中断向量表重定位，从默认的ROM重新设置的RAM中；二是调用__main(),这是ARM里负责跳转到main()的入口函数。在调用main()之前，__main()会根据链接脚本生成的记录，初始化所有RW段，清零所有ZI段，并负责处理Load Section和Exec Section地址不同的段的代码搬移。在全局数据/代码都准备就绪之后，__main()会自动跳转到main()中。关于链接脚本，详情参见文档 |Link Script & Keil SDK Config|.

   #. main() in arch_main.c
      
      对于用户来说，通常关心的软件处理基本是从main()开始。在main()里，主要做了如下工作：
      
      1. 系统初始化，包括AHB/APB时钟配置，RF寄存器初始化，默认pinshare配置，外设初始化等。
      
      2. BLE相关的初始化工作，modem初始化，modem参数校准，patch初始化，NVDS初始化，以及BLE IP的初始化等。
      
      3. 全局中断使能。没有FreeRTOS则直接进入While循环，有FreeRTOS的情况下会通过rtos_task_init()初始化所有任务，并自动开始调度。自此，Apollo的软件全部启动完成，正式开始执行BLE和用户应用程序以及相关外设。

   #. FreeRTOS任务与内部消息处理
   
      基于FreeRTOS的软件体系，其核心为两个任务：BLE任务和APP任务。操作系统初始化时会有两个队列，BLE任务和APP任务通过这两个队列进行消息收发和处理。整个系统的消息处理流程如下：

      .. image:: folder_structure_software_architecture_img8.png

      1. 表示应用程序通过APP任务给BLE任务发送上层命令，例如开始发送adv包，和某个设备建立连接，向对端传送数据等。协议栈里向上提供AHI(App Host Interface)接口，应用程序向协议栈发送命令和接收数据时都是AHI角色。
      
      #. BLE任务收到APP任务发送过来的命令后，会通过os_bridge和软中断接口，调用内部模块解析APP命令。解析之后会根据命令内容将命令转发给协议栈内部不同的任务。这里的任务不同于APP和BLE任务，它们是协议栈内部的核心任务，用户不可见。
      
      #. 协议栈内部任务收到消息后，会根据消息的具体内容去配置BLE硬件，使其按照用户的要求执行特定的行为，例如发送adv包，扫描对方的adv包并建立连接，发送数据等。
      
      #. 在D处理完成后，协议栈会向之前命令的发送者（AHI角色）发送一条完成事件。
      
      #. 而当底层BLE硬件在某些特定情况下，例如收到数据，会通过中断处理函数给BLE 协议栈发送一条消息，协议栈通过内部处理，最终发送给APP任务（步骤D），或者内部处理。

      在上面的流程中，只有A和D是用户可见流程，操作的接口是FreeRTOS的消息队列。其余处理都是协议栈、Kernel和BLE硬件内部处理，用户不可见。
