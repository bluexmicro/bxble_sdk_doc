=================
DMAC
=================
"""""""""""""""""
特性
"""""""""""""""""

* 多通道并行（Apollo 6通道）

* 中断传输、阻塞传输

* 传输模式：外设-内存，内存-内存

* 外设-内存传输使用硬件握手

"""""""""""""""""
使用
"""""""""""""""""

1. 设备初始化

.. code:: c

    #include "app_dmac_wrapper.h"
    app_dmac_init_wrapper();
    
Apollo系统已经自动执行过上述初始化，用户无需再次初始化。

2. 启动传输

.. code:: c

    app_dmac_transfer_param_t dma_param = {
            .callback = callback_function,
            .callback_param = cb_param,
            .src = src_data,
            .dst = dst_data,
            .length = length,
            .src_tr_width = Transfer_Width_8_bits,
            .dst_tr_width = Transfer_Width_8_bits,
            .src_msize = Burst_Transaction_Length_4,
            .dst_msize = Burst_Transaction_Length_4,
            .tt_fc = Memory_to_Memory_DMAC_Flow_Controller,
            .src_per = 0,
            .dst_per = 0,
            .int_en = Interrupt_Enabled,
    };
    uint8_t ch_idx = app_dmac_start_wrapper(&dma_param);
    
src_tr_width决定了length的单位，若src_tr_width == Transfer_Width_8_bits，则length == n对n个字节的传输；若src_tr_width == Transfer_Width_16_bits，则length == n对应2n个字节的传输。

src的地址的对齐要求由src_tr_width决定，即src_tr_width == Transfer_Width_16_bits； src应两字节对齐，src_tr_width == Transfer_Width_32_bits，src应四字节对齐。   
    
对于内存-内存的DMA传输，tt_fc应固定设置为Memory_to_Memory_DMAC_Flow_Controller，src_msize,dst_msize可以设置为任意合法的Burst_Transaction_Length_*枚举类型。

若中断使能，则回调函数在传输完成后中断处理上下文中被调用；若中断禁用，则不会调用回调函数。
    
外设-内存之间的DMA传输已集成在相应外设驱动中，无需用户额外配置DMA。
    
3. 等待传输完成（仅当传输参数int_en == Interrupt_Disabled时使用）

.. code:: c

    app_dmac_transfer_wait_wrapper(ch_idx);
    


