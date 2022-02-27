.. _storage_nvs:

Zephyr NVS模块使用说明
#########################

NVS是非易失性存储的简写, 而Zephyr中的NVS单只是对Flash功能进行包装，并提供一个“识别号-数据”对的访问接口，用于存储系统中经常改变的数据。例如播放器的音量，当前的环境噪声，设备重启的次数。我们知道在Flash上写入数据是需要先擦除，而Flash的擦除次数是有限制的，引入NVS的目的是通过Flash的空间换取Flash不被频繁的擦除，并均衡使用Flash的磨损。

Zephyr的NVS基本构成如下：

NVS以扇区(sector)为单位管理Flash，一个扇区包含一个或以上的Flash页，扇区必须按照Flash的页大小对齐。NVS至少要2个扇区，NVS永远维持一个空扇区做记录回收用，NVS写入记录的数据小于一个扇区。

NVS数据以记录为单位，一条记录就是一个id-data对。在NVS中以唯一id来标识记录。id的长度为16bit，最大0xFFFF被作为特殊用途，也就是说NVS可以存储的数据条目不能超过65535条。

NVS写入记录时不会擦除原来的记录，而是先检查NVS上已有记录，如果有相同的id-data对就不进行写入，如果没有则进行写入。所以NVS内可能会存在多条id相同的记录。

NVS读出记录时，会读出指定id最近的一条写入记录。

接口
====

基于\ ``include/fs/nvs.h``\进行注释分析

.. code:: c

    /**
    * @brief nvs_init
    *
    * 在Flash上初始化NVS文件系统.
    *
    * @param fs 指向nvs
    * @param dev_name Flash设备名
    * @retval 0 表示成功
    * @retval 负数为失败返回的errno
    */
    int nvs_init(struct nvs_fs *fs, const char *dev_name);

    /**
    * @brief nvs_clear
    *
    * 清除NVS文件系统
    * @param fs 指向nvs
    * @retval 0 表示成功
    * @retval 负数为失败返回的errno
    */
    int nvs_clear(struct nvs_fs *fs);

    /**
    * @brief nvs_write
    *
    * 向nvs写入一条记录.
    *
    * @param fs 指向nvs
    * @param id 记录的ID
    * @param data 要被写入记录的数据
    * @param len 被写入的长度
    * @return 返回实际的写入数据长度，应该和len相等，当返回0时表示找到相同的id-data，无需再次写入。负数为失败返回的errno。
    */
    ssize_t nvs_write(struct nvs_fs *fs, uint16_t id, const void *data, size_t len);

    /**
    * @brief nvs_delete
    *
    * 从nvs中删除一条记录
    *
    * @param fs 指向nvs
    * @param id 删除记录的ID
    * @retval 0 表示成功
    * @retval 负数为失败返回的errno
    */
    int nvs_delete(struct nvs_fs *fs, uint16_t id);

    /**
    * @brief nvs_read
    *
    * 从nvs中读出一条记录.
    *
    * @param fs 指向nvs
    * @param id 要读出记录的ID
    * @param data 被读出的数据
    * @param len 读出数据的长度
    *
    * @return 返回实际的读出数据长度，应该和len想等。当返回值大于len时表示该条记录还有数据没有读完。负数为失败返回的errno。
    */
    ssize_t nvs_read(struct nvs_fs *fs, uint16_t id, void *data, size_t len);

    /**
    * @brief nvs_read_hist
    *
    * 读NVS内的历史记录
    *
    * @param fs 指向nvs
    * @param id 要读出记录的ID
    * @param data 被读出的数据
    * @param len 读出数据的长度
    * @param cnt 历史计数器，0表示最近的一个
    *
    * @return 返回实际的读出数据长度，应该和len想等。当返回值大于len时表示该条记录还有数据没有读完。负数为失败返回的errno。
    */
    ssize_t nvs_read_hist(struct nvs_fs *fs, uint16_t id, void *data, size_t len, uint16_t cnt);

    /**
    * @brief nvs_calc_free_space
    *
    * 计算NVS剩余可用空间.
    *
    * @param fs 指向nvs
    *
    * @return 返回NVS上可用空间的大小.如果返回负数则为错误的errno
    */
    ssize_t nvs_calc_free_space(struct nvs_fs *fs);


使用
====

默认情况下NVS是被禁用的，通过下面配置项可以启用NVS

.. code:: kconfig

    # 启用NVS
    CONFIG_NVS=y

    # NVS依赖Flash，需要启用Flash驱动
    CONFIG_FLASH=y
    CONFIG_FLASH_PAGE_LAYOUT=y

    # 当使用的flash地址空间被MPU保护时，需要配置允许对应地址空间被写入
    CONFIG_MPU_ALLOW_FLASH_WRITE=y


以下示例代码展示了如何使用nvs记录系统启动的次数和检查系统是否正常关机。

.. code:: c

    #define REBOOT_COUNT_ID 0
    #define POWER_NORMAL_ID 1
    #define FLASH_LABLE "FLASH_ESP32C3"
    struct nvs_fs fs;
    struct flash_pages_info info;
    uint16_t reboot_cnt = 0;
    bool power_normal = false;

    const struct device *flash_dev = device_get_binding(FLASH_LABLE);

    flash_get_page_info_by_offs(flash_dev, fs.offset, &info);

    // nvs放到flash的0x250000处，每个扇区使用一个flash页面，nvs总计3个扇区
    fs.offset = 0x250000 
    fs.sector_size = info.size;
    fs.sector_count = 3U;

    // 初始化nvs
    rc = nvs_init(&fs, FLASH_LABLE);
    if (rc) {
        printk("Flash Init failed\n");
        return;
    }

    // 读出上一次reboot count，如果存在做一次加1
    rc= nvs_read(&fs, REBOOT_COUNT_ID, &reboot_cnt, sizeof(reboot_cnt));
    if(rc == sizeof(reboot_cnt)){
        reboot_cnt++;
    }else{
        reboot_cnt = 0;
    }

    // 更新重启的次数
    nvs_write(&fs, REBOOT_COUNT_ID, &reboot_cnt, sizeof(reboot_cnt));

    //读出关机记录
    rc= nvs_read(&fs, POWER_NORMAL_ID, &power_normal, sizeof(power_normal));
    if(rc == sizeof(power_normal)){
        //删除关机记录
        nvs_delete(&fs, POWER_NORMAL_ID);
    }

    if(power_normal == true){
            // 正常关机处理
    }else{
            // 异常关机处理
    }

    // 处理
    switch(status){
        case DO_SOMTING:
        break;
        case POWER_OFF:
        //关机流程，写入正常关机标记
        nvs_write(&fs, POWER_NORMAL_ID, &power_normal, sizeof(power_normal));
        //进行关机处理
        break;
        case NVS_FORMAT:
        //清除NVS文件系统
        nvs_clear(&fs);
        break;
        default:
        break;
    }



参考
====

https://docs.zephyrproject.org/latest/reference/storage/nvs/nvs.html