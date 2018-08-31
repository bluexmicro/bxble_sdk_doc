
NVDS
======

NVDS是FLASH中保存系统、用户数据的区域。例如蓝牙设备地址、蓝牙设备名称等系统信息以及用户自定义需要长期存储的数据会存放在该区域。

NVDS TAG
---------

在NVDS中，数据以标签Tag的形式存储，每一Tag中，包含该Tag的大小、名称、状态、数据内容等所有信息。BX2400为开发者提供了在C源代码中直接编辑NVDS数据的方法。如果在NVDS中增加新的Tag，需要分别在modules/nvds/api/nvds_tag.h下声明新的Tag结构体，在modules/nvds/src/nvds_tag.c下定义新的Tag数据内容。

例如，定义名为BX_USR_TAG0的Tag，其中保存2个uint32_t类型的数据，步骤如下：

#. 添加Tag名称到枚举类型（nvds_tag.h）

    .. image:: nvds_usr_tag.png
    
#. 声明Tag结构体（nvds_tag.h）

    .. image:: nvds_usr_data_define.png
    
    在nvds_data_t类型定义中添加：
    
    .. image:: nvds_decl_tag.png
    
#. 定义Tag数据内容（nvds_tag.c）
    
    若2个uint32_t类型的数分别为0x12345678,0xfedcba98
    
    在nvds_data变量内添加：
    
    .. image:: nvds_define_tag.png
    
NVDS API
--------

.. code:: c
    
    /* 读取tag内容，填入buf，更新*lengthPtr */
    uint8_t nvds_get(uint8_t tag, nvds_tag_len_t * lengthPtr, uint8_t *buf);
    
    /* 删除tag */
    uint8_t nvds_del(uint8_t tag);
    
    /* 把buf中长度为length的数据写入tag */
    uint8_t nvds_put(uint8_t tag, nvds_tag_len_t length, uint8_t *buf);

    
NVDS存储结构
------------


    
NVDS访问策略
------------

NVDS利用FLASH存储数据，因此NVDS数据的读、写、删都要建立在FLASH的工作特性上。与BX2400搭配的是常见的Nor FLASH，Nor FLASH一般支持随机读取、按Sector擦除以及擦除后烧写，因此NVDS数据的写、删是完全不同于内存SRAM数据的写、删的。

NVDS Tag的删除采用标记形式。程序映像创建时，NVDS Tag中的Label被初始化为0xff，若要删除该Tag，则将FLASH中该Tag的Label中的Valid bit写0。

NVDS Tag的写入采用标记和增量烧写形式。如果要写入的Tag已经存储在NVDS中，则将该Tag在NVDS中当前的记录设为无效（label.valid写0）。接着定位到当前NVDS Tag区域的尾部，写入新的Tag。

随着系统长时间的运行，有些Tag会被多次修改，因此NVDS Tag区域的数据不断膨胀，其中包含着许多有效的Tag和许多无效的Tag。当Tag区域数据膨胀将要超过Nvds_Size时，系统会将当前NVDS SECTION的有效Tag搬移至另一NVDS SECTION。这样就将原本NVDS SECTION中无效的Tag对应大小的空间节省出来。若Tag的不断修改造成数据区域膨胀超过4K时，则重复上述操作。在从源地址搬移NVDS_SECTION到目的地址操作前，系统首先按Sector擦除FLASH目的地址数据，接着通过内存中存储的NVDS数据索引遍历读取FLASH中Tag数据内容，依次写入新的NVDS_SECTION，并更新内存NVDS数据索引信息，最后在新NVDS_SECTION的Header区域写入Magic_flag和Version版本号，新NVDS_SECTION中Version版本号在老NVDS_SECTION Version版本号的基础上自增1。这样NVDS的空间就可以循环往复利用，又减少了频繁擦除造成的FLASH寿命缩减，且能保证在搬移NVDS_SECTION过程中出现意外掉电等情况，不会造成FLASH中NVDS信息失效。


NVDS软件实现
------------


NVDS内存数据结构定义如下：

.. code:: c

    typedef struct
    {
        uint8_t struct_size; //Tag 结构大小
        uint8_t name;		//Tag 名称
        uint16_t offset;	//Tag 在NVDS中的偏移
    }nvds_tag_hash_table_t;
    
    typedef struct{
        nvds_tag_hash_table_t tag_table[NVDS_TAG_TABLE_MAX]; //NVDS Tag索引表
        uint8_t (*read)(uint32_t src,uint32_t length,uint8_t *dst);
        uint8_t (*write)(uint32_t dst,uint32_t length,uint8_t *src);
        uint8_t (*erase)(uint32_t addr);
        uint8_t *base;		// 当前NVDS_SECTION在FLASH中的基址
        uint8_t *available;	// 当前NVDS_SECTION Tag数据的末尾，可用空间的起始地址
        uint32_t ver;		// 当前NVDS_SECTION的Version版本号
        uint8_t current_blk;// 当前NVDS_SECTION：0或1
    }nvds_env_t;
    
    nvds_env_t nvds_env;

上电启动后，系统调用nvds_init()初始化NVDS。首先分别从两块NVDS_SECTION中读取Version版本号，选择较新的NVDS_SECTION初始化nvds_env.base, nvds_env.available, nvds_env.ver, nvds_env.current_blk等值。为了加速用户访问NVDS Tag时的索引速度，初始化时，系统会遍历NVDS Tag，读取Tag Header，根据size和label的valid bit确定当前Tag是否具有有效数据。如果具有有效数据，则在内存tag_table数组中建立Tag的hash索引，以双散列方式解决hash冲突。

系统调用nvds_get()读取NVDS Tag时，利用Tag名称从nvds_env.tag_table哈希表中搜索定位，再从FLASH中读取Tag数据内容。

系统调用nvds_del()删除NVDS Tag时，利用Tag名称从nvds_env.tag_table哈希表中搜索定位，将该Tag Label的Valid bit写0，并将哈希表中该Tag的索引项内容清0。

系统调用nvds_put()写入NVDS Tag时，首先检查当前NVDS_SECTION是否有足够的空间进行增量烧写，若没有，则先进行NVDS_SECTION的搬移。接着利用Tag名称从nvds_env.tag_table哈希表中搜索定位，若存在，则在nvds_env.available所指示地址开始写入新的Tag，更新哈希表中相应的索引项，并且将老Tag置为无效，若不存在，则从nvds_env.tag_table中分配一个项用以索引，并向FLASH中写入Tag。


