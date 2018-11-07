Tools installation
^^^^^^^^^^^^^^^^^^^^^^^^^

编译链接下载调试之前，需要在PC上安装一些基本的软件工具：

1. JLink

    BX2400使用JLink的SWD作为基本调试工具，下载地址https://www.segger.com/downloads/jlink/ 
    
    **!!!注意：必须是JLink V6以上版本，否则Apollo SDK无法正常运行**
    
    安装完成后，可以在设备管理器中找到对应的JLink驱动程序，插在PC上的JLink设备可以正确识别。
    
    .. image:: tools_installation1.png
    
    JLink工具链中使用比较频繁的是JFlash和JLink Commander，分别用于烧写Flash和寄存器级别调试。相关内容在 :doc:`flash_programming_guide_using_jflash` 和 :doc:`debug_env_and_tools` 中有详细介绍
    
#. Keil

    Apollo使用Keil作为基本的调试IDE，用户需要自行安装功能完整的Keil
    
#. nRF Connect

    nRF Connect是Nordic开发的一款手机平台应用，包括Android和iOS都有相应的安装包。nRF Connect可以scan/connect其他设备，或者作为从设备发ADV(需要OS支持)