BXFS
=====
BXFS is a simplified Nor Flash file system used for non-volatile data storage which features tree structure for managing directories and records, a wear-leveling algorithm based on rolling writing/erasing and resilience to power failure.


Introduction
-------------
There are two important aspects for profiling BXFS design. One aspect is how it works from the perspective of upper layer applications. The other is how it works with the lower layer physical storage media.

From the perspective of upper layer applications, data are stored in records. Each record belongs to its parent directory. Directories and records forms a tree structure. Root directory (ROOT_DIR) is the top node in this tree structure. It exists once BXFS is initialized and can never be deleted. New directories can be created as the child of ROOT_DIR or any existing directory. Writing to a non-existing record will create the record. Writing to an existing record will update its original content. Both directory and record can be removed.

Unlike normal file system, the name of record or directory in BXFS is denoted by two-byte unsigned integer instead of character strings. This minimizes the footprint of data stored in Flash. Different directories with a same parent must have different names while directories in different parents may use a same integer as theirs name. Record naming follows the same rule.

With repect to the low level operation, BXFS maps all upper layer operations to writing new log into Flash. There are four types of log information: *ADD_RECORD*, *ADD_DIR*, *REMOVE_RECORD*, *REMOVE_DIR*. Log information already written will not be modified unless it is erased during garbage collection. 

In fact, writing log to Flash does not always invoke Flash programming. There is a 256-byte cache between BXFS writing and Flash programming. Every time BXFS writes a new log, data will be stored in the cache. Data in the cache will only be programmed into Flash by explicit call to flush cache or when the cache is full.

How to use BXFS
----------------
During system initializing, *bxfs_init()* is called. Then all other BXFS APIs are avaiable.

Make Directory
~~~~~~~~~~~~~~~~
.. code:: c
    
    #define DIR_1_NAME 0x1
        #define DIR_11_NAME 0x1
        #define DIR_12_NAME 0x2
    #define DIR_2_NAME 0x2
        #define DIR_21_NAME 0x1
        #define DIR_22_NAME 0x2
    bxfs_dir_t dir_1;
    bxfs_dir_t dir_11;
    bxfs_dir_t dir_12;
    bxfs_dir_t dir_2;
    bxfs_dir_t dir_21;
    bxfs_dir_t dir_22;
    
    void make_directories()
    {
        uint8_t ret_val;
        ret_val = bxfs_mkdir(&dir_1,ROOT_DIR,DIR_1_NAME);
        BX_ASSERT(ret_val == BXFS_NO_ERROR || ret_val == BXFS_DIR_KEY_ALREADY_EXISTED);
        ret_val = bxfs_mkdir(&dir_11,dir_1,DIR_11_NAME);
        BX_ASSERT(ret_val == BXFS_NO_ERROR || ret_val == BXFS_DIR_KEY_ALREADY_EXISTED);
        ret_val = bxfs_mkdir(&dir_12,dir_1,DIR_12_NAME);
        BX_ASSERT(ret_val == BXFS_NO_ERROR || ret_val == BXFS_DIR_KEY_ALREADY_EXISTED);
        ret_val = bxfs_mkdir(&dir_2,ROOT_DIR,DIR_2_NAME);
        BX_ASSERT(ret_val == BXFS_NO_ERROR || ret_val == BXFS_DIR_KEY_ALREADY_EXISTED);
        ret_val = bxfs_mkdir(&dir_21,dir_2,DIR_21_NAME);
        BX_ASSERT(ret_val == BXFS_NO_ERROR || ret_val == BXFS_DIR_KEY_ALREADY_EXISTED);
        ret_val = bxfs_mkdir(&dir_22,dir_2,DIR_22_NAME);
        BX_ASSERT(ret_val == BXFS_NO_ERROR || ret_val == BXFS_DIR_KEY_ALREADY_EXISTED);    
    }

    
Read Write Record
~~~~~~~~~~~~~~~~~~


Delete Record
~~~~~~~~~~~~~~


Delete Directory
~~~~~~~~~~~~~~~~~~


List All Records in A Directory
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



