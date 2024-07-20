---
layout: post

title: Linux Kernel 统一设备模型
categories: [Linux,driver]
tags: [Linux,Kernel,platform]
typora-root-url: ..
---

## Linux Kernel 统一设备模型

#### 介绍

> ​	The Linux Kernel Driver Model is a unification of all the disparate driver models that were previously used in the kernel. It is intended to augment the bus-specific drivers for bridges and devices by consolidating a set of data and operations into globally accessible data structures.
>
> ​	Traditional driver models implemented some sort of tree-like structure (sometimes just a list) for the devices they control. There wasn’t any uniformity across the different bus types.
>
> ​	The current driver model provides a common, uniform data model for describing a bus and the devices that can appear under the bus. The unified bus model includes a set of common attributes which all busses carry, and a set of common callbacks, such as device discovery during bus probing, bus shutdown, bus power management, etc.
>
> ​	The common device and bridge interface reflects the goals of the modern computer: namely the ability to do seamless device “plug and play”, power management, and hot plug. In particular, the model dictated by Intel and Microsoft (namely ACPI) ensures that almost every device on almost any bus on an x86-compatible system can work within this paradigm. Of course, not every bus is able to support all such operations, although most buses support most of those operations.
>
> ref：https://docs.kernel.org/driver-api/driver-model/overview.html

​	**Linux内核驱动程序模型是以前在内核中使用的所有不同驱动程序模型的统一。**也就是用来解决以前不同驱动各自为政的问题。

​	**通过将一组数据和操作整合到全球可访问的数据结构中去增强特定总线驱动对于bridges(特殊设备还是桥梁的意思？)和设备。**意思是提供了一个统一个抽象增强驱动对设备的访问能力。

​	**传统的驱动程序模型为其控制的设备实现了某种树状结构（有时只是一个列表）。不同类型的bus没有任何统一性。**没见过没有驱动模型如何实现的，每个设备单独开一个文件，然后具有父子关系的组织成树？

​	**当前的驱动程序模型提供了一个通用、统一的数据模型，用于描述bus和可能出现bus下的设备。统一的bus模型包括所有bus都携带的一组公共属性，以及一组公共回调，例如bus探测期间的设备发现、bus关闭、bus电源管理等。**意思是提供一个数据结构描述bus，一个数据结构描述device，一个数据结构描述bus的公共属性。然后还提供一组callback用以处理各种事件。

​	**通用设备和桥接接口反映了现代计算机的目标：即能够实现无缝设备“即插即用”、电源管理和热插拔。特别是，英特尔和微软指定的模型（即ACPI）确保了x86兼容系统上几乎任何总线上的几乎每个设备都可以在这种模式下工作。当然，并不是每个bus都能支持所有此类操作，尽管大多数bus支持大多数此类操作。**

​	下面是wowo的理解。

Linux设备模型的核心思想是（通过xxx手段，实现xxx目的）：

1. **用Device（struct device）和Device Driver（struct device_driver）两个数据结构，分别从“有什么用”和“怎么用”两个角度描述硬件设备。这样就统一了编写设备驱动的格式，使驱动开发从论述题变为填空体，从而简化了设备驱动的开发。**

2. **同样使用Device和Device Driver两个数据结构，实现硬件设备的即插即用（热拔插）。**
   在Linux内核中，只要任何Device和Device Driver具有相同的名字，内核就会执行Device Driver结构中的初始化函数（probe），该函数会初始化设备，使其为可用状态。
   而对大多数热拔插设备而言，它们的Device Driver一直存在内核中。当设备没有插入时，其Device结构不存在，因而其Driver也就不执行初始化操作。当设备插入时，内核会创建一个Device结构（名称和Driver相同），此时就会触发Driver的执行。这就是即插即用的概念。

3. **通过"Bus-->Device”类型的树状结构（见2.1章节的图例）解决设备之间的依赖，而这种依赖在开关机、电源管理等过程中尤为重要。**
   试想，一个设备挂载在一条总线上，要启动这个设备，必须先启动它所挂载的总线。很显然，如果系统中设备非常多、依赖关系非常复杂的时候，无论是内核还是驱动的开发人员，都无力维护这种关系。
   而设备模型中的这种树状结构，可以自动处理这种依赖关系。启动某一个设备前，内核会检查该设备是否依赖其它设备或者总线，如果依赖，则检查所依赖的对象是否已经启动，如果没有，则会先启动它们，直到启动该设备的条件具备为止。而驱动开发人员需要做的，就是在编写设备驱动时，告知内核该设备的依赖关系即可。

4. **使用Class结构，在设备模型中引入面向对象的概念，这样可以最大限度地抽象共性，减少驱动开发过程中的重复劳动，降低工作量。**

#### 设备

> At the lowest level, every device in a Linux system is represented by an instance of struct device. The device structure contains the information that the device model core needs to model the system. Most subsystems,however, track additional information about the devices they host. As a result, it is rare for devices to be represented by bare device structures; instead, that structure, like kobject structures, is usually embedded within a higher-level representation of the device.

​	在最底层，Linux系统中的每个设备都由struct device的一个实例表示。设备结构包含设备模型核心对系统建模所需的信息。然而，大多数子系统都会跟踪有关其托管设备的附加信息。因此，很少用裸器件结构来表示器件；相反，该结构与kobject结构一样，通常嵌入在设备的高级表示中。也就是说该结构体往往不单独使用而是嵌入到其他结构体中使用。

```
struct device {
	struct kobject kobj; /*继承kobject的特性*/
	struct device		*parent; /* 父设备 */
	struct device_private	*p; /* 私有数据 */
	const char		*init_name; /* initial name of the device */
	const struct device_type *type; /**/

	const struct bus_type	*bus;	/* 指明设备的类型详细看下面device_type */
	struct device_driver *driver;	/* 分配的驱动 */
	void		*platform_data;	/* 特定于平台的数据，设备核心不会触及它 */
	void		*driver_data;	/* 驱动程序数据，使用dev_set_drvdata/dev_get_drvdata设置和获取 */
	struct mutex		mutex;	/* 同步调用device的driver的锁 */

//dma相关

	/* arch specific additions */
	struct dev_archdata	archdata;

	struct device_node	*of_node; /* 对应设备树节点 */
	struct fwnode_handle	*fwnode; /* 固件设备节点 */


	dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	u32			id;	/* device id */

	spinlock_t		devres_lock;
	struct list_head	devres_head;

	const struct class	*class;
	const struct attribute_group **groups;	/* optional groups */
};
```

```
设备的类型“struct device”嵌入其中。类或总线可以包含不同类型的设备，如“partitions”和“disks”、“mouse”和“event”。这标识设备类型并携带类型特定信息，相当于kobject的kobj_type。
如果指定了“name”，则uevent会将其包含在DEVTYPE变量中。

struct device_type {
	const char *name;
	const struct attribute_group **groups;
	int (*uevent)(const struct device *dev, struct kobj_uevent_env *env);
	char *(*devnode)(const struct device *dev, umode_t *mode,
			 kuid_t *uid, kgid_t *gid);
	void (*release)(struct device *dev);

	const struct dev_pm_ops *pm;
};
```

##### 设备的加载

```
这是device_register（）的第2部分，尽管可以单独调用 _iff_ device_initialize（）已单独调用。这通过kobject_add（）将@dev添加到kobject层次结构中，将其添加到设备的全局和同级列表中，然后将它添加到驱动程序模型的其他相关子系统中。
对于任何设备结构，不要多次调用此例程或device_register（）。驱动程序模型核心不适用于未注册然后恢复活力的设备。（除其他外，很难保证所有对@dev前一个化身的引用都已删除。）而是分配并注册一个新的结构设备。
int device_add(struct device *dev)
{
...
	error = bus_add_device(dev); 此时将device绑定到bus上
...
}

初始化并注册dev
int device_register(struct device *dev)
{
	device_initialize(dev);
	return device_add(dev);
}
```

##### probe

##### attribute

​	展示不展开。



#### 驱动

> The device driver-model tracks all of the drivers known to the system. The main reason for this tracking is to enable the driver core to match up drivers with new devices. Once drivers are known objects within the system, however, a number of other things become possible. Device drivers can export information and configuration variables that are independent of any specific device.

​	device_driver模型跟踪系统已知的所有驱动程序。这种跟踪的主要原因是使驱动程序核心能够将驱动程序与新设备相匹配。然而，一旦驱动程序成为系统中的已知对象，许多其他事情就成为可能。设备驱动程序可以导出独立于任何特定设备的信息和配置变量。也就是这个结构体是驱动的抽象，为了能够自动match，所以使用device_driver命名。一旦driver匹配成功，便可以导出为其他的表现形式。

```
struct device_driver {
	const char		*name; 
	const struct bus_type	*bus; /* 挂载的总线 */
/* module相关 */
	bool suppress_bind_attrs;	/* 通过sys fs绑定attri */
	enum probe_type probe_type; /* probe方式的类型 如下枚举 */
enum probe_type {
	PROBE_DEFAULT_STRATEGY,
	PROBE_PREFER_ASYNCHRONOUS,
	PROBE_FORCE_SYNCHRONOUS,
};

/* match table 匹配表用这个表的字段来匹配设备 */
	const struct of_device_id	*of_match_table;
/* The ACPI match table. 不太清楚ACPI啥意思 */
	const struct acpi_device_id	*acpi_match_table;

/* some callback like probe */

/* attribute */

/* dirver的私有数据 */
	struct driver_private *p;
};

```

##### 驱动的加载

驱动有两种方式加载**静态加载**和**动态加载**。两种方式最终都会到`do_one_initcall`函数。

​	如果使用`platform`总线一般驱动由`module_platform_driver`宏定义。

​	可以看到`module`中的`fn`最终会变成如下情况，暂时到`driver_register`这个地方。

```
module_init(__driver##_init)

static int __init __driver##_init(void) \
{
	drv->driver.owner = owner;
	drv->driver.bus = &platform_bus_type;

	return driver_register(&drv->driver);
}
```

​	然后在`do_one_initcall` 中调用。

```
int __init_or_module do_one_initcall(initcall_t fn)
{
...
	do_trace_initcall_start(fn);
	ret = fn();
	do_trace_initcall_finish(fn, ret);
...
}
```

​	除了上述module_platform_driver宏之外，还有一些宏用于platform init

```
#define module_platform_driver_probe(__platform_driver, __platform_probe) \
static int __init __platform_driver##_init(void) \
{ \
	return platform_driver_probe(&(__platform_driver), \
				     __platform_probe);    \
} \
module_init(__platform_driver##_init); \
static void __exit __platform_driver##_exit(void) \
{ \
	platform_driver_unregister(&(__platform_driver)); \
} \
module_exit(__platform_driver##_exit);

#define module_platform_driver(__platform_driver) \
	module_driver(__platform_driver, platform_driver_register, \
			platform_driver_unregister)

#define platform_driver_register(drv) \
	__platform_driver_register(drv, THIS_MODULE)

int __platform_driver_register(struct platform_driver *drv,
				struct module *owner)
{
	drv->driver.owner = owner;
	drv->driver.bus = &platform_bus_type;

	return driver_register(&drv->driver);
}

#define module_driver(__driver, __register, __unregister, ...) \
static int __init __driver##_init(void) \
{ \
	return __register(&(__driver) , ##__VA_ARGS__); \
} \
module_init(__driver##_init); \
static void __exit __driver##_exit(void) \
{ \
	__unregister(&(__driver) , ##__VA_ARGS__); \
} \
module_exit(__driver##_exit);
```

​	本质上都是调用`platform_driver_probe`函数。可以看到在`platform`中可以自动的`driver_register`。`platform`更多资料请参考下面。

- 将struct device类型的变量注册到内核中时自动触发（device_register，device_add，device_create_vargs，device_create）。**device添加创建注册时**
- 将struct device_driver类型的变量注册到内核中时自动触发（driver_register） **driver注册时**
- 手动查找同一bus下的所有device_driver，如果有和指定device同名的driver，执行probe操作（device_attach）
- 手动查找同一bus下的所有device，如果有和指定driver同名的device，执行probe操作（driver_attach）
- 自行调用driver的probe接口，并在该接口中将该driver绑定到某个device结构中----即设置dev->driver（device_bind_driver）



参考链接：

https://blog.csdn.net/zxcv788999/article/details/137927923

https://www.cnblogs.com/xiaojiang1025/p/6196198.html

##### 

#### 总线

> A bus is a communication channel between the processor and an input/output device. To ensure that the model is generic, all input/output devices are connected to the processor via such a bus (even if it can be a virtual one without a physical hardware correspondent).

​	bus是在处理器与io设备之间沟通的通道。为了保证模型的通用性，所有的io设备都必须通过总线连接到处理器。这里的总线可以是物理总线，也可以是虚拟总线。

​	总线由bus_type结构表示。它包含名称、默认属性、总线的方法、PM操作和驱动程序核心的私有数据。

​	Linux内核中，有三种比较特殊的bus（或者是子系统），分别是system bus、virtual bus和platform bus。它们并不是一个实际存在的bus（像USB、I2C等），而是为了方便设备模型的抽象，而虚构的。

**system bus**是旧版内核提出的概念，用于抽象系统设备（如CPU、Timer等等）。而新版内核认为它是个坏点子，因为任何设备都应归属于一个普通的子系统（New subsystems should use plain subsystems, drivers/base/bus.c, line 1264），所以就把它抛弃了（不建议再使用，它的存在只为兼容旧有的实现）。

**virtaul bus**是一个比较新的bus，主要用来抽象那些虚拟设备，所谓的虚拟设备，是指不是真实的硬件设备，而是用软件模拟出来的设备，例如虚拟机中使用的虚拟的网络设备（有关该bus的描述，可参考该链接处的解释：https://lwn.net/Articles/326540/）。

**platform bus**就比较普通，它主要抽象集成在CPU（SOC）中的各种设备。这些设备直接和CPU连接，通过总线寻址和中断的方式，和CPU交互信息。 

```
struct bus_type {
	const char		*name;
	const char		*dev_name;
	const struct attribute_group **bus_groups;
	const struct attribute_group **dev_groups;
	const struct attribute_group **drv_groups;

	int (*match)(struct device *dev, struct device_driver *drv);
	int (*uevent)(const struct device *dev, struct kobj_uevent_env *env);
	int (*probe)(struct device *dev);
...
	const struct dev_pm_ops *pm;
};
```

​	这是对于bus的整体的抽象，也就是每个`bus`都会有如上的抽象，但是每个`bus`的细节有不一样，所以有一个`struct`表示`bus`的`private`数据

```
/**
 * struct subsys_private - structure to hold the private to the driver core portions of the bus_type/class structure.
 	这个结构体泳衣保存bus_type与class的驱动核心部分的私有数据
 * This structure is the one that is the actual kobject allowing struct
 * bus_type/class to be statically allocated safely.  Nothing outside of the
 * driver core should ever touch these fields.
 该结构是允许安全地静态分配structbus_type/class的实际kobject。驱动程序核心之外的任何东西都不应该触及这些字段。
 */
struct subsys_private {
	struct kset subsys;
	struct kset *devices_kset;
	struct list_head interfaces;
	struct mutex mutex;

	struct kset *drivers_kset;
	struct klist klist_devices; //device链表
	struct klist klist_drivers; //driver链表
	struct blocking_notifier_head bus_notifier;
	unsigned int drivers_autoprobe:1;
	const struct bus_type *bus;
	struct device *dev_root;
};
```

> wowo大佬发言：
> 按理说，这个结构就是集合了一些bus模块需要使用的私有数据，例如kset啦、klist啦等等，命名为bus_private会好点（就像device_driver模块一样）。不过为什么内核没这么做呢？看看include/linux/device.h中的struct class结构（我们会在下一篇文章中介绍class）就知道了，因为class结构中也包含了一个一模一样的struct subsys_private指针，看来class和bus很相似啊。
>
> 想到这里，就好理解了，无论是bus，还是class，还是我们会在后面看到的一些虚拟的子系统，它都构成了一个“子系统（sub-system）”，该子系统会包含形形色色的device或device_driver，就像一个独立的王国一样，存在于内核中。而这些子系统的表现形式，就是/sys/bus（或/sys/class，或其它）目录下面的子目录，每一个子目录，都是一个子系统（如/sys/bus/spi/）。

##### bus注册/注销

​	**注册**

```
/**
 * bus_register - register a driver-core subsystem
 * @bus: bus to register
 * 注册驱动程序核心子系统
 * Once we have that, we register the bus with the kobject
 * infrastructure, then register the children subsystems it has:
 * the devices and drivers that belong to the subsystem.
 一旦我们有了它，我们就用kobject基础设施注册总线，然后注册它的子系统：属于子系统的设备和驱动程序。
 */
int bus_register(const struct bus_type *bus)
{
	int retval;
	struct subsys_private *priv;
	struct kobject *bus_kobj;
	struct lock_class_key *key;
	/* 申请一个subsys_private */
	priv = kzalloc(sizeof(struct subsys_private), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;
	/* 关联subsys_private与bus */
	priv->bus = bus;

	BLOCKING_INIT_NOTIFIER_HEAD(&priv->bus_notifier);
  /* 对于 kset的kobj设置 */
	
	/* 放到全局链表中为bus_to_subsys提供基础 */	
	bus_kobj->kset = bus_kset;
	
	/* 导出到sys file */
	
	/* 初始化链表等 */

	/* 导出到文件夹 */
	retval = sysfs_create_groups(bus_kobj, bus->bus_groups);
	if (retval)
		goto bus_groups_fail;

}
```

​	**注销**

```
void bus_unregister(const struct bus_type *bus)
{
	/* 获取subsys private指针 */
	struct subsys_private *sp = bus_to_subsys(bus);

	struct kobject *bus_kobj;

	/* 配置sysfs */
	bus_kobj = &sp->subsys.kobj;
	sysfs_remove_groups(bus_kobj, bus->bus_groups);
	remove_probe_files(bus);
	bus_remove_file(bus, &bus_attr_uevent);
	/* 注销kset */
	kset_unregister(sp->drivers_kset);
	kset_unregister(sp->devices_kset);
	kset_unregister(&sp->subsys);
	subsys_put(sp);
}
```

##### **添加device**/driver

```
/**
	功能：
 * - Add device's bus attributes.  添加设备的总线属性
 * - Create links to device's bus. 创建到设备总线的链接
 * - Add the device to its bus's list of devices. 将设备添加到其总线的设备列表中。
 */
int bus_add_device(struct device *dev)
{
	/* 从device中拿出来bus然后转为private */
	struct subsys_private *sp = bus_to_subsys(dev->bus);
	int error;
	/* 如果拿不到，也就是没有bus存在 */
	/*
	对于许多没有分配总线的设备来说，这是一种正常的操作，只是说一切都很顺利。
	*/
	if (!sp) {return 0;}

  /*sp中的引用现在递增，当设备从总线上移除时将被删除*/

	/* 添加属性 创建链接 */
...
	/* 将设备添加到总线的device链表中 */
	klist_add_tail(&dev->p->knode_bus, &sp->klist_devices);
...
}
```

```
int bus_add_driver(struct device_driver *drv)
{
	struct subsys_private *sp = bus_to_subsys(drv->bus);
	struct driver_private *priv;

	/* 同上 */
	if (!sp)
		return -EINVAL;

  /*设置私有数据 注册到sysfs*/

	/* 添加到driver链表中 */
	klist_add_tail(&priv->knode_bus, &sp->klist_drivers);
	/* 如果autoprobe为true那么自动attach */
	if (sp->drivers_autoprobe) {
		error = driver_attach(drv);
		if (error)
			goto out_del_list;
	}
	/**/ 
	module_add_driver(drv->owner, drv);
	/*该函数的主要作用为更新引用计数，并创建两个符号链接：driver文件夹下的module，<module>/drivers/文件夹下的driver。*/
	/* 创建文件夹之类的操作 */
}
```

##### probe

​	上述`driver`和`device`的`probe`都是调用，实际发生`probe`的时机是`bus`完成的，因为只有`bus`才能匹配`device`于`driver`。

**device_probe**

​	在上述`device_add`的时候会调用`bus_add_device`，但是`bus_add_device`并不会自动`probe`。但是`device_add`调用`bus_probe_device`去自动`probe`。

```
int device_add(struct device *dev)
{
...
	error = bus_add_device(dev);
	if (error)
		goto BusError;
	bus_probe_device(dev);
...
}
```

**driver_probe**

```
/**
 * bus_add_driver - Add a driver to the bus.
 * @drv: driver.
 */
int bus_add_driver(struct device_driver *drv)
{
...
	klist_add_tail(&priv->knode_bus, &sp->klist_drivers);
	if (sp->drivers_autoprobe) {
		error = driver_attach(drv);
...
}

```

​	在`driver`被添加到`bus`的时候会被`probe`，会通过`driver_attach`完成。

综上：

​	`driver`在`bus_add_driver`的时候会通过`driver_attach`完成`probe`。

​	`device`在`device_add`的时候通过调用`bus_probe_device`完成`probe`。

**Bus初始化的流程**

```
start_kernel  
-> rest_init();
    -> kernel_thread(kernel_init, NULL, CLONE_FS);
        -> kernel_init()
            -> kernel_init_freeable();
                -> do_basic_setup();
                    -> driver_init();  
                        ->platform_bus_init();
```

```
spi总线
static int __init spi_init(void)
{
...
	status = bus_register(&spi_bus_type);
...
}
postcore_initcall(spi_init);
```

```
i2c总线
static int __init i2c_init(void)
{
    ...
    bus_register(&i2c_bus_type);
    ...
}
postcore_initcall(i2c_init);
```

```
int __init platform_bus_init(void)
{
    ...
    bus_register(&platform_bus_type);
    ...
}
```

```
/**
 * driver_init - initialize driver model.
 *
 * Call the driver model init functions to initialize their
 * subsystems. Called early from init/main.c.
 */
void __init driver_init(void)
{
	/* These are the core pieces */
...
	/* These are also core pieces, but must come after the
	 * core core pieces.
	 */
	of_core_init();
	platform_bus_init();
...
}

```



#### 详细流程

![img](./assets/pics/o_211223063913_dts_probe.jpg)



ref：https://www.cnblogs.com/lyndonlu/articles/15723743.html

#### **设备模型框架下驱动开发的基本步骤**


​	1.分配一个struct device类型的变量，填充必要的信息后，把它注册到内核中。

​	2.分配一个struct device_driver类型的变量，填充必要的信息后，把它注册到内核中。

​	这两步完成后，内核会在合适的时机（后面会讲），调用struct device_driver变量中的probe、remove、suspend、resume等回调函数，从而触发或者终结设备驱动的执行。而所有的驱动程序逻辑，都会由这些回调函数实现，此时，驱动开发者眼中便不再有“设备模型”，转而只关心驱动本身的实现。

**补充：**

1. 一般情况下，Linux驱动开发很少直接使用device和device_driver，因为内核在它们之上又封装了一层，如soc device、platform device等等，而这些层次提供的接口更为简单、易用（也正是因为这个原因，本文并不会过多涉及device、device_driver等模块的实现细节）。

2. 内核提供很多struct device结构的操作接口（具体可以参考include/linux/device.h和drivers/base/core.c的代码），主要包括初始化（device_initialize）、注册到内核（device_register）、分配存储空间+初始化+注册到内核（device_create）等等，可以根据需要使用。

3. device和device_driver必须具备相同的名称，内核才能完成匹配操作，进而调用device_driver中的相应接口。这里的同名，作用范围是同一个bus下的所有device和device_driver。

4. device和device_driver必须挂载在一个bus之下，该bus可以是实际存在的，也可以是虚拟的。

5. driver开发者可以在struct device变量中，保存描述设备特征的信息，如寻址空间、依赖的GPIOs等，因为device指针会在执行probe等接口时传入，这时driver就可以根据这些信息，执行相应的逻辑操作了。