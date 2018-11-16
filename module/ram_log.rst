
RAM LOG
==========


RAMLOG简介
----------------------


RAM LOG 是为了解决在没有连接JLink时候，或者JLink没有持续的读取RTT时候，导致无法查看RTT LOG的问题所设计的。

当没有连接JLink，或者没有持续的读取RTT时候，此时发生了HardFault或者其他原因，需要查看LOG，此时仅仅能够查看最开始的1K的LOG。这样显然不合理。

如果开启了RAMLOG，系统就可以将LOG打在RAM中一个副本，并且循环覆盖。RAM中会保持最新的一部分LOG内容。


RAMLOG 配置
----------------------

RAMLOG默认是关闭的，需要在“Trunk\\plf\\bx_debug\\log\\log.h”进行配置开启。和RAM LOG相关的参数有如下几行

.. code:: c

    #define USE_INTERNAL_LOG        0
    #define INTERNAL_LOG_DEEPTH     10240  //set ram log parameter
    #define INTERNAL_LOG_MAX        1024

各个参数含义如下表

========================== =================================================
 宏名称                      含义
========================== =================================================
USE_INTERNAL_LOG            是否开启RAMLOG，0为关闭，1为开启
INTERNAL_LOG_DEEPTH         RAMLOG缓冲区深度，单位是Byte
INTERNAL_LOG_MAX            单条LOG最大长度，打LOG最大长度不得超过此数据
========================== =================================================


RAMLOG 观察
----------------------

RAMLOG的数据结构如下表所示：

.. code:: c

    typedef struct
    {
        char    *current_pointer;   //sprintf buffer pointer
        uint32_t current_offset;    //offset in internal_log_buf
        uint32_t log_cnt;           //log counter
        uint32_t log_line_cnt;      //new line log counter
    }ram_log_t;
    
    ram_log_t ram_log;


比较重要的参数就是 current_pointer ,current_pointer表示当前RAMLOG的写指针。

如果需要查看RAMLOG，首先需要看 ram_log.current_pointer 的内容，该内容就是RAMLOG最新一次存放的地址。

可以在memory browser输入该地址，在该地址前面的内容就是最新打出来的LOG。

注意LOG为循环覆盖打印，旧的LOG会被新的LOG覆盖掉。



























