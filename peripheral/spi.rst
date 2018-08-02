=================
SPI
=================
"""""""""""""""""
特性
"""""""""""""""""

* 数据帧4-32bit

* 时钟极性、相位选择

* 非阻塞传输（中断模式、DMA模式）

* 硬件CS控制

"""""""""""""""""
使用
"""""""""""""""""

1. 设备实例化

.. code:: c

    #include "app_spi.h"   
    app_spi_inst_t spim1_inst = SPIM_INSTANCE(1); // SPI_Master_1 实例化
    app_spi_inst_t spis_inst = SPIS_INSTANCE();   // SPI_Slave 实例化
    
2. 参数设置，设备初始化

.. code:: c

    void spim1_init()
    {
        spim1_inst.param.u.master.clk_div = 2;
        spim1_inst.param.dfs_32 = DFS_32_32_bits;
        spim1_inst.param.cpol = Inactive_Low;
        spim1_inst.param.cph = SCLK_Toggle_In_Middle;
        spim1_inst.param.clk_pin_no = 9;
        spim1_inst.param.mosi_pin_no = 11;
        spim1_inst.param.miso_pin_no = 10;
        spim1_inst.param.u.master.cs_pin_no[1] = 7;
        spim1_inst.param.u.master.valid_cs_mask = 0x2;//CS0无效、CS1有效
        app_spi_init(&spim1_inst.inst);
    }
    
    void spis_init()
    {   
        spis_inst.param.dfs_32 = DFS_32_32_bits;
        spis_inst.param.cpol = Inactive_Low;
        spis_inst.param.cph = SCLK_Toggle_In_Middle;
        spis_inst.param.clk_pin_no = 4;
        spis_inst.param.mosi_pin_no = 6;
        spis_inst.param.miso_pin_no = 5;
        spis_inst.param.u.slave.cs_pin_no = 3;
        app_spi_init(&spis_inst.inst);
    }
    
BX2400中SPIM和SPIS的管脚是固定的，不可任意配置。

数据对齐要求：

.. list-table::
    
    * - dfs_32 <= DFS_32_8_bits
      - DFS_32_8_bits < dfs_32 <= DFS_32_16bits
      - DFS_32_16_bits < dfs_32 <= DFS_32_32bits    
    * - 字节对齐
      - 双字节对齐
      - 四字节对齐      

3. 数据收发

.. code:: c
    
    //SPI Master
    app_spi_transmit(&spim1_inst.inst,cs_sel_mask,spim_tx_data,tx_length,spim_tx_cb,tx_cb_param);
    app_spi_receive(&spim1_inst.inst,cs_sel_mask,spim_rx_data,rx_length,spim_rx_cb,rx_cb_param);
    app_spi_transmit_receive(&spim1_inst.inst,cs_sel_mask,spim_tx_data,spim_rx_data,tx_rx_length,spim_tx_rx_cb,tx_rx_cb_param); // full duplex tx & rx
    
    //SPI Slave
    app_spi_transmit(&spis_inst.inst,0,spis_tx_data,tx_length,spis_tx_cb,tx_cb_param);
    app_spi_receive(&spis_inst.inst,0,spis_rx_data,rx_length,spis_rx_cb,rx_cb_param);
    app_spi_transmit_receive(&spis_inst.inst,0,spis_tx_data,spis_rx_data,tx_rx_length,spis_tx_rx_cb,tx_rx_cb_param); // full duplex tx & rx
    
收、发完成时，回调函数在中断处理上下文中被调用。

对于同一个SPI实例，必须在一次传输完成后，才可以发起下一次传输，即发起一种SPI传输后，必须在收发完成，回调函数被调用的时刻之后，才可以再发起SPI传输。

存储收发数据的缓冲区必须是动态或静态分配的全局数组，且在收发完成的回调函数被调用时，才能释放。
    
