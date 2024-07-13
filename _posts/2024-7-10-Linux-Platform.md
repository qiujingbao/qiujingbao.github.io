---
layout: post

title: Linux Kernel Platform总线
typora-root-url: ..
---

## platform总线

​	上述也说很少直接使用device与driver，下面是platform包装的。

涉及文件：

```
drivers/of/platform.c
include/linux/platform_device.h
drivers/base/platform.c
```

**platform_driver**

```
struct platform_driver {
	/* probe函数 可以看到传入platform_device */
	int (*probe)(struct platform_device *);

	/* other callback */
	
	/*实际驱动*/
	struct device_driver driver;
	const struct platform_device_id *id_table;
	bool prevent_deferred_probe;
	/* dma */
};
```

**platform_device**

```

struct platform_device {
	const char	*name;
	int		id;
	/* 自动设置id */
	bool		id_auto;
	/* 实际device */
	struct device	dev;

	/* 资源 */
	struct resource	*resource;

	/* platform_device_id */
	const struct platform_device_id	*id_entry;

	 /*强制匹配的司机名称。不要直接设置，因为核心会释放它。使用drive_set_override（）来设置或清除它。*/
	const char *driver_override;

	/* MFD cell pointer */
	/* arch specific additions */
};

```

```
int platform_device_register(struct platform_device *pdev)
{
...
	return platform_device_add(pdev);
}

int platform_device_add(struct platform_device *pdev)
{
...
	ret = device_add(dev);
}
EXPORT_SYMBOL_GPL(platform_device_add);
```

```
int __platform_register_drivers(struct platform_driver * const *drivers,
				unsigned int count, struct module *owner);
void platform_unregister_drivers(struct platform_driver * const *drivers,
				 unsigned int count);

#define platform_register_drivers(drivers, count) \
	__platform_register_drivers(drivers, count, THIS_MODULE)


int __platform_register_drivers(struct platform_driver * const *drivers,
				unsigned int count, struct module *owner)
{
	unsigned int i;
	int err;

	for (i = 0; i < count; i++) {
...
		err = __platform_driver_register(drivers[i], owner);
...
	}
}

int __platform_driver_register(struct platform_driver *drv,
				struct module *owner)
{
...
	return driver_register(&drv->driver);
}
EXPORT_SYMBOL_GPL(__platform_driver_register);
```

**上述是device层次上看，以下是platform层次。**

​	(1)先插上USB设备并挂到总线中，然后在安装USB驱动程序过程中从总线上遍历各个设备，看驱动程序是否与其相匹配，如果匹配就将两者邦定。这就是platform_driver_register()->driver_register
​	(2)先安装USB驱动程序，然后当有USB设备插入时，那么就遍历总线上的各个驱动，看两者是否匹配，如果匹配就将其绑定。这就是platform_device_register()->device_add()

## device加载时机

​	从上文知道通过`device_register`进行设备注册，而`device_register`会调用`device_add`将设备放到总线上然后`probe`。那么device在什么时候被注册？

上述是dtb解析的过程，那么`device_node`如何转变为`platform_device`？

```
#0  of_platform_default_populate_init ()
    at drivers/of/platform.c:521
#1  0xffffffe0002000ea in do_one_initcall (
    fn=0xffffffe000020b80 <of_platform_default_populate_init>) at init/main.c:1217
#2  0xffffffe000002d8a in do_initcall_level (
    command_line=<optimized out>,
    level=<optimized out>) at init/main.c:1290
#3  do_initcalls () at init/main.c:1306
#4  do_basic_setup () at init/main.c:1326
#5  kernel_init_freeable () at init/main.c:1528
#6  0xffffffe0008ead72 in kernel_init (unused=0x0)
    at init/main.c:1415
#7  0xffffffe0002012d0 in handle_exception ()
    at arch/riscv/kernel/entry.S:219
```

```
 of_platform_default_populate_init()
                                        |
                            of_platform_default_populate();
                                        |
                            of_platform_populate();
                                        |
                            of_platform_bus_create()
                _____________________|_________________
                |                                      |
        of_platform_device_create_pdata()       of_platform_bus_create()
        _________________|____________________
       |                                      |
 of_device_alloc()                        of_device_add()      
```

**总结：**

​	设备驱动模型是linux kernel用来描述设备，驱动，总线的一个抽象的数据模型。用以解决以前混乱局面的问题。现在被广泛应用在操作系统中（RTT ，[Zephyr](https://zephyrproject.org/)）。

```
qemu-system-riscv64 \
        -nographic -machine virt \
        -kernel  /home/qiu/riscv/linux-5.10.42/arch/riscv/boot/Image \
        -initrd  /home/qiu/riscv/busybox-1.33.1/rootfs.img  \
        -append "root=/dev/ram rdinit=/sbin/init nokasrl" \
        -s -S
```

