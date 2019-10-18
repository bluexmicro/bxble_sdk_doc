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
During system initializing, *bxfs_init()* is called. Then all other BXFS APIs are available.

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

After these operations, the tree looks like:

::

    ROOT_DIR
    |-DIR_1_NAME
    | |-DIR_11_NAME
    | |-DIR_12_NAME
    |-DIR_2_NAME
      |-DIR_21_NAME
      |-DIR_22_NAME

    
Write Record
~~~~~~~~~~~~~~
.. code :: c
   
    #define ROOT_RECORD_1 0x1
    #define ROOT_RECORD_1_DATA "root_record_1_data"
    #define DIR_1_RECORD_1 0x1
    #define DIR_1_RECORD_1_DATA "dir_1_record_1_data"
    #define DIR_2_RECORD_1 0x1
    #define DIR_2_RECORD_1_DATA "dir_2_record_1_data"
    #define DIR_11_RECORD_2 0x2
    #define DIR_11_RECORD_2_DATA "dir_11_record_2_data"
    
    void records_write()
    {
        uint8_t ret_val = bxfs_write(ROOT_DIR,ROOT_RECORD_1,ROOT_RECORD_1_DATA,sizeof(ROOT_RECORD_1_DATA));
        BX_ASSERT(ret_val == BXFS_NO_ERROR);
        uint8_t ret_val = bxfs_write(dir_1,DIR_1_RECORD_1,DIR_1_RECORD_1_DATA,sizeof(DIR_1_RECORD_1_DATA));
        BX_ASSERT(ret_val == BXFS_NO_ERROR);
        uint8_t ret_val = bxfs_write(dir_2,DIR_2_RECORD_1,DIR_2_RECORD_1_DATA,sizeof(DIR_2_RECORD_1_DATA));
        BX_ASSERT(ret_val == BXFS_NO_ERROR);
        uint8_t ret_val = bxfs_write(dir_11,DIR_11_RECORD_2,DIR_11_RECORD_2_DATA,sizeof(DIR_11_RECORD_2_DATA));
        BX_ASSERT(ret_val == BXFS_NO_ERROR);
    }
    
After these operations, the tree looks like:

::

    ROOT_DIR
    |-DIR_1_NAME
    | |-DIR_11_NAME
    | | |+DIR_11_RECORD_2
    | |-DIR_12_NAME
    | |+DIR_1_RECORD_1
    |-DIR_2_NAME
    | |-DIR_21_NAME
    | |-DIR_22_NAME
    | |+DIR_2_RECORD_1
    |+ROOT_RECORD_1
    
Read Record
~~~~~~~~~~~~
.. code :: c

    uint8_t dir_11_record_2[20];
    void dir_11_record_2_read()
    {
        uint16_t length = 20;
        uint8_t ret_val = bxfs_read(dir_11,DIR_11_RECORD_2,dir_11_record_2,&length);
        BX_ASSERT(ret_val == BXFS_NO_ERROR && 
            length == sizeof(DIR_11_RECORD_2_DATA) &&
            memcmp(DIR_11_RECORD_2_DATA,dir_11_record_2,length) == 0 );
    }

Delete Record
~~~~~~~~~~~~~~
.. code :: c

    void root_record_1_del()
    {
        uint8_t ret_val = bxfs_del_record(ROOT_DIR,ROOT_RECORD_1);
        BX_ASSERT(ret_val == BXFS_NO_ERROR);
    }

After these operations, the tree looks like:

::

    ROOT_DIR
    |-DIR_1_NAME
    | |-DIR_11_NAME
    | | |+DIR_11_RECORD_2
    | |-DIR_12_NAME
    | |+DIR_1_RECORD_1
    |-DIR_2_NAME
      |-DIR_21_NAME
      |-DIR_22_NAME
      |+DIR_2_RECORD_1

Delete Directory
~~~~~~~~~~~~~~~~~~
.. code :: c

    void dir_21_del()
    {
        bool force = false; // DIR_21 is empty, so 'force' is false
        uint8_t ret_val = bxfs_del_dir(dir_21,force);
        BX_ASSERT(ret_val == BXFS_NO_ERROR);
    }
    
    void dir_2_del()
    {
        bool force = true; // DIR_2 is not empty,so 'force' is true
        uint8_t ret_val = bxfs_del_dir(dir_2,force);
        BX_ASSERT(ret_val == BXFS_NO_ERROR);
    }

After these operations, the tree looks like:

::

    ROOT_DIR
    |-DIR_1_NAME
      |-DIR_11_NAME
      | |+DIR_11_RECORD_2
      |-DIR_12_NAME
      |+DIR_1_RECORD_1

**Validate All Changes**
~~~~~~~~~~~~~~~~~~~~~~~~~
.. code :: c

    void validate_all_changes_at_once()
    {
        bxfs_write_through();
    }    

Once bxfs_write_through() has been called, all changes made just now are stored to Flash. All information can be restored correctly if there is a power failure afterwards.

List All Records in A Directory
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.. code :: c

    uint16_t dir_1_record_list[10];
    void get_record_list_in_dir_1()
    {
        uint16_t num = 10;
        uint8_t ret_val = bxfs_record_list_get(dir_1,&num,dir_1_record_list);
        BX_ASSERT(ret_val == BXFS_NO_ERROR && num == 1 && dir_1_record_list[0] == DIR_1_RECORD_1);
    }
    
Erase all data in BXFS
~~~~~~~~~~~~~~~~~~~~~~~~
.. code :: c
    
    bxfs_erase_data();
    
All data stored in BXFS, such as BLE bonding information, Mesh network status and user customized data, will be erased physically. This is commonly used for restoring factory settings.

Flash Storage Format
---------------------

The format of 4 types of log information is following:

+-------------------------------------------------------------------------------------------------------------------------------------------+
| log format                                                                                                                                |
+====================+=====+======+======+=========+=====+======+====+======+=====+============+=================+=======+==================+
|type \\         byte| 0   | 1    | 2    | 3       | 4   |  5   |  6 | 7    |  8  | 9 ... N-3  | N-2             | N - 1 | total_length     |
+--------------------+-----+------+------+---------+-----+------+----+------+-----+------------+-----------------+-------+------------------+
|add_record          | 0x0 |record_name  |parent_node_idx|data_length|crc16       | record_data| crc16 of record_data    | 9+data_length + 2|
+--------------------+-----+-------------+---------------+-----------+------------+------------+-------------------------+------------------+
|add_dir             | 0x1 |dir_name     |parent_node_idx|node_idx   |crc16       |                                      | 9                |
+--------------------+-----+-------------+---------------+-----------+------------+--------------------------------------+------------------+
|remove_record       | 0x2 |record_name  |parent_node_idx|crc16      |                                                   | 7                |
+--------------------+-----+-------------+---------------+-----------+---------------------------------------------------+------------------+
|remove_dir          | 0x3 |node_idx     |crc16          |                                                               | 5                |
+--------------------+-----+-------------+---------------+-----------+---------------------------------------------------+------------------+

For each section in use, the heading 6 bytes are section_head.

+------------------------------------------+
|section_head                              |
+=====+==+========+====+========+====+=====+ 
|byte |0 |1       |2   | 3      |4   |5    |
|     +--+--------+----+--------+----+-----+
|     |loop_count |node_offset  |crc16     |
+-----+-----------+-------------+----------+

The following picture shows BXFS storage with totally 4 sections and 5 logs across the first three sections.

.. image :: bxfs_format.png


Wear Levelling
---------------

Flash memory has a limited number of program-erase cycles(P/E cycles). In order to prolong the longevity of Flash, we should not erase some specific sectors frequently. Flash sectors used by BXFS are going to be erased with equal opportunity. New log information is programmed into Flash at the address that is the next byte following the previous log.

Along with new log programming into Flash, the available space continuously decreases. At the same time, some old logs which have been removed or modified by subsequent logs become obsolete. When the available space reduces to a specific level, the operation of garbage collection is triggered. BXFS reserves some erasable units as buffering section for garbage collection. In the procedure of garbage collection, valid logs in the oldest section are copied to the available space, then the oldest section is erased, thus releasing the space occupied by obsolete logs. This procedure continues if the condition of garbage collection is met again and again.

Based on this, all sections are programmed and erased equally. Any single section won't be worn down before all the others are nearly worn down.

Data Integrity and Power Failure Resilience
---------------------------------------------
BXFS ensures the data integrity. All parts of a log are authenticated by crc16 during BXFS initializtion and operation. Once bxfs_write_through() is called, all valid data can be recovered from a power loss. Because of the characteristic of BXFS that any page of Flash already programmed won't be modified, a power loss during Flash programming will only break the integrity of logs currently being programmed and won't lead to any damage to other logs already programmed. After reboot, BXFS will recognize these broken logs and correctly recover other data.



