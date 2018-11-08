Flash programming guide using JFlash
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Apollo支持通过JFlash读写Flash。在操作之前，需要下载JLink(V6版本以上，`JLink下载`_)并安装，之后按照如下步骤操作：
    
    .. _JLink下载: https://www.segger.com/downloads/jlink/

1. 准备好DVK/EVK，通过SWD接口将JLink连接到板子上，JLink另一端插入PC，在PC上能正确识别JLink设备

#. 上电之前，在DVK/EVK上，将P16拉高，再根据IO电压配置确定P23拉高或拉低

    - 这一步的目的，是为了使IC上电时从Uart启动，而不是Flash。如果在烧写Flash之前，Flash里已经有了可执行文件，而且有睡醒功能，此时一旦从Flash上电，软件会迅速从Flash boot并进入用户程序，而JLink在深睡眠下无法顺利连接到板子，从而导致JFlash连接失败。
 
#. 打开JFlash应用程序

    .. image:: flash_programming_guide_using_jflash_img0.png

    - 第一次打开JFlash，需要通过other选择.jflash文件，在Apollo SDK的tools\prog_tool目录下，用户需要根据Flash的工作电压选择对应的.jflash文件
 
    - 选择对应的.jflash文件后，打开，在JFlash的Project对话框中能看到该.jflash内部一些配置，比如SWD速度配置，CPU类型，大小端类型，RAM地址等。这些配置在JFlash中已经默认配置完成，用户通常不需要手动更改
 
#. 打开.jflash工程文件之后，点击File菜单下的Open data file，选择将要烧写的flash.hex；或者将flash.hex直接拖入JFlash界面

#. 点击Target菜单下的Production Programming，JFlash会自动执行connect/erase/write/verify，成功后会显示烧写成功对话框

#. 在EVK/DVK拉低P16到地，根据Flash电压配置P23，之后重新上电。

#. 说明：

    - P16拉高：表示Boot from Uart
    
    - P16拉低：表示Boot from Flash
    
    - P23拉高：表示Boot设备工作电压为3.3V
    
    - P23拉低：表示Boot设备工作电压为1.8V
    