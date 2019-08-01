Memory map and Image structure
================================

Runtime memory map
---------------------

+------------+----------+
|0x20300100  |          |
|            | Registers|
|0x20100000  |          |
+------------+----------+
|0x1000000   |          |
|            | FLASH    |
|0x800000    | 8M Max   |
+------------+----------+
|0x130000    | EM(BLE)  |
|            | 32K      |
|0x128000    +----------+
|            | SRAM     |
|0x100000    | 160K     |
+------------+----------+
|0x20000     |          |
|            |  ROM     |
|0x0         |  128K    |
+------------+----------+


.. note::
    The address ranged from 0x800000 to 0x1000000 is memory-mapped for QSPI-FLASH only when hardware cache is working. Hardware cache automatically manages QSPI Controller, fetches data from Flash, returns the data to system bus. If the program is going to perform the operation, such as **erase**, **program**, **read**, on Flash, hardware cache will be disabled until the operation is done.

Flash usage
------------

+-------------+----------------------+-----------------+
|Flash Offset |Memory Mapped Address |                 |
+=============+======================+=================+
|data_base    |0x800000+data_base    |bxfs data        |
+-------------+----------------------+-----------------+
|             |                      |                 |
+-------------+----------------------+-----------------+
|ota_base     |0x800000+ota_base     |ota image        |
+-------------+----------------------+-----------------+
|             |                      |                 |
+-------------+----------------------+-----------------+
|0x3000       |0x803000              |active image     |
+-------------+----------------------+-----------------+
|0x2000       |0x802000              |information page |
+-------------+----------------------+-----------------+
|0x0          |0x800000              |bootloader       |
+-------------+----------------------+-----------------+

The bootloader is placed at the first 8KB of Flash. 

The next 4KB is used for information page. 

Information page
    An area where product parameters are stored. This area are often programed to different values during production. The first 6 bytes of information page are predefined for BLE Public MAC Address.
    
The ota_base and data_base are designated in config.ini,see  :ref:`flash_parameters_configuration`.

Ota_base
    When performing firmware OTA, the new image is temporarily placed at ota_base. The new image will be copied to original image base 0x803000 after authentication in the boot sequence.

Data_base
    Persistent data are stored in flash starting from data_base by bxfs. Such as security data of bonding in BLE application, network status data in MESH application.
    See :doc:`bxfs_intro` for more about bxfs.
