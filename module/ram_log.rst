
RAM LOG
==========


RAMLOG简介
----------------------


RAM LOG 是为了解决在没有连接JLink时候，或者JLink没有持续的读取RTT时候，导致无法查看RTT LOG的问题所设计的。

当没有连接JLink，或者没有持续的读取RTT时候，此时发生了HardFault或者其他原因，需要查看LOG，此时仅仅能够查看最开始的1K的LOG。这样显然不合理。

如果开启了RAMLOG，系统就可以将LOG打在RAM中一个副本，并且循环覆盖。RAM中会保持最新的一部分LOG内容。


RAMLOG 配置
----------------------

RAMLOG默认是关闭的，需要在“Trunk\\plf\\bx_debug\\bx_log_def.h”进行配置开启。

.. code:: c

    #define LOG_BACKEND (JLINK_RTT | RAM_LOG)  //使用 JLINK_RTT + RAM_LOG



和RAM LOG相关的参数有如下几行,“Trunk\\plf\\bx_debug\\log\\ram_log.h”中配置

INTERNAL_LOG_MAX_SIZE 为RAMLOG总的缓冲区大小

INTERNAL_LOG_SINGLE_SIZE 为每一条LOG打印函数最大能够承受的数量，单条LOG超过本数值会导致内存溢出。


.. code:: c

    //set ram log parameter
    #define INTERNAL_LOG_MAX_SIZE           10240  //max usage of ram log size
    #define INTERNAL_LOG_SINGLE_SIZE        1024   //max size per item



RAMLOG 观察
----------------------

RAMLOG的数据结构如下表所示：

.. code:: c

    uint8_t  ram_log_buf[INTERNAL_LOG_MAX_SIZE];
    uint8_t *ram_log_pointer = ram_log_buf;



首先需要找到 ram_log_buf 数组的地址，该数组保存了所有的RAM LOG。

其次需要找到 ram_log_pointer ，该数值表示了最后一条RAM LOG写到的位置。 注意如果LOG数据比较多，会循环覆盖掉最旧的。

如果没有循环覆盖，则LOG内容为：{&ram_log_buf[0] -> ram_log_pointer}

如果存在循环覆盖，则最新LOG内容为： {ram_log_pointer -> &ram_log_buf[INTERNAL_LOG_MAX_SIZE]} 以及 {&ram_log_buf[0] -> ram_log_pointer}
















