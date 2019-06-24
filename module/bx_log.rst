
BX_LOG
======

BX_LOG通过SEGGER_RTT将输出日志信息，支持模块化日志标签和编译期日志分级。

分级说明
---------

.. code:: c

    #define LVL_ERROR 1    //错误
    #define LVL_WARN 2     //警告
    #define LVL_INFO 3     //信息
    #define LVL_DBG  4     //调试

当全局与模块日志输出级别均大于等于当前日志级别的时候，当前日志会被输出。

使用示例
---------

#. 在源文件顶部定义该文件日志标签和输出级别

    .. code:: c
        
        #define LOG_TAG              "example"
        #define LOG_LVL              LVL_DBG
        #include "bx_log.h"
    
    LOG_TAG和LOG_LVL定义必须置于#include "bx_log.h"之前。

    若无LOG_TAG定义，默认为"NO TAG"；若无LOG_LVL定义，默认为LVL_DBG。

#. 在函数中使用LOG_X()调用，用法与printf()一致

#. 在bx_log.h中修改GLOBAL_OUTPUT_LVL可以改变全局输出级别

