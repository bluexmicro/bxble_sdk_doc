
新版OTA结构
===============


OTA image 的制作
-------------------

在编译完code之后，会生成两个hex，一个是【osapp_dis_server_with_bootloader.hex】，一个是【osapp_dis_server.hex】（Keil在Objects目录下）。

将【osapp_dis_server.hex】拖动到JFlash，然后点【File】->【Save data file as...】，保存为bin文件即可。在手机中选择该bin文件进行升级。


手机和板子的行为
-------------------------

手机APP在发送BXOTAS_REQ时候，会同时发送bin文件的CRC32以及Length。手机不会去发送整个image_header(256byte)内容。

image_header:长度256byte，有valid flag，crc32，legnth，reversed。

板子收到之后，会先擦除【length+image header】大小的内容。然后再进行后续升级操作。

OTA的镜像存放地址为：config.ini文件中的ota_base所指定。

OTA镜像存放结构为：image header + imnage content (JFlash输出的bin)

板子升级完毕之后会把image_header中valid flag，crc32，legnth这些区域进行填充。

    
bootram行为
------------

新版bootram加入了：ota，wdt，adc

OTA：如果检测到image1位置的valid_flag有效，那么认为刚刚OTA完毕。会进行CRC校验，成功的话将image1拷贝到image0处，然后擦除image1。
这样下次检测image1的valid flag就是无效了。每次启动都是从image0处进行启动。

WDT：如果频繁上下电会抖动，导致启动失败。所以bootram里面在reset跳转之前打开wdt，在用户main关闭wdt。模式为直接复位。

ADC：进入bootram之后会等待电压超过3.0V才进行跳转main函数。可以使用宏关闭。

