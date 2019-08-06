Debug Environment and Tools
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Apollo使用Keil作为调试环境，辅以Segger RTT的方式log输出，在出错时通过HardFault中断进行错误定位。

1. DEBUG环境/debug_flash.ini

   用户的DEBUG环境为Keil MDK：

   .. image:: debug_env_and_tools_img1.png

   在工程配置选项里，有个debug_flash.ini需要注意：

   .. image:: debug_env_and_tools_img2.png

   debug_flash.ini文件是在Keil在debug时的配置文件，Keil会根据debug_flash.ini的内容决定如果下载可执行文件，以及后续的动作。debug_flash.ini的内容是：
   
      LOAD .\\Objects\\Apollo_00.axf NOCODE

   可以看到debug_flash.ini下载可执行文件的指令，且有NOCODE标志，表示只是将将调试信息装载进来，但并不下载任何实质性的数据/指令到开发板，这种调试模式运行的前提，是用户需要事先将生成的hex/bin文件烧写入Flash中，之后用户程序会从Flash启动开始走完整的流程。

#. Keil调试环境

   无论是RAM中直接下载运行程序，还是Flash里启动再跳转到RAM中运行，调试的方法都是一样的。Keil里比较常用的调试方式有暂停/运行，断点，单步调试，查看变量/寄存器/堆栈等。

   a. 暂停/运行

      正在运行的应用程序，可以通过点击暂停/运行的方式查看当前运行的指令位置，继续运行当前应用。

      .. image:: debug_env_and_tools_img3.png

      当暂停应用程序运行时，可以在代码窗口里看到当前C语言指令运行的位置，在反汇编窗口里能看到相对应的汇编指令。

      注意：对于Apollo工程来说，并非所有应用都可以暂停再运行。当应用程序里有一些与定时器相关操作时，暂停程序的运行会导致不可预知的错误。例如，当Apollo与另一个蓝牙设备建立连接后，暂停程序的运行会导致CPU无法及时处理蓝牙Mac相关的中断请求和定时器任务，连接通常都会中断，即便继续运行也无法恢复。

   #. 断点

      对于应用程序调试来说，断点是一种非常有用的调试方法。在Keil里，在程序运行之前/之后，在C语言指令/汇编指令前可以单击设置断点，在程序运行到这里时会自动停止。

      .. image:: debug_env_and_tools_img4.png

      如图所示，程序前灰色区域为可以设置断点的位置。当断点设置成功后，Keil里会显示红色原点，而程序一旦断下，则暂停按钮会自动无效，运行按钮自动生效。继续运行，程序会继续往下走，断点再生效时又会停止。通过在合适的位置设置断点，可以有效验证程序的运行流程，辅助程序结果判断。
      
      在Keil界面里，还可以通过点击调试按钮右侧的断点按钮选择不同的断点操作。这里主要有设置断点/取消断点/暂停所有断点设置/清除所有断点等，用户可以根据自己的调试需求进行操作。

      注意：和暂停/运行操作类似，断点的设置同样有可能导致应用程序出现不可预知的错误，原因相同，所以断点的设置同样要慎重。

   #. 单步调试

      Keil里的可以对C或者汇编进行单步调试：

      .. image:: debug_env_and_tools_img5.png

      C下的单步调试，有四种基本操作。

      a) 逐行调试。每次调试都走C指令的一条语句，如果当前停留在一个函数调用，这种单步操作会进入到函数内部

      #) 执行完当前行的语句。这种单步调试会执行完当前行的C语句，对比逐行调试，如果当前停留在一个函数调用，这种单步调试会直接将函数执行完毕，并停在函数调用后的下一行。

      #) 执行完当前所在的函数并跳出。

      #) 执行到当前的光标出。在代码里光标点击一行还未执行到的代码，这种调试就可以直接在程序运行到这一行时停止。

      在汇编模式下，前面两种调试操作的效果相同，都是运行一条汇编指令，不再有函数的区别。后两种则与C模式下的调试效果相同。

   #. 查看变量/寄存器/堆栈

      在Keil里，可以在View菜单中打开一些查看窗口，用来查看一些即时的值。

      a) 变量

         在Watch窗口里，输入正确的表达式/变量名，即可监测全局变量的数值。

         .. image:: debug_env_and_tools_img6.png

         如上图所示，在代码运行到port.c文件里的时候，同样可以查看modem.c里的全局变量。

      #) 寄存器

         在Registers窗口，可以实时查看CPU通用寄存器的值。这在单步调试和错误定位时往往十分有用。

         .. image:: debug_env_and_tools_img7.png

      #) 堆栈

         在单步调试/断点的调试过程中，可以在Call Stack窗口里查看当前代码的调用堆栈结构。

         .. image:: debug_env_and_tools_img8.png

         在查看堆栈时，还可以同时查看上层调用的局部变量。查看堆栈能辅助调试一些特殊函数的调用，尤其是一些被多次/多处调用的函数。

#. Segger RTT/Log输出

   RTT是Segger在借鉴了SWO和SemiHost的基础上，提供的一种嵌入式交互技术。根据Segger的介绍，RTT可以在不影响CPU运行的前提下，实现微处理器的输入输出。在Apollo里，默认只使用了信息输出这一种功能。

   .. image:: debug_env_and_tools_img9.png

   RTT工作的基本原理，是在RAM内部开辟了一块存储空间，当需要打印信息的时候，通过SDK提供的API，按照要求的格式将信息（比如字符串）填到这块存储区域中。而PC端会有线程，根据具体的配置，通过JLink不断去读这块存储区域里的内容。当读取并解析出软件的输出后，会在PC端输出打印信息。使用RTT的步骤如下： 

   a) 软件接口

      在文件中，需要#include “log.h”，然后按照如下方式调用打印信息的API:

      LOG_INIT();

      LOG(LOG_LVL_INFO,"main\\n");

      LOG_INIT()需要在软件运行起来时调用一次，作为RTT的初始化，之后则不再需要调用。之后打印输出，只需要调用LOG即可，如上所示，可以打印出一个简单的字符串。而如果要打印复杂信息，如下所示：

      LOG(LOG_LVL_INFO,"GAPM profile added indication, id:%d, nb:%d, hdl:%d\n", param->prf_task_id, param->prf_task_nb, param->start_hdl);

      输出具体的数字，需要按照类似printf的调用方式来调用。

   #) 定位RTT存储区域首地址

      在工程下的listings文件夹下，有一个map文件，包含可执行文件的反汇编输出。用文本文件的方式打开这个文件，寻找_segger_rtt，可以看到如下信息：

      .. image:: debug_env_and_tools_img10.png

      后面的十六进制数字为_segger_rtt结构链接后的地址。将这个地址复制，然后将编译链接后的工程调试起来。

   #) 在RTT控制面板配置

      当使用JLink下载可执行文件到目标板并调试起来之后，会在PC端启动一个进程，显示在桌面右下角：

      .. image:: debug_env_and_tools_img11.png

      双击打开后，选择RTT选项配置面板，将之前map里找到的_segger_rtt十六进制地址填入RTT Address内，然后点击Start：

      .. image:: debug_env_and_tools_img12.png

      当RTT成功定位到RAM里的RTT存储区域时，会显示Located RTT control block @ 0xXXXXXXXX，否则定位失效，需要查找具体原因。

      注意：在准备打印RTT的同时，有时会出现RTT locked by other JLink这种提示，原因是PC端又开了类似于JFlash的JLink软件，JLink Control Panel会有多个实例。这时只需要在桌面右下角找到其他Control Panel实例，填入RTT地址即可。

   #) 另一种查看RTT log的方法

      打开J-Link RTT Viewer.exe软件，按如下图配置，将之前map里找到的_segger_rtt十六进制地址填入最下面的Address框中：
      
      .. image:: debug_env_and_tools_img13.png

      然后点击"OK"，若没其他问题，即可在窗口中看到log：

      .. image:: debug_env_and_tools_img14.png

#. Segger RTT 输入

      RTT支持两个方向上的多个通道，向上到主机，向下到目标板，所以我们可以像串口那样往IC端发送信息以方便调试，所需步骤如下：

   a) 修改J-Link RTT Viewer.exe的配置

      在菜单栏上依次选择 “Input” -> "Sending..." -> "Send on Enter"，以及“Input” -> "End of Line..." 根据需要选择行结束符，不需要就选择None。

      .. image:: debug_env_and_tools_img15.png
      
      .. image:: debug_env_and_tools_img16.png

   #) 修改IC端RTT接收buf的大小

      根据自己需要修改SEGGER_RTT_Conf.h文件下的“BUFFER_SIZE_DOWN”，该宏定义了RTT IC端接收PC上位机数据的buf大小，默认是16，这里该成101：
      
      .. image:: debug_env_and_tools_img17.png

   #) 接收数据实现

      定义一个数组buf，然后在程序中利用timer或其他方法，重复地调用SEGGER_RTT_Read(0,buf,sizeof(buf))函数去获取PC端发送到IC端RTT的接收buf里的数据，该函数返回值是获取到的字符长度，若长度为非零，便去解析数据，最终实现数据的接收处理。
   
      .. image:: debug_env_and_tools_img18.png

   #) 使用J-Link RTT Viewer.exe发送数据

      在RTT Viewer窗口的最下面输入框里输入要发送的内容，然后点击“Enter”：
      
      .. image:: debug_env_and_tools_img19.png

      即可在窗口里看到返回的log，包括接收到的数据长度和具体内容。

#. HardFault中断

   HardFault是ARM 处理器中常见且有用的错误中断。当CPU遇到一些硬件异常，例如指令异常，总线出错，空指针等，HardFault中断会触发。在中断触发时，会将导致错误中断产生时的一些CPU寄存器压入当前栈中，同时将EXC_RETURN写入LR,以表明中断类型。关于EXC_RETURN的内容，可以参考ARM Cortex-M权威指南中的相关内容。

   在Apollo中，会有HardFault处理函数来专门处理这一部分内容，具体的软件代码在HardFault_Handler_C函数中。该函数的主要内容就是将HardFault压栈的寄存器dump出来，并通过RTT打印，以帮助用户快速定位异常的触发位置。

   注意：在压栈的寄存器里，最常用最重要的两个通常是PC和LR. PC可以告诉用户异常触发时的PC值，但是如果软件出现指针异常，PC值有可能是一个非法值，基本没有参考意义。这时就需要通过LR来定位软件异常所在。根据CMSIS调用规则，在函数调用时LR通常会被压栈，然后更新为被调用函数返回的下一条指令地址，而当函数退出时，压栈的LR会弹出到PC中。因此LR通常不会像PC一样被异常修改，可以持续可靠的跟踪函数的调用流程。

   注意：这里仅讨论最常见的函数调用情况，LR的值代表的含义需要视具体情况而定，这里需要用户深入研究CMSIS的调用返回机制，以对LR有更深入的理解。

   .. code:: c
      
      void HardFault_Handler_C(uint32_t msp,uint32_t psp,uint32_t lr,
                               uint32_t r4,uint32_t r5,uint32_t r6,
                               uint32_t r7)
      {
          enum{
              R0_INSTACK,
              R1_INSTACK,
              R2_INSTACK,
              R3_INSTACK,
              R12_INSTACK,
              LR_INSTACK,
              PC_INSTACK,
              xPSR_INSTACK,
          };
          uint32_t *sp = 0;
          LOG(LOG_LVL_ERROR, "!!!!!!HardFault Handler is triggered!!!!\r\n");
          LOG(LOG_LVL_ERROR, "Prolog:\r\n");
          LOG(LOG_LVL_ERROR, "R4   = 0x%08x\r\n", r4);
          LOG(LOG_LVL_ERROR, "R5   = 0x%08x\r\n", r5);
          LOG(LOG_LVL_ERROR, "R6   = 0x%08x\r\n", r6);
          LOG(LOG_LVL_ERROR, "R7   = 0x%08x\r\n", r7);
          LOG(LOG_LVL_ERROR, "lr   = 0x%08x\r\n", lr);
          LOG(LOG_LVL_ERROR, "msp  = 0x%08x\r\n", msp);
          LOG(LOG_LVL_ERROR, "psp  = 0x%08x\r\n", psp);
          if(lr==0xfffffffd)
          {
                sp = (uint32_t*)psp;
                LOG(LOG_LVL_ERROR,"PSP Stack Info:\r\n");
          }
          else{
                sp = (uint32_t*)msp;
                LOG(LOG_LVL_ERROR,"MSP Stack Info:\r\n");
          }
          // Try to dump
          LOG(LOG_LVL_ERROR, "R0   = 0x%08x\r\n", sp[R0_INSTACK]);
          LOG(LOG_LVL_ERROR, "R1   = 0x%08x\r\n", sp[R1_INSTACK]);
          LOG(LOG_LVL_ERROR, "R2   = 0x%08x\r\n", sp[R2_INSTACK]);
          LOG(LOG_LVL_ERROR, "R3   = 0x%08x\r\n", sp[R3_INSTACK]);
          LOG(LOG_LVL_ERROR, "R12  = 0x%08x\r\n", sp[R12_INSTACK]);
          LOG(LOG_LVL_ERROR, "LR   = 0x%08x\r\n", sp[LR_INSTACK]);
          LOG(LOG_LVL_ERROR, "PC   = 0x%08x\r\n", sp[PC_INSTACK]);
          LOG(LOG_LVL_ERROR, "xPSR = 0x%08x\r\n", sp[xPSR_INSTACK]);
          return;
      }

   HardFault触发时压栈的通用寄存器包括R0/R1/R2/R3/R12/LR/PC/xPSR，而在某些特殊情况下，错误的跟踪需要其他寄存器的值，因此R4 – psp也会在HardFault里一并打印出来。

   这里有两个LR，第一个LR时HardFault触发时填入LR的EXC_RETURN值，而第二个LR是中断触发时压入栈中的LR寄存器的值。
   

