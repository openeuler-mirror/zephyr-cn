.. _develop_subsys_storage_flash_map:

Zephyr Flash Map实现分析
#########################

 在嵌入式系统中，通常会根据功能对Flash区域进行划分，然后按照区域进行使用。Zephyr的Flash Map模块对该功能进行抽象并API封装，方便对Flash区域的封装和使用。

分区
====

在zephyr的\ ``include/storage/flash_map.h``\中定义了描述分区的结构体\ ``struct flash_area``\, 各描述字段如下

.. code:: c

    struct flash_area {
        uint8_t fa_id;              //flash分区ID
        uint8_t fa_device_id;       //兼容mcuboot定义的flash分区
        uint16_t pad16;         
        off_t fa_off;               //该flash分区在flash中的偏移地址
        size_t fa_size;             //该flash分区的大小
        const char *fa_dev_name;    //flash分区所在flash的设备名
    };


zephyr的Flash Map就是由\ ``struct flash_area``\构成，我们可以通过设备树创建分区也可以在运行时自己指定\ ``struct flash_area``\

设备树创建分区
~~~~~~~~~~~~~~~

zephyr使用设备树中的fixed-partitions绑定生成分区表，该flash map的分区表在编译时决定，运行时无法被修改。
下面示例设备树：

.. code:: devicetree

    &flexspi {
        status = "okay";
        ahb-prefetch;
        ahb-read-addr-opt;
        rx-clock-source = <1>;
        reg = <0x402a8000 0x4000>, <0x60000000 DT_SIZE_M(8)>;
        is25wp064: is25wp064@0 {
            compatible = "nxp,imx-flexspi-nor";
            size = <67108864>;
            label = "IS25WP064";
            reg = <0>;
            spi-max-frequency = <133000000>;
            status = "okay";
            jedec-id = [9d 70 17];
            erase-block-size = <65536>;
            write-block-size = <1>;

            partitions {
                compatible = "fixed-partitions";
                #address-cells = <1>;
                #size-cells = <1>;

                boot_partition: partition@0 {
                    label = "bootloader";
                    reg = <0x00000000 DT_SIZE_K(512)>;
                };

                data_partition: partition@80000 {
                    label = "data";
                    reg = <0x00080000 DT_SIZE_K(512)>;
                };
                app_partition: partition@100000 {
                    label = "app";
                    reg = <0x00100000 DT_SIZE_M(7)>;
                };
            };
        };
    };


上面示例将一片8M的Flash分成了3个分区：

* boot_partition: flash内偏于地址为0x0, 大小为512K
* data_partition: flash内偏于地址为0x80000, 大小为512K
* app_partition: flash内偏于地址为0x100000, 大小为7M

在\ ``subsys/storage/flash_map/flash_map_default.c``\通过设备树的宏读取设备树中的partition信息并转换成\ ``struct flash_area``\结构体数组\ ``default_flash_map``\

.. code:: c

    //读取设备树
    #define FLASH_AREA_FOO(part)						\
        {.fa_id = DT_FIXED_PARTITION_ID(part),				\
        .fa_off = DT_REG_ADDR(part),					\
        .fa_dev_name = DT_LABEL(DT_MTD_FROM_FIXED_PARTITION(part)),	\
        .fa_size = DT_REG_SIZE(part),},

    //遍历所有分区
    #define FOREACH_PARTITION(n) DT_FOREACH_CHILD(DT_DRV_INST(n), FLASH_AREA_FOO)

    //生成flash map数组
    const struct flash_area default_flash_map[] = {
        DT_INST_FOREACH_STATUS_OKAY(FOREACH_PARTITION)
    };

    //flash map中含有分区的总数
    const int flash_map_entries = ARRAY_SIZE(default_flash_map);

    //flash map的指针
    const struct flash_area *flash_map = default_flash_map;


上面示例设备树被转化成的\ ``default_flash_map``\为下面内容

.. code:: c

    const struct flash_area default_flash_map[] = {
        //bootloadr分区
        {.fa_id = 0,
        .fa_off = 0x0,
        .fa_dev_name = "IS25WP064",
        .fa_size = 0x80000,},

        //data分区
        {.fa_id = 1,
        .fa_off = 0x80000,
        .fa_dev_name = "IS25WP064",
        .fa_size = 0x80000,},

        //app分区
        {.fa_id = 2,
        .fa_off = 0x100000,
        .fa_dev_name = "IS25WP064",
        .fa_size = 0x700000,},

    };

.. note::

    fa_id是由\ ``DT_FIXED_PARTITION_ID(part)``\生成，设备树被解析产生在\ ``devicetree_unfixed.h``\中对应如下，也就是按照设备树partition的顺序进行排列的
.. code:: c

    #define DT_N_S_soc_S_spi_402a8000_S_is25wp064_0_S_partitions_S_partition_0_PARTITION_ID 0
    #define DT_N_S_soc_S_spi_402a8000_S_is25wp064_0_S_partitions_S_partition_80000_PARTITION_ID 1
    #define DT_N_S_soc_S_spi_402a8000_S_is25wp064_0_S_partitions_S_partition_100000_PARTITION_ID 2


运行时创建分区
~~~~~~~~~~~~~~~

运行时创建分区非常简单，直接定义一个\ ``struct flash_area``\变量，并按需求赋值即可。例如在实际写代码过程中，我想在data分区首部4k放一个零时的配置分区，那么可以

.. code:: c

    struct flash_area cfg_partition={
        .fa_id = 6,
        .fa_off = 0x80000,
        .fa_dev_name = "IS25WP064",
        .fa_size = 0x1000,
    };


API
====

使用示例
~~~~~~~~~~~~~~~

下面示例简单演示在data_partition的0x2000处写入4096个byte的数据，再读出0x1000处4096个byte

.. code:: c

    const struct flash_area *partition;
    //打开data_partition
    flash_area_open(FLASH_AREA_ID(data_partition), &partition);
    //擦除data_partition中偏移地址0x2000处4k大小的flash空间
    flash_area_erase(partition, 0x2000, 4096);
    //写入4K数据到data_partition中偏移地址0x2000
    flase_area_write(partition, 0x2000, write_buf, 4096);
    //从data_partition中偏移地址为0x1000处读出4K数据
    flase_area_read(partition, 0x1000, read_buf, 4096);
    //关闭partition
    flase_area_close(partition);


下面示例运行时自定义的flase_area

.. code:: c

    //自定义flase_area
    struct flash_area cfg_partition={
        .fa_id = 6,
        .fa_off = 0x80000,
        .fa_dev_name = "IS25WP064",
        .fa_size = 0x1000,
    };
    //擦除cfg_partition中偏移地址0x0处4k大小的flash空间
    flash_area_erase(&cfg_partition, 0x0, 4096);
    //写入4K数据到cfg_partition中偏移地址0x0
    flase_area_write(&cfg_partition, 0x0, write_buf, 4096);

.. important::

    注意: 运行时自定义的flase_area, 不需要使用\ ``flash_area_open``\和\ ``flash_area_close``\进行管理，而能直接做读写操作.

API说明
~~~~~~~~~~~~~~~

flash_area的API几乎从API名和参数就可以自说明，这里就仅做简单注释说明，稍微复杂一点的API通过后文的API实现分析也能知道详细用途。

.. code:: c

    //判断flash partition是否存在
    //例如FLASH_AREA_LABEL_EXISTS(data)，会转换为一个宏，通过判断该宏是否被定义来判断label为data的partition是否存在
    #define FLASH_AREA_LABEL_EXISTS(label) \
        DT_HAS_FIXED_PARTITION_LABEL(label)

    //节点lbl的lable
    //例如FLASH_AREA_LABEL_STR(data_partition)就是"data"
    #define FLASH_AREA_LABEL_STR(lbl) \
        DT_PROP(DT_NODE_BY_FIXED_PARTITION_LABEL(lbl), label)

    //指定partition的ID
    //例如例如FLASH_AREA_LABEL_STR(data)就是1
    #define FLASH_AREA_ID(label) \
        DT_FIXED_PARTITION_ID(DT_NODE_BY_FIXED_PARTITION_LABEL(label))

    //分区在flash中的偏移地址
    //例如FLASH_AREA_OFFSET(data)是0x8000
    #define FLASH_AREA_OFFSET(label) \
        DT_REG_ADDR(DT_NODE_BY_FIXED_PARTITION_LABEL(label))

    //分区的大小
    //FLASH_AREA_SIZE(data)是512K
    #define FLASH_AREA_SIZE(label) \
        DT_REG_SIZE(DT_NODE_BY_FIXED_PARTITION_LABEL(label))

    //打开flash_area，获取指定id的flash_area
    int flash_area_open(uint8_t id, const struct flash_area **fa);
    //关闭flash_area
    void flash_area_close(const struct flash_area *fa);
    //读取flash_area内指定地址和长度的内容
    int flash_area_read(const struct flash_area *fa, off_t off, void *dst,
                size_t len);
    //向flash_area内指定地址写入指定长度的内容，写入长度要与flash_area_align返回的值大小对齐
    int flash_area_write(const struct flash_area *fa, off_t off, const void *src,
                size_t len);
    //擦除flash_arean内指定地址和长度的区域，擦除长度要满足flash本身或驱动的长度对齐限制，可以从flash_area_get_sectors得到。
    int flash_area_erase(const struct flash_area *fa, off_t off, size_t len);
    //获取flash_area写入对齐长度
    uint8_t flash_area_align(const struct flash_area *fa);
    //获取flash_area内扇区排列分布情况
    int flash_area_get_sectors(int fa_id, uint32_t *count,
                struct flash_sector *sectors);

    //使用user_cb遍历所有flash分区
    void flash_area_foreach(flash_area_cb_t user_cb, void *user_data);

    //坚持flash分区是否有驱动
    int flash_area_has_driver(const struct flash_area *fa);

    //获取flash分区的flash驱动device
    const struct device *flash_area_get_device(const struct flash_area *fa);
    uint8_t flash_area_erased_val(const struct flash_area *fa);

    //对flash分区内指定区域的内容进行sha256校验比较
    int flash_area_check_int_sha256(const struct flash_area *fa,
                    const struct flash_area_check *fac);


实现分析
========

flash_area的实现比较简单，主要是对flash驱动进行了封装，并加入分区的管理。

open & close
~~~~~~~~~~~~~

\ ``flash_area_open``\就是根据id取得编译时确定好的\ ``struct flash_area``\数组成员指针

.. code:: c

    int flash_area_open(uint8_t id, const struct flash_area **fap)
    {
        const struct flash_area *area;
        //没有flash_map
        if (flash_map == NULL) {
            return -EACCES;
        }

        //通过id找出匹配的flash_area
        //get_flash_area_from_id通过匹配flash_map中成员的fa_id和参数id，完成查找
        area = get_flash_area_from_id(id);
        if (area == NULL) {
            return -ENOENT;
        }

        *fap = area;
        return 0;
    }

\ ``flash_area_close``\是一个空函数，未做任何事。

read & write & erase
~~~~~~~~~~~~~~~~~~~~~

读写擦三个API，操作的方式都一样：通过device_get_binding获取flash device，然后通过flash API操作。只列出read的实现说明：

.. code:: c

    int flash_area_read(const struct flash_area *fa, off_t off, void *dst,
                size_t len)
    {
        const struct device *dev;
        
        //坚持操作的offset和len是否超过分区范围
        if (!is_in_flash_area_bounds(fa, off, len)) {
            return -EINVAL;
        }

        //获取flash device
        dev = device_get_binding(fa->fa_dev_name);

        //进行flash操作，对于写和擦除的实现这里就调用flase_write和flash_erase
        return flash_read(dev, fa->fa_off + off, dst, len);
    }

    //分区范围检查非常简单，就是将offset和len与分区的大小进行对比
    static inline bool is_in_flash_area_bounds(const struct flash_area *fa,
                        off_t off, size_t len)
    {
        return (off >= 0) && ((off + len) <= fa->fa_size);
    }


获取flash分区相关信息
~~~~~~~~~~~~~~~~~~~~~

检查是否有flash驱动支持该flash分区

.. code:: c

    int flash_area_has_driver(const struct flash_area *fa)
    {
        //获取分区flash驱动设备名对应的flash device，如果不为NULL表示有驱动可用
        if (device_get_binding(fa->fa_dev_name) == NULL) {
            return -ENODEV;
        }

        return 1;
    }

获取flash分区所使用的flash device

.. code:: c

    const struct device *flash_area_get_device(const struct flash_area *fa)
    {
        return device_get_binding(fa->fa_dev_name);
    }

获取flash分析写入对齐长度

.. code:: c

    uint8_t flash_area_align(const struct flash_area *fa)
    {
        const struct device *dev;

        dev = device_get_binding(fa->fa_dev_name);
        //写入对齐的长度和分区所在的flash写入对齐长度一致
        return flash_get_write_block_size(dev);
    }

flash分区中被擦除位置的值

.. code:: c

    uint8_t flash_area_erased_val(const struct flash_area *fa)
    {
        const struct flash_parameters *param;
        //和flash驱动擦除flash后的值一致
        param = flash_get_parameters(device_get_binding(fa->fa_dev_name));

        return param->erase_value;
    }


分区sha256校验对比
~~~~~~~~~~~~~~~~~~

当配置\ ``CONFIG_FLASH_AREA_CHECK_INTEGRITY=y``\后flash map提供\ ``flash_area_check_int_sha256``\支持对分区内指定区域的sha256计算并比较。

.. code:: c

    struct flash_area_check {
        const uint8_t *match;		// 256bit比较样本
        size_t clen;			// 比较内容的长度
        size_t off;				// 比较内容在分区内的偏移地址
        uint8_t *rbuf;			// 计算sha256用的buf
        size_t rblen;			// 计算sha256用的buf长度
    };

    //计算flash_area内从off开始内容长度为clen的sha256，并与match比较，一致时返回0，不一致时返回负数
    int flash_area_check_int_sha256(const struct flash_area *fa,
                    const struct flash_area_check *fac)
    {
        unsigned char hash[TC_SHA256_DIGEST_SIZE];
        struct tc_sha256_state_struct sha;
        const struct device *dev;
        int to_read;
        int pos;
        int rc;

        if (fa == NULL || fac == NULL || fac->match == NULL ||
            fac->rbuf == NULL || fac->clen == 0 || fac->rblen == 0) {
            return -EINVAL;
        }

        //检查的sha256的内容是否已经超出分区范围
        if (!is_in_flash_area_bounds(fa, fac->off, fac->clen)) {
            return -EINVAL;
        }

        //初始化sha256计算模块
        if (tc_sha256_init(&sha) != TC_CRYPTO_SUCCESS) {
            return -ESRCH;
        }

        dev = device_get_binding(fa->fa_dev_name);
        to_read = fac->rblen;

        for (pos = 0; pos < fac->clen; pos += to_read) {
            if (pos + to_read > fac->clen) {
                to_read = fac->clen - pos;
            }

            //读出flash分区内的内容
            rc = flash_read(dev, (fa->fa_off + fac->off + pos),
                    fac->rbuf, to_read);
            if (rc != 0) {
                return rc;
            }

            //计算读出内容的sha256值
            if (tc_sha256_update(&sha,
                        fac->rbuf,
                        to_read) != TC_CRYPTO_SUCCESS) {
                return -ESRCH;
            }
        }

        //完成hash计算
        if (tc_sha256_final(hash, &sha) != TC_CRYPTO_SUCCESS) {
            return -ESRCH;
        }

        //比较sha256
        if (memcmp(hash, fac->match, TC_SHA256_DIGEST_SIZE)) {
            return -EILSEQ;
        }

        return 0;
    }


遍历flash分区
~~~~~~~~~~~~~

通过flash_area_foreach可以遍历所有的flash分区，读取flash分区信息

.. code:: c

    //遍历分区回调
    typedef void (*flash_area_cb_t)(const struct flash_area *fa,
                    void *user_data);

    //用户实现一个flash_area_cb_t类型的回调user_cb，在函数内对flash_map逐个遍历，将flash_area的信息送给回调
    void flash_area_foreach(flash_area_cb_t user_cb, void *user_data)
    {
        for (int i = 0; i < flash_map_entries; i++) {
            user_cb(&flash_map[i], user_data);
        }
    }


参考
=====

https://docs.zephyrproject.org/latest/reference/storage/flash_map/flash_map.html