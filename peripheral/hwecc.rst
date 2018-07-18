=================
HWECC
=================
"""""""""""""""""
特性
"""""""""""""""""

* ECC-P256 (defined in `FIPS 186-4 <https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.186-4.pdf>`_)
"""""""""""""""""
使用
"""""""""""""""""

1. 设备初始化

.. code:: c

    #include "app_hwecc_wrapper.h"
    app_hwecc_init_wrapper();
    
BX2400系统已经自动执行过上述初始化，用户无需再次初始化。
    
2. ECC计算

.. code:: c

    void start_ecc_calculation(uint8_t const *secret_key,uint8_t const *public_key_x,uint8_t const *public_key_y,uint8_t *result_x,uint8_t *result_y,void (*callback)(void *),void *callback_param)
    {
        ecc_queue_t ecc_param = 
        {
            .in = {
                .secret_key = secret_key,
                .publick_key = {
                    [0] = public_key_x,
                    [1] = public_key_y,
                },
            },
            .out = {
                [0] = result_x,
                [1] = result_y,
            },
            .cb = callback,
            .dummy = callback_param,
        };
        app_hwecc_calculate_wrapper(&ecc_param);    
    }

