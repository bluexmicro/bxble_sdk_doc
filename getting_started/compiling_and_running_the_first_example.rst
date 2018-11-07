Compiling and running a first example
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在开发自己的应用程序之前，需要在PC上验证是否已经完整安装所需要的工具链。安装工具链所需要的软件和步骤，请参考文档 :doc:`tools_installation` 

工具链安装完备之后，可以打开Apollo SDK里的Keil工程文件。

.. image:: compiling_and_running_the_first_example_img1.png

初次打开的Apollo Keil工程如下：

.. image:: compiling_and_running_the_first_example_img2.png

对于Apollo调试，通常需要两个基本步骤：编译链接生成hex和运行调试hex

1. 编译链接生成hex

   在Apollo SDK中，有若干APP Demo，初次上手的用户可以选择其中一个，编译链接。步骤如下：

   a. 打开bx_sys_config.h，配置本地蓝牙地址

      #define BX_DEV_ADDR {0x91,0x22,0x33,0x44,0x55,0x66}

      该头文件里其余的配置项保持默认值。

   #. 打开osapp_config.h，选择一个APP的宏。推荐选择OSAPP_DIS_SERVER，在这个应用里，开发板会发带有广播数据的广播包，而扫描者（例如手机）可以扫描到该广播包，并建立连接。

      .. image:: compiling_and_running_the_first_example_img3.png

      **!!!注意：这里的宏必须使能一个，且只能使能一个**

   #. 编译链接

      .. image:: compiling_and_running_the_first_example_img4.png

      完成之后，可以在Build Output里看到结果输出：

      .. image:: compiling_and_running_the_first_example_img5.png

   #. 配置Debug选项：

      .. image:: compiling_and_running_the_first_example_img6.png

      Debug选项中，确保调试初始化文件为debug_flash.ini

#. 运行调试hex

    在Keil目录下找到生成的hex文件，并下载该文件到Flash中，详细步骤可以参考文档 :doc:`flash_programming_guide_using_jflash` 

   #. hex运行起来后，可以通过手机或者抓包器看到空中广播包：

      .. image:: compiling_and_running_the_first_example_img8.png

      **!!!注意：避免在同一环境中运行两个地址完全相同的蓝牙应用**

   #. 可以在Keil环境里暂停/运行，设置断点，单步调试等。具体细节参见文档 :doc:`debug_env_and_tools`
      
      **!!!注意：在Apollo里，某些特定状态下，Keil里无法执行上面的调试步骤，例如当IC在反复睡醒时，此时CPU处于关电状态，JLink无法连接到IC。用户可以使能bx_sys_config.h中的宏DEBUGGER_ATTACHED，在此条件下编译生成的hex会跳过深睡眠的处理，保证在调试时用户的目标板可以连接到JLink**     
