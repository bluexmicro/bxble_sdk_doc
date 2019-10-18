=================
UART
=================
"""""""""""""""""
特性
"""""""""""""""""

* 最大波特率：2M

* 奇、偶校验

* 非阻塞传输（中断模式、DMA模式）

* 流控：自动流控、手动流控 （仅对UART0，UART1没有流控管脚）

"""""""""""""""""
使用
"""""""""""""""""
             
1. 设备实例化

.. code:: c

    #include "app_uart.h"   
    app_uart_inst_t uart0 = UART_INSTANCE(0);
   
2. 参数设置，设备初始化

.. code:: c

    uart0.param.baud_rate = UART_BAUDRATE_115200;   
    uart0.param.rx_pin_no = 13;
    uart0.param.tx_pin_no = 12;
    uart0.param.rts_pin_no = 20;
    uart0.param.cts_pin_no = 21;
    uart0.param.parity_en = 0;
    uart0.param.flow_control_en = 1;
    uart0.param.auto_flow_control = 0;
    uart0.param.tx_dma = 1;
    uart0.param.rx_dma = 1;
    app_uart_init(&uart0.inst);
    
3. 数据接收、发送
    
.. code:: c
 
    app_uart_read(&uart0.inst,bufptr,size,callback,dummy);
    app_uart_write(&uart0.inst,bufptr,size,callback,dummy);

收发完成时，callback在中断处理上下文中被调用。

对于同一个UART实例，必须在一次传输完成后，才可以发起下一次传输，即第一次发起app_uart_read后，必须在收发完成，callback被调用的时刻之后，才可以第二次发起app_uart_read。app_uart_write同理。

bufptr不能指向栈上的数据区，且必须在收发完成之后，即callback被调用的时刻之后，才能释放。

4. 读取消

.. code:: c

    uint32_t remaining_length = app_uart_read_cancel(&uart0.inst);
    
返回值 == 注册读操作时期望读取的长度 - 调用app_uart_read_cancel时实际收到的数据长度。 
        
5. 流控

自动流控无需代码参与。

手动流控：

.. code:: c

    app_uart_flow_on(&uart0.inst);                // 拉低RTS
    bool is_de_asserted = app_uart_flow_off(&uart0.inst); //拉高RTS
    
6. 设备卸载

.. code:: c
    
    app_uart_uninit(&uart0.inst);
    
设备卸载需要在收发完成之后进行，否则可能造成非预期的情况。如读操作迟迟没有完成，需要先取消读操作，再进行设备卸载。