NVDS
======

NVDS is the abbreviation of Non-Volitale Data Storage. BLE SDK make use of NVDS APIs to get or set Device Address, Bonding information and System configuration parameters. NVDS gets and sets data according to a 1-byte *tag*. Tags are divided into read-only and read-write.

Read-Only Tags
---------------

    ===========================  =========================== 
    TAG                           Where is the TAG stored ?
    ===========================  =========================== 
    NVDS_TAG_BD_ADDRESS            Flash Information Page
    NVDS_TAG_DEVICE_NAME           Hardcoded in Firmware
    NVDS_TAG_EXT_WAKEUP_TIME       Hardcoded in Firmware
    NVDS_TAG_OSC_WAKEUP_TIME       Hardcoded in Firmware
    NVDS_TAG_RM_WAKEUP_TIME        Hardcoded in Firmware
    NVDS_TAG_SLEEP_ENABLE          Hardcoded in Firmware
    NVDS_TAG_EXT_WAKEUP_ENABLE     Hardcoded in Firmware
    NVDS_TAG_LPCLK_DRIFT           Hardcoded in Firmware
    ===========================  ===========================
    
Read-Write Tags
-----------------

Except Read-Only tags, all others tags are Read-Write. NVDS operations to Read-Write tags are implemented based on BXFS. At *nvds_init()*, a directory for NVDS is made under BXFS root directory. All NVDS operations to a Read-Write tag map to corresponding operations of BXFS to the record under NVDS directory. For example:

::
    
    nvds_get(0xa0)  --->    bxfs_read(nvds_dir,0xa0)   
    nvds_put(0xa0)  --->    bxfs_write(nvds_dir,0xa0)
    nvds_del(0xa0)  --->    bxfs_del_record(nvds_dir,0xa0)
    
Becasue of data caching feature of BXFS, it's also necessary to validate your changes by **nvds_write_through()**.