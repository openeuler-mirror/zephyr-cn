.. _driver_flash_introduction:

Zephyr Flash驱动之驱动模型
##############################

本文介绍zephyr传感器驱动接口定义,使用和实现接口说明。

概述
====

Zephyr在flash.h中定义了统一的flash驱动接口，zephyr
flash的驱动模型提供了如下接口功能： 

* 对Flash操作的读，写，擦。

  * ``flash_read``
  * ``flash_write``
  * ``flash_erase``  

* 读取page layout信息``CONFIG_FLASH_PAGE_LAYOUT=y``后有效

  * ``flash_get_page_info_by_offs``
  * ``flash_get_page_info_by_idx``
  * ``flash_get_page_count``
  * ``flash_page_foreach`` 

* 读取sfdp信息

  * ``flash_sfdp_read``
  * ``flash_read_jedec_id`` 

* 读取flash信息

  * ``flash_get_write_block_size``
  * ``flash_get_parameters``

接口
====

类型定义
--------

``typedef bool (*flash_page_cb)(const struct flash_pages_info *info, void *data)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
通过\ ``flash_page_foreach()``\遍历flash page时，注册回调\ ``flash_page_cb``\来遍历所有flash page。

**参数**

* **info** 当前页信息
* **data** 回调的私有数据 

**返回值**

* **true** 继续遍历
* **false** 中止遍历

函数
----

``int flash_read(const struct device *dev, off_t offset, void *data, size_t len)`` 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
读Flash，无论那种flash对读地址和大小都没有对齐的要求。

**参数**

* **dev** Flash设备  
* **offset** Flash内的偏移地址
* **data** 存放读出数据的内存地址
* **len** 要读出数据的长度

**返回值**

0表示成功，负值表示失败。



``int flash_write(const struct device *dev, off_t offset, void *data, size_t len)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
写Flash，写地址和长度要符合Flash驱动的对齐要求。 

**参数** 

* **dev** Flash设备
* **offset** Flash内的偏移地址 
* **data** 存放写入数据的内存地址 
* **len** 要写入数据的长度 

**返回值**

0表示成功，负值表示失败。



``int int flash_erase(const struct device *dev, off_t offset, size_t size)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
擦除Flash，擦除地址和长度按Flash硬件页大小对齐，当配置\ ``CONFIG_FLASH_PAGE_LAYOUT=y``\ 的情况下可以通过\ ``flash_get_page_info_by_offs()``\ 和\ ``flash_get_page_info_by_idx()``\ 获取页大小。

**参数**

* **dev** Flash设备 
* **offset** Flash内的偏移地址 
* **len** 要擦除的长度 

**返回值** 

0表示成功，负值表示失败。



``int flash_get_page_info_by_offs(const struct device *dev, off_t offset, struct flash_pages_info *info)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
获取指定偏移地址所在页的信息。 

**参数** 

* **dev** Flash设备 
* **offset** Flash内的偏移地址 
* **info** 页信息，\ ``struct flash_pages_info``,包括页的地址，大小和索引号 

**返回值**

0表示成功，负值表示失败。



``int flash_get_page_info_by_idx(const struct device *dev, uint32_t page_index, struct flash_pages_info *info)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
获取指定索引号的页信息。 

**参数**

* **dev** Flash设备 
* **page_index** Flash内的页索引号 
* **info** 页信息，\ ``struct flash_pages_info``,包括页的地址，大小和索引号 

**返回值**
0表示成功，负值表示失败。



``int flash_get_page_count(const struct device *dev)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
获取Flash的页总数。 

**参数**

* **dev** Flash设备 

**返回值** 

Flash的页总数



``void flash_page_foreach(const struct device *dev, flash_page_cb cb, void *data)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
遍历Flash上所有的页面，从0开始每个页面执行一次cb，将页面的信息传递给cb

**参数** 

* **dev** Flash设备 
* **cb** 页面信息处理回调 
* **data** 回调函数私有参数



``int flash_sfdp_read(const struct device *dev, off_t offset, void *data, size_t len)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
读取 JEDEC JESD216，标准的SFDP数据。该API在配置了\ ``CONFIG_FLASH_JESD216_API=y``\ 才有效

**参数** 

* **dev** Flash设备 
* **offset** Flash上SFDP区内的偏移地址 
* **data** 存放读出数据的内存地址 
* **len** 要读出数据的长度 

**返回值** 

0表示成功，\ ``-ENOTSUP``\驱动不支持该功能，负值表示失败。



``int flash_read_jedec_id(const struct device *dev, uint8_t *id)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
读取Flash的JEDEC ID。该API在配置了\ ``CONFIG_FLASH_JESD216_API=y``\ 才有效 

**参数** 
* **dev** Flash设备 
* **id** 指向存储jedec id的内存，大小为3字节 

**返回值**

0表示成功，\ ``-ENOTSUP``\驱动不支持该功能，负值表示失败。



``size_t flash_get_write_block_size(const struct device *dev)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
获取驱动支持的最小写单位大小，由于驱动写算法差异该值可能Flash本身硬件规定的不一致

**参数** 

**dev** Flash设备 

**返回值** 

单位写入块大小



``const struct flash_parameters *flash_get_parameters(const struct device *dev)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
获取Flash参数，\ ``struct flash_parameters``\ 中包含了写单位大小和被擦除后默认的值。
**参数** 
* **dev** Flash设备

**返回值** 

指向flash参数

驱动实现接口
============

和Zephyr其它驱动一样，flash驱动的实现者只需要将flash.h中规定好的API实现，并进行注册，就可以通过Flash接口进行访问。参考\ `Zephyr驱动模型实现方式 <https://lgl88911.github.io/2018/08/03/Zephyr%E9%A9%B1%E5%8A%A8%E6%A8%A1%E5%9E%8B%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F/>`__
Flash的驱动实现接口定义在flash.h中，如下

.. code:: c

   //读
   typedef int (*flash_api_read)(const struct device *dev, off_t offset,
                     void *data,
                     size_t len);

   //写
   typedef int (*flash_api_write)(const struct device *dev, off_t offset,
                      const void *data, size_t len);

   //擦除
   typedef int (*flash_api_erase)(const struct device *dev, off_t offset,
                      size_t size);

   //写保护控制
   typedef int (*flash_api_write_protection)(const struct device *dev,
                         bool enable);

   //获取参数
   typedef const struct flash_parameters* (*flash_api_get_parameters)(const struct device *dev);

   //获取page layout
   typedef void (*flash_api_pages_layout)(const struct device *dev,
                          const struct flash_pages_layout **layout,
                          size_t *layout_size);

   //读取sfdp
   typedef int (*flash_api_sfdp_read)(const struct device *dev, off_t offset,
                      void *data, size_t len);

   //读取jedec id
   typedef int (*flash_api_read_jedec_id)(const struct device *dev, uint8_t *id);

   __subsystem struct flash_driver_api {
       flash_api_read read;
       flash_api_write write;
       flash_api_erase erase;
       flash_api_write_protection write_protection;
       flash_api_get_parameters get_parameters;
   #if defined(CONFIG_FLASH_PAGE_LAYOUT)
       flash_api_pages_layout page_layout;
   #endif /* CONFIG_FLASH_PAGE_LAYOUT */
   #if defined(CONFIG_FLASH_JESD216_API)
       flash_api_sfdp_read sfdp_read;
       flash_api_read_jedec_id read_jedec_id;
   #endif /* CONFIG_FLASH_JESD216_API */
   };

实现驱动时，完成以上函数指针原型的API，然后创建\ ``struct flash_driver_api``\变量，再通过\ ``DEVICE_DT_INST_DEFINE``\进行注册，就完成了Flash驱动的添加。

使用示例
========

Zephyr中Flash的操作非常容易，简单示例如下

.. code:: c

   const struct device *flash_dev = device_get_binding(DT_CHOSEN_ZEPHYR_FLASH_CONTROLLER_LABEL);
   struct flash_pages_info info;

   flash_get_page_info_by_offs(flash_dev, page_addr, &info);
   flash_erase(flash_dev, page_addr, info.size);
   flash_write(flash_dev, page_addr, buf_array, sizeof(buf_array));
   flash_read(flash_dev, page_addr, read_array, sizeof(read_array));

关于JESD216
===========

``flash_sfdp_read``\ 读出的是原始的sfdp数据，需要使用jesd216.h中的API才能解析出串行Flash的Discoverable参数。
