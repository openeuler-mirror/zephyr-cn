.. _storage_flash_stream:

Zephyr flash流式操作
#########################

本文分析Zephyr的flash流式操作如何实现。

概述
=====

Flash在写入前必须擦除，擦除有page的限制，写入根据设备的不同可能会有对齐的限制。由于其操作的特殊性Flash无法直接接受流式数据的写入(连续的大小不定的数据)，为应对该情况Zephyr对flash驱动进行了封装提供stream_flash，用于支持流式数据的写入。
stream_flash模块的特性如下：

* 提供缓存，将流式数据收集到指定量后再写入到Flash
* 支持缓存回读
* 支持断电恢复机制

stream_flash最典型的应用就是DFU升级，在Zephyr中DFU升级和coredump写入flash都使用了stream_flash。

API
====

API说明
~~~~~~~

flash_stream的API声明在\ ``include/storage/stream_flash.h``\中，这里做注释说明

.. code:: c

    /**
    * @brief 初始化flash_stream.
    *
    * @param ctx flash_stream的上下文，相关的管理数据都保存在其中
    * @param fdev flash_stream使用的flash设备，流式数据将写入该设备
    * @param buf flash_stream使用的缓存，写入的流式数据将先保存到该缓存
    * @param buf_len 缓存大小，不能大于flash page尺寸，并按照flash的最小可写单位write-block-size对齐。
    * @param offset flash设备中的偏移地址
    * @param size 最大可写入量。当设置为0时表示可以从offset开始一直写到flash末尾
    * @param cb 每次flash写操作完成后就会调用这个回调
    *
    * @return 0为成功，负数为失败
    */
    int stream_flash_init(struct stream_flash_ctx *ctx, const struct device *fdev,
                uint8_t *buf, size_t buf_len, size_t offset, size_t size,
                stream_flash_callback_t cb);

    /**
    * @brief  写入数据到flash stream
    *
    * @param ctx flash_stream的上下文
    * @param data 写入数据指针
    * @param len 写入数据长度
    * @param flush 最后一笔写入时使用true强制写入到flash内，当为false时只有在stream_flash满时才会写flash
    *
    * @return 0为成功，负数为失败
    */
    int stream_flash_buffered_write(struct stream_flash_ctx *ctx, const uint8_t *data,
                    size_t len, bool flush);

    /**
    * @brief 获取目前有多少数据被写入到flash.
    *
    * @param ctx flash_stream的上下文
    *
    * @return 已写入到flash的数据长度.
    */
    size_t stream_flash_bytes_written(struct stream_flash_ctx *ctx);


在配置`CONFIG_STREAM_FLASH_ERASE=y`的情况下提供擦除API

.. code:: c

    /**
    * @brief 擦除flash页
    *
    * @param ctx flash_stream的上下文
    * @param off 擦除off所在页面
    *
    * @return 0为成功，负数为失败
    */
    int stream_flash_erase_page(struct stream_flash_ctx *ctx, off_t off);


在配置`CONFIG_STREAM_FLASH_PROGRESS=y`的情况下提供恢复机制API

.. code:: c

    /**
    * @brief 加载上一次保存的写入进度到flash_stream上下文中
    *
    * @param ctx flash_stream的上下文
    * @param settings_key setting模块中标识进度的key
    *
    * @return 0为成功，负数为失败
    */
    int stream_flash_progress_load(struct stream_flash_ctx *ctx,
                    const char *settings_key);

    /**
    * @brief 保存当前flash_stream的写进度 .
    *
    * @param ctx flash_stream的上下文
    * @param settings_key setting模块中标识进度的key
    *
    * @return 0为成功，负数为失败
    */
    int stream_flash_progress_save(struct stream_flash_ctx *ctx,
                    const char *settings_key);

    /**
    * @brief 清除flash_stream保存的进度.
    *
    * @param ctx flash_stream的上下文
    * @param settings_key setting模块中标识进度的key
    *
    * @return 0为成功，负数为失败
    */
    int stream_flash_progress_clear(struct stream_flash_ctx *ctx,
                    const char *settings_key);


使用示例
~~~~~~~~

这里做简单的使用示例如下注释，当不需要做断电恢复时，移除\ ``stream_flash_progress_xxx``\相关流程即可

.. code:: c

    #define STREAM_FLASH_KEY    "prop/stream_flash_key"

    struct stream_flash_ctx stream_flash;

    int data_verify(uint8_t *buf, size_t len, size_t offset)
    {
        //校验回读数据
        ...

        //通知显示当前下载进度last_writed
        size_t last_writed = stream_flash_bytes_written() 
        ....

        //保存当前下载进度
        stream_flash_progress_save(&stream_flash, STREAM_FLASH_KEY);

        //返回0表示校验通过
        return 0;
    }

    void download_task(void)
    {
        const struct device *fdev = device_get_binding(FLASH_NAME);
        uint_t buf[64*1024];

        //设用FLASH_NAME设备
        //缓存为buf，大小为64K
        //stream_flash，位于flash的0x100000处，大小为3M
        int stream_flash_init(&stream_flash, fdev,
                                buf, sizeof(buf), 0x100000, 3*1024*1024,
                                data_verify);

        //加载上次位置
        stream_flash_progress_load(&stream_flash, STREAM_FLASH_KEY);

        //读取上次写入的位置，通知从该处下载数据
        size_t last_writed = stream_flash_bytes_written() 
        .... nodify(last_writed)

        while(1){
            uint8 rev_buf[32];
            size_t rev_len;
            //接收数据
            int finish = revice(rev_buf, &rev_len);

            if(finish){
                //下载完所有数据成做flush写
                stream_flash_buffered_write(&stream_flash, rev_buf, rev_len, true);
                //清除下载进度，退出下载
                stream_flash_progress_clear(&stream_flash, STREAM_FLASH_KEY);
                break;
            }else{
                //写入当前下载数据，继续下载
                stream_flash_buffered_write(&stream_flash, rev_buf, rev_len, false);
            }
        }
    }



实现分析
========

stream flash实现的核心就是将数据缓存到buffer中，当buffer满后一次性写入到flash。通过\ ``struct stream_flash_ctx``\进行管理

.. code:: c

    struct stream_flash_ctx {
        uint8_t *buf; //写缓存，写入数据时先保存在该buffer中
        size_t buf_len; //写缓存的长度
        size_t buf_bytes; //写缓存中有效数据的长度
        const struct device *fdev; //stream_flash操控的flash device，数据实际要被写入到该flash设备上
        size_t bytes_written; //总计写了多少数据到flash
        size_t offset; //stream_flash从flash内该偏移地址开始使用
        size_t available; //flash内还剩多少区域可以被stream_flash使用
        stream_flash_callback_t callback; //每写一次flash，调用一次该callback
    #ifdef CONFIG_STREAM_FLASH_ERASE
        off_t last_erased_page_start_offset; //上一次擦除flash page所在的地址
    #endif
    };

各字段图示如下，后面的分析可以参考该图进行理解

.. image:: ../../../images/develop/subsys/storage/stream_flash.png

初始化
~~~~~~

stream flash初始化主要完成：对setting系统的初始化用于存储下载进度，完成对\ ``struct stream_flash_ctx``\的初始化赋值。

.. code:: c

    int stream_flash_init(struct stream_flash_ctx *ctx, const struct device *fdev,
                uint8_t *buf, size_t buf_len, size_t offset, size_t size,
                stream_flash_callback_t cb)
    {
        if (!ctx || !fdev || !buf) {
            return -EFAULT;
        }

    //初始化setting子系统，为存储下载进度做准备
    #ifdef CONFIG_STREAM_FLASH_PROGRESS
        int rc = settings_subsys_init();

        if (rc != 0) {
            LOG_ERR("Error %d initializing settings subsystem", rc);
            return rc;
        }
    #endif

        struct _inspect_flash inspect_flash_ctx = {
            .buf_len = buf_len,
            .total_size = 0
        };

        //缓存的长度必须与write-block-size对齐
        if (buf_len % flash_get_write_block_size(fdev)) {
            LOG_ERR("Buffer size is not aligned to minimal write-block-size");
            return -EFAULT;
        }

        //遍历flash的页，计算flash的大小
        flash_page_foreach(fdev, find_flash_total_size, &inspect_flash_ctx);

        /* The flash size counted should never be equal zero */
        if (inspect_flash_ctx.total_size == 0) {
            return -EFAULT;
        }

        //检查stream_flash使用的范围是否在flash内
        if ((offset + size) > inspect_flash_ctx.total_size ||
            offset % flash_get_write_block_size(fdev)) {
            LOG_ERR("Incorrect parameter");
            return -EFAULT;
        }

        //初始化flash_stream管理上下文
        ctx->fdev = fdev;
        ctx->buf = buf;
        ctx->buf_len = buf_len;
        ctx->bytes_written = 0;
        ctx->buf_bytes = 0U;
        ctx->offset = offset;
        //计算flash_stream实际可使用的flash空间，size为0时，就是从offset开始到flash结束的长度
        ctx->available = (size == 0 ? inspect_flash_ctx.total_size - offset :
                        size);
        ctx->callback = cb;

    #ifdef CONFIG_STREAM_FLASH_ERASE
        ctx->last_erased_page_start_offset = -1;
    #endif

        return 0;
    }


写入
~~~~

.. code:: c

    int stream_flash_buffered_write(struct stream_flash_ctx *ctx, const uint8_t *data,
                    size_t len, bool flush)
    {
        int processed = 0;
        int rc = 0;
        int buf_empty_bytes;

        if (!ctx) {
            return -EFAULT;
        }

        //计算flash内剩余的空间是否能容纳还没写入的数据总长度
        if (ctx->bytes_written + ctx->buf_bytes + len > ctx->available) {
            return -ENOMEM;
        }

        //写入flash_stream数据比缓存空间大时，先将缓存空间填满，再写入到flash
        while ((len - processed) >=
            (buf_empty_bytes = ctx->buf_len - ctx->buf_bytes)) {
            //填满缓存空间
            memcpy(ctx->buf + ctx->buf_bytes, data + processed,
                buf_empty_bytes);

            //将缓存空间数据写到flash
            ctx->buf_bytes = ctx->buf_len;
            rc = flash_sync(ctx);

            if (rc != 0) {
                return rc;
            }

            processed += buf_empty_bytes;
        }

        //剩余的数据不足以填满缓存空间的，就保留在缓存空间
        if (processed < len) {
            memcpy(ctx->buf + ctx->buf_bytes,
                data + processed, len - processed);
            ctx->buf_bytes += len - processed;
        }

        //如果指定flush，无论缓存空间剩多少都一次性写入到flash中
        if (flush && ctx->buf_bytes > 0) {
            rc = flash_sync(ctx);
        }

        return rc;
    }


flash写入流程，由于缓存的大小小于页，因此写入可以直接以页来操作

.. code:: c

    static int flash_sync(struct stream_flash_ctx *ctx)
    {
        int rc = 0;
        //计算flash内写入地址
        size_t write_addr = ctx->offset + ctx->bytes_written;
        size_t buf_bytes_aligned;
        size_t fill_length;
        uint8_t filler;


        if (ctx->buf_bytes == 0) {
            return 0;
        }

        //擦除页，页所在位置由写入数据最后一byte位置计算而得
        if (IS_ENABLED(CONFIG_STREAM_FLASH_ERASE)) {

            rc = stream_flash_erase_page(ctx,
                            write_addr + ctx->buf_bytes - 1);
            if (rc < 0) {
                LOG_ERR("stream_flash_erase_page err %d offset=0x%08zx",
                    rc, write_addr);
                return rc;
            }
        }

        //写入的数据不能和write-block-size对齐时，未对齐部分填入擦除flash后的值，通常时0xff
        //该操作只会在最后一笔数据flush发生
        fill_length = flash_get_write_block_size(ctx->fdev);
        if (ctx->buf_bytes % fill_length) {
            fill_length -= ctx->buf_bytes % fill_length;
            filler = flash_get_parameters(ctx->fdev)->erase_value;      //获取擦除flash后的值，通常是0xff

            memset(ctx->buf + ctx->buf_bytes, filler, fill_length);
        } else {
            fill_length = 0;
        }

        //写入flash
        buf_bytes_aligned = ctx->buf_bytes + fill_length;
        rc = flash_write(ctx->fdev, write_addr, ctx->buf, buf_bytes_aligned);

        if (rc != 0) {
            LOG_ERR("flash_write error %d offset=0x%08zx", rc,
                write_addr);
            return rc;
        }

        if (ctx->callback) {
            /* Invert to ensure that caller is able to discover a faulty
            * flash_read() even if no error code is returned.
            */
            for (int i = 0; i < ctx->buf_bytes; i++) {
                ctx->buf[i] = ~ctx->buf[i];
            }

            rc = flash_read(ctx->fdev, write_addr, ctx->buf,
                    ctx->buf_bytes);
            if (rc != 0) {
                LOG_ERR("flash read failed: %d", rc);
                return rc;
            }

            rc = ctx->callback(ctx->buf, ctx->buf_bytes, write_addr);
            if (rc != 0) {
                LOG_ERR("callback failed: %d", rc);
                return rc;
            }
        }

        ctx->bytes_written += ctx->buf_bytes;
        ctx->buf_bytes = 0U;

        return rc;
    }


上面你可能会发现，无论缓存是多大，即使小于一个page都会被执行\ ``stream_flash_erase_page``\，这会不会导致前面写入在同一页得数据被擦除呢? 我们来看一下其实现

.. code:: c

    int stream_flash_erase_page(struct stream_flash_ctx *ctx, off_t off)
    {
        int rc;
        struct flash_pages_info page;

        //找到要擦除的页
        rc = flash_get_page_info_by_offs(ctx->fdev, off, &page);
        if (rc != 0) {
            LOG_ERR("Error %d while getting page info", rc);
            return rc;
        }

        //如果该页的地址和上一次擦除的一致，就不再执行擦除，从而避免丢失
        if (ctx->last_erased_page_start_offset == page.start_offset) {
            return 0;
        }

        LOG_DBG("Erasing page at offset 0x%08lx", (long)page.start_offset);
        //执行擦除
        rc = flash_erase(ctx->fdev, page.start_offset, page.size);

        if (rc != 0) {
            LOG_ERR("Error %d while erasing page", rc);
        } else {
            //更新上一次擦除地址
            ctx->last_erased_page_start_offset = page.start_offset;
        }

        return rc;
    }


恢复机制实现
~~~~~~~~~~~~

恢复机制使用setting模块存储和改写\ ``struct stream_flash_ctx``\中的\ ``bytes_written``\字段完成，setting是一个可读写模块后端可以对接nvs或者文件等，这里不做展开说明，只用指定setting以key+value的map形式保存即可。

.. code:: c

    int stream_flash_progress_save(struct stream_flash_ctx *ctx,
                    const char *settings_key)
    {
        if (!ctx || !settings_key) {
            return -EFAULT;
        }
        //将bytes_written写入到setting中
        int rc = settings_save_one(settings_key,
                    &ctx->bytes_written,
                    sizeof(ctx->bytes_written));

        if (rc != 0) {
            LOG_ERR("Error %d while storing progress for \"%s\"",
                rc, settings_key);
        }

        return rc;
    }


    int stream_flash_progress_load(struct stream_flash_ctx *ctx,
                    const char *settings_key)
    {
        if (!ctx || !settings_key) {
            return -EFAULT;
        }

        //在回调settings_direct_loader中对cts中的bytes_written进行更新
        int rc = settings_load_subtree_direct(settings_key,
                            settings_direct_loader,
                            (void *) ctx);

        if (rc != 0) {
            LOG_ERR("Error %d while loading progress for \"%s\"",
                rc, settings_key);
        }

        return rc;
    }

    static int settings_direct_loader(const char *key, size_t len,
                    settings_read_cb read_cb, void *cb_arg,
                    void *param)
    {
        struct stream_flash_ctx *ctx = (struct stream_flash_ctx *) param;

        //查找满足条件的key
        if (settings_name_next(key, NULL) == 0) {
            size_t bytes_written = 0;

            //通过setting的callback和key读出bytes_written
            ssize_t len = read_cb(cb_arg, &bytes_written,
                        sizeof(bytes_written));

            //判断读出的长度是否合法
            if (len != sizeof(ctx->bytes_written)) {
                LOG_ERR("Unable to read bytes_written from storage");
                return len;
            }

            //更新ctx中的bytes_written
            if (bytes_written >= ctx->bytes_written) {
                ctx->bytes_written = bytes_written;
            } else {
                LOG_WRN("Loaded outdated bytes_written %zu < %zu",
                    bytes_written, ctx->bytes_written);
                return 0;
            }

            //更新last_erased_page_start_offset
    #ifdef CONFIG_STREAM_FLASH_ERASE
            int rc;
            struct flash_pages_info page;
            off_t offset = (off_t) (ctx->offset + ctx->bytes_written) - 1;

            /* Update the last erased page to avoid deleting already
            * written data.
            */
            if (ctx->bytes_written > 0) {
                rc = flash_get_page_info_by_offs(ctx->fdev, offset,
                                &page);
                if (rc != 0) {
                    LOG_ERR("Error %d while getting page info", rc);
                    return rc;
                }
                ctx->last_erased_page_start_offset = page.start_offset;
            } else {
                ctx->last_erased_page_start_offset = -1;
            }
    #endif /* CONFIG_STREAM_FLASH_ERASE */
        }

        return 0;
    }


    int stream_flash_progress_clear(struct stream_flash_ctx *ctx,
                    const char *settings_key)
    {
        if (!ctx || !settings_key) {
            return -EFAULT;
        }
        //删除存储的进度
        int rc = settings_delete(settings_key);

        if (rc != 0) {
            LOG_ERR("Error %d while deleting progress for \"%s\"",
                rc, settings_key);
        }

        return rc;
    }


参考
====

https://docs.zephyrproject.org/latest/reference/storage/stream/stream_flash.html

