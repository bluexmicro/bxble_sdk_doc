Link Script & Keil SDK Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在Keil下编译链接Apollo SDK，需要熟悉Keil下的各项工程配置。

1.  链接脚本

    在Target Options的Linker里，可以配置链接脚本：

    .. image:: link_script_and_keil_sdk_cfg_img1.png

    点击右侧的Edit按钮，可以在Keil里直接编辑链接脚本文件。首先，用户需要了解链接脚本的基本语法结构：

    .. image:: link_script_and_keil_sdk_cfg_img2.png

    ARM的链接脚本的主要构成是Load Section和Exec Section。考虑一个典型的Flash上运行软件的例子：IC从Flash启动，之后需要要把所有的数据和一部分代码搬移到RAM中（例如Flash驱动，中断向量表，任务调度函数等，没办法放在Flash中），此时IC启动后ARM默认的__main()需要知道代码搬移的起始地址和目的地址，这就是通过链接脚本指定的。起始地址就是Load Section，而目标地址就是Exec Section。 Apollo的链接脚本范例如下:

    ..  code:: c

        LR_RAM1 RAM_BASE ALIGN 0x80{
            ISR_VECTOR +0 
            {
                *(RESET, +First)
                *(jump_table_area)
            }
            CODE_DATA_BSS +0
            {
                *(+RO)
                *(+RW)
                *(+ZI)
            }
            HEAP_STACK +0 UNINIT
            {
                *(HEAP)
                *(STACK)
            } 
            ScatterAssert(ImageLimit(HEAP_STACK)<=EM_START)
        }
        
        LR_RAM3 AlignExpr(EM_END,0x4) ALIGN 0x4
        {
            RWIP_ENV +0
            {
                rwip.o(+ZI)
            }
            BOOT_PARAMS +0 UNINIT
            {
                *(boot_tunnel)
            }
            RAM_UNLOADED +0 EMPTY 12
            {
            }
            ScatterAssert(ImageLimit(RAM_UNLOADED)<=ROM_CODE_ZI_BASE)
        }
        
        LR_RAM2 CACHE_BASE
        {
            NVDS_AREA +0 PADVALUE 0xffffffff
            {
                *(nvds_area)
            }
        }

    这里主要包括3个Load Section，每个Load Section又包含了数目不同的Exec Section。以第一个Load Section为例，LR_RAM1表示Load Section的名字，RAM_BASE表示Load Section的起始地址，ALIGN 0x80则表示起始地址必须是0x80对齐。ISR_VECTOR是第一个Exec Section的名字，+0表示起始地址相对之前的Load Section LR_RAM1的偏移是0，也就是说地址相同。*(RESET, +First)和*(jump_table_area)则表示该Exec Section的两个输入，第一个输入是在汇编里定义的段，具体参考源文件startup_apollo_00.s，第二个输入是C源代码里定义的jump_table_area段。第二种很常用，其在C代码中的写法是：

    ..  code:: c

        #define JUMP_TABLE_SECTION __attribute__((section("jump_table_area")))
        void *const jump_table[JUMP_TABLE_SIZE] JUMP_TABLE_SECTION;

    这样在链接的时候，jump_table这个数组就会被链接入ISR_VECTOR这个Exec Section所在的内存区域内。

    关于上面这段链接脚本的更多解释如下：

    RO表示只读，包括RO-DATA和RO-CODE，分别表示全局的只读数据区和代码区。RW表示可读可写的全局数据区，且初始值不为0. ZI表示初始值为0，或没有初始值的全局变量。UINIT表示该段不需要初始化，AlignExpr(EM_END,0x4)表示以EM_END为基准值计算出一个4字节对齐的地址值。ScatterAssert是链接脚本语言里的断言语句，通常用来对某些段的大小进行检查，防止越界。PADVALUE +0xffffffff表示该段的初始值为0xffffffff.

    关于链接脚本更详细的信息，可以参考ARM官方PDF文档armlink_user_guide.pdf

#.  工程管理/编译链接

    a.  工程文件管理

        点击图中按钮可以管理工程内所有.c/.s文件：

        .. image:: link_script_and_keil_sdk_cfg_img3.png

        之后在打开的对话框中可以增减Targets/Groups/Files:

        .. image:: link_script_and_keil_sdk_cfg_img4.png

        关于Group/Files内容，Apollo更新后与图片显示不同，具体以SDK内容为准。

    #.  工程管理选项

        -   Device选择

            .. image:: link_script_and_keil_sdk_cfg_img5.png

            Apollo内的MCU为Cortex-M0+。

        -   Target配置

            .. image:: link_script_and_keil_sdk_cfg_img6.png

            Target里主要配置ROM和RAM的地址以及大小。关于RAM中的具体内容，请参考文档Memory Distribution. 另外，Apollo使用MicroLIB，因此相应的选项也需要选中。

        -   Output选项

            .. image:: link_script_and_keil_sdk_cfg_img7.png

            Output选项里，主要配置输出的文件信息。Apollo里的配置，尽可能输出更多的信息方便调试。Apollo里不需要hex文件，用户如果需要可以选择生成。

        -   Listing选项

            .. image:: link_script_and_keil_sdk_cfg_img8.png

            Listing里主要需要配置生成map文件，以及map文件里的内容。Map文件是生成可执行文件后，通过反汇编生成的包含有调试信息的文本文件。在具体的调试过程中，反汇编生成的Map或者asm文件是重要的参考资料。

        -   User选项

            .. image:: link_script_and_keil_sdk_cfg_img9.png

            User选项里，主要配置了在生成可执行文件后，需要执行的自定义命令，这里主要有2个：#1是一个批处理文件，其内容包括用fromelf命令从axf文件中生成bin文件和反汇编asm文件；#2是利用自定义的bin_merge.exe生成flash.bin文件。用户最终需要生成一个可以烧写入Flash里的bin文件，这个文件就是通过这里的After Build命令生成。

        -   C/C++选项

            .. image:: link_script_and_keil_sdk_cfg_img10.png

            C/C++选项里，最需要关注的是配置头文件路径。Apollo已经默认配置好需要的路径，用户如果增加了自定义的头文件目录，需要在这里添加到列表中，否则编译时会报错。其余的配置保持默认值即可。

        -   Asm选项

            该选项里没有具体的配置内容，除非需要增加自定义汇编文件，否则用户不需要关注这里太多。

        -   Linker选项

            .. image:: link_script_and_keil_sdk_cfg_img11.png

            链接选项里主要是配置链接脚本和链接选项。

            关于链接脚本的具体内容，可以参考文档Link scripts/Keil SDK Config

            Apollo包含ROM，因此在用户程序链接的时候，需要引用符号表，也就是rom_syms_armcc.txt，这个文件已经包含在工程目录中，用户一般不需要关心。

        -   Debug选项

            .. image:: link_script_and_keil_sdk_cfg_img12.png

            Debug选项里，左半边面板属于Simulator下的配置选项，用户不需要关心。右侧的选项中，需要注意的是初始化文件。Apollo SDK里有两个ini文件：debug.ini和debug_flash.ini，分别代表软件代码直接在RAM中调试，和从Flash启动再到跳转到RAM中运行的选项。关于两种调试方式的选择，通常建议如下：

            -   直接下载到RAM中调试：在用户开发前中期，BLE的主要功能未调试完成，不太关注Flash相关的操作时。此时配置初始化文件debug.ini

            -   Flash启动到RAM中运行调试：在用户开发后期，当BLE主要功能已经调试完毕，需要从启动开始调试，或者开始关注Flash相关操作时。此时配置初始化文件debug_flash.ini
            另外，在选择J-LINK/J-TRACE Cortex选项后，点击右侧setting选项里需要将调试接口配置为SWD：

            .. image:: link_script_and_keil_sdk_cfg_img13.png

            Port选项里需要选择SW，而速度推荐1MHz

            Trace选项中的内容不需要关心。Flash DownLoad中取消所有配置：

            .. image:: link_script_and_keil_sdk_cfg_img14.png

            Flash擦写选项主要在Utilities下配置。

        -   Utilities选项

            Utilities里默认配置外部工具擦写Flash。

            .. image:: link_script_and_keil_sdk_cfg_img15.png
            
            这里的prog.bat定位于\tools\prog_tool文件夹，但由于不同的Flash电压对应不同的配置文件，因此需要用户根据使用的Flash电压，手动修改具体的bat文件为prog_1v8.bat/prog_3v3.bat.
            
            .. image:: link_script_and_keil_sdk_cfg_img16.png
            
            之后在Keil主界面运行Flash Download，可以弹出JFlash界面。
            
            .. image:: link_script_and_keil_sdk_cfg_img17.png
            
            关于JFlash擦写工具，可以参考文档JFlash Tools.
            
            ``!!! 注意：``
            
            -    JLink安装软件版本需要是V6以上；
            
            -    JLink需要安装在系统盘的Program Files (x86)文件夹下，否则bat文件无法定位到JFlash可执行文件的具体位置。

    #.  编译链接

        Keil的编译有三种选择：

        .. image:: link_script_and_keil_sdk_cfg_img18.png

        第一个按钮表示编译当前文件，第二个按钮表示编译整个工程，第三个按钮表示重新编译整个工程。后两个的差别在于，当工程已经编译过了，第二种编译方式只会编译修改过的文件，而第三种则是将所有文件都全部重新编译。
        
        编译链接成功时，可以看到类似如下信息输出：

        .. image:: link_script_and_keil_sdk_cfg_img19.png

        如何编译链接运行一个范例程序，可以参考文档Compiling and running example



