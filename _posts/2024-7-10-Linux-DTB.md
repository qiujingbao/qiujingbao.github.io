---
layout: post

title: 设备树
typora-root-url: ..
---

## DTB加载过程

##### 涉及文件

```
drivers/of/fdt.c
arch/arm/kernel/setup.c
init/main.c
```

#### 启动流程简要分析

```
汇编部分省略。
从start_kernel开始。
void start_kernel(void)
{
...
	setup_arch(&command_line);
...
	/* Do the rest non-__init'ed, we're now alive */
	rest_init();
...
}
```

#### dtb如何传递到kernel？或者是kernel如何确定设备信息

#### **如何确定设备信息**

```
r0一般设置为0；
r1一般设置为machine id (在使用设备树时该参数没有被使用)；
r2一般设置ATAGS或DTB的开始地址；
```

（1）传统方式：对于如何确定mdesc，旧的方法是静态定义若干的machine描述符(struct machine_desc)，在系统启动的时候，通过machine type ID作为索引，在这些静态定义的machine描述符中，找到对应哪个ID匹配的描述符。
（2）设备树：通过__atags_pointer来找到对应的machine_desc设备描述符
const struct machine_desc *machine_desc __initdata;

​	两种方式都是由确定。正如他的描述一样。

> 如果dtb被传递给内核，则使用它来选择正确的machine_desc并设置系统。
>
> If a dtb was passed to the kernel, then use it to choose the correct machine_desc and to setup the system.

```
struct machine_desc {
...
    unsigned int        nr;        /* architecture number    */
    const char        *name;        /* architecture name    */
    unsigned long        atag_offset;    /* tagged list (relative) */
    const char *const     *dt_compat;    /* array of device tree
...
}
```

#### **dtb加载过程具体实现**

> 在早期启动过程中，架构设置代码通过不同的辅助回调函数多次调用
>
> of_scan_flat_dt()来解析设备树数据，然后进行分页设置。of_scan_flat_dt()
>
> 代码扫描设备树，并使用辅助函数来提取早期启动期间所需的信息。通常情况下，
>
> early_init_dt_scan_chosen()辅助函数用于解析所选节点，包括内核参数，
>
> early_init_dt_scan_root()用于初始化DT地址空间模型，early_init_dt_scan_memory()
>
> 用于确定可用RAM的大小和位置。
>
> 在ARM上，函数setup_machine_fdt()负责在选择支持板子的正确machine_desc
>
> 后，对设备树进行早期扫描。
>
> Documentation/translations/zh_CN/devicetree/usage-model.rst

#### **ARM实现**	

主要依靠这个函数setup_machine_fdt，此函数由arch自定义，用以区分设备的选择方式。

```
/* 此函数每个arch都有一个定义 */

/* 这是arm处理器的定义在此返回mdesc，在下面扫描 */
void __init setup_arch(char **cmdline_p)
{
...
	setup_processor();
	if (atags_vaddr) {
		mdesc = setup_machine_fdt(atags_vaddr);
...
	/* 注意此变量是个全局变量 如下定义*/
	machine_desc = mdesc;
...
	unflatten_device_tree();
...
}

const struct machine_desc *machine_desc __initdata;
```

##### fdt加载

​	如果是fdt传入则调用unflatten_device_tree进行解析。

```
void __init setup_arch(char **cmdline_p)
{
...
		mdesc = setup_machine_fdt(atags_vaddr);
...
	unflatten_device_tree();
}
```

##### 静态定义

​	如果不是fdt文件则继续向下进行到arch_install的时候通过静态描述的回调函数进行device加载。如下是静态的实例。

```
DT_MACHINE_START(sam9x60_dt, "Microchip SAM9X60")
	/* Maintainer: Microchip */
	.init_machine	= sam9x60_init,
	.dt_compat	= sam9x60_dt_board_compat,
MACHINE_END

static void __init sam9x60_init(void)
{
	/* 完成dtb的解析 */
	of_platform_default_populate(NULL, NULL, NULL);

	sam9x60_pm_init();
}
```

​	调用过程，我qemu模拟的是riscv，没有arm所以只能一点一点看了。

```
可以看到如下函数返回的是struct machine_desc
const struct machine_desc * __init setup_machine_fdt(void *dt)
{
	const struct machine_desc *mdesc;
	if (!early_init_dt_scan(dt))
		return NULL;
...
	/* 此函数属于/driver/of/fdt.c */
	mdesc = of_flat_dt_match_machine(NULL, arch_get_next_mach);
...
}

static noinline void __ref __noreturn rest_init(void)
{
...
		pid = user_mode_thread(kernel_init, NULL, CLONE_FS);
...
}


static noinline void __init kernel_init_freeable(void)
{
...
	do_basic_setup();
...
}

static void __init do_basic_setup(void)
{
...
	do_initcalls();
}

static void __init do_initcalls(void)
{
	for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++) {
		/* Parser modifies command_line, restore it each time */
		strcpy(command_line, saved_command_line);
		do_initcall_level(level, command_line);
	}
}
其中在arch_install中定义了。
static int __init customize_machine(void)
{
	/*
	 * customizes platform devices, or adds new ones
	 * On DT based machines, we fall back to populating the
	 * machine from the device tree, if no callback is provided,
	 * otherwise we would always need an init_machine callback.
	 */
	if (machine_desc->init_machine)
		machine_desc->init_machine();

	return 0;
}
arch_initcall(customize_machine);
```

​	如果定义了init_machine就应该走这一条路径，但是搜索kernel最新代码发现，很多cpu都不会使用这种方式了。

##### 	

#### 	riscv实现

```
/* 而在riscv中 */
void __init setup_arch(char **cmdline_p)
{
	parse_dtb();
	setup_initial_init_mm(_stext, _etext, _edata, _end);
...
	/* arm通过这个函数完成了device tree解析 */
	unflatten_device_tree();
...
}

static void __init parse_dtb(void)
{
	/* Early scan of device tree from init memory */
	/* arm也是在setup_machine_fdt中调用了这个函数 */
	if (early_init_dt_scan(dtb_early_va)) {
		const char *name = of_flat_dt_get_machine_name();

		if (name) {
			pr_info("Machine model: %s\n", name);
			dump_stack_set_arch_desc("%s (DT)", name);
		}
	} else {
		pr_err("No DTB passed to the kernel\n");
	}
}
```

```
#0  unflatten_device_tree () at drivers/of/fdt.c:1231
#1  0xffffffe000004516 in setup_arch () at arch/riscv/kernel/setup.c:88
#2  0xffffffe00000279c in start_kernel () at init/main.c:870
#3  0xffffffe00000208e in _start_kernel () at arch/riscv/kernel/head.S:281
```

```
arch/arc/include/asm/mach_desc.h
arch/arm/include/asm/mach/arch.h
```

#### dtb注册device

​	上述给了一个fdt的地址，下面是如何解析成device。

```
early_init_dt_scan()

unflatten_device_tree()
```

​	首先是`early_init_dt_scan`。

```
early_init_dt_scan
		-----early_init_dt_verify(对dtb头进行检查)
   		 	  -----early_init_dt_scan_nodes
//driver/of/fdt.c  
void __init early_init_dt_scan_nodes(void)
{
	int rc;

	/* Initialize {size,address}-cells info */
	early_init_dt_scan_root();

	/* Retrieve various information from the /chosen node */
	rc = early_init_dt_scan_chosen(boot_command_line);
	if (rc)
		pr_warn("No chosen node found, continuing without\n");

	/* Setup memory, calling early_init_dt_add_memory_arch */
	early_init_dt_scan_memory();

	/* Handle linux,usable-memory-range property */
	early_init_dt_check_for_usable_mem_range();
}

```

​	此函数的作用：

1. 从设备树中读取chosen节点的信息,包括命令行boot_command_line,initrd location及size
2. 得到根节点的{size,address}-cells信息
3. 读出设备树的系统内存设置

`unflatten_device_tree`

```
/**
 * unflatten_device_tree - create tree of device_nodes from flat blob
 *
 * unflattens the device-tree passed by the firmware, creating the
 * tree of struct device_node. It also fills the "name" and "type"
 * pointers of the nodes so the normal device-tree walking functions
 * can be used.
 */
void __init unflatten_device_tree(void)
{
	/* 如果是ACPI模式 不使用bootloader传来的dtb*/

	/*
	 *当ACPI已启用或引导加载程序未提供空根节点时，填充空根节点。
	 */
	if (!fdt) {
...
	}
	__unflatten_device_tree(fdt, NULL, &of_root,
				early_init_dt_alloc_memory_arch, false);

	/*获取指向“/scholed”和“/aliages”节点的指针，以便在任何地方使用*/
	of_alias_scan(early_init_dt_alloc_memory_arch);

	/* unittes */
	unittest_unflatten_overlay_base();
}
```

```
/**
create tree of device_nodes from flat blob
 */
void *__unflatten_device_tree(const void *blob,
			      struct device_node *dad,
			      struct device_node **mynodes,
			      void *(*dt_alloc)(u64 size, u64 align),
			      bool detached)
{
	/* First pass, scan for size */
	/* 第一次调用 因为参数所以返回size 从fdt分配并填充设备节点 */
	size = unflatten_dt_nodes(blob, NULL, dad, NULL);
	if (size <= 0)
		return NULL;
	/* 对齐 */
	size = ALIGN(size, 4);
	pr_debug("  size is %d, allocating...\n", size);

	/* 分配空间用于展开树 */
	mem = dt_alloc(size + 4, __alignof__(struct device_node));
...

	/*第二次调用 因为分配好了空间  所以是真正的展开 */
	ret = unflatten_dt_nodes(blob, mem, dad, mynodes);
...
	return mem;
}

/* 从fdt中展开device tree */
static int unflatten_dt_nodes(const void *blob,
			      void *mem,
			      struct device_node *dad,
			      struct device_node **nodepp)
{

	/*
	如果@dad有效，我们将取消对设备子树的渲染。在第一深度级别中可能存在多个节点。我们需要将@depth设置为1，以使fdt_next_node（）满意，因为当发现负@depth时，它会立即返回。否则，设备
除了第一个节点之外，其他节点都不会成功地取消填充。
	*/
	if (dad)
		depth = initial_depth = 1;

	root = dad;
	nps[depth] = dad;

	for (offset = 0;
	     offset >= 0 && depth >= initial_depth;
	     offset = fdt_next_node(blob, offset, &depth)) {
...
		if (!IS_ENABLED(CONFIG_OF_KOBJ) &&
		    !of_fdt_device_is_available(blob, offset))
			continue;

		ret = populate_node(blob, offset, &mem, nps[depth],
				   &nps[depth+1], dryrun);
...
	}
...
}

```

​	unflatten_device_tree() 函数是为了解析dtb文件，DTS节点信息被解析出来，将DTB转换成节点是device_node的树状结构     

 	of_alias_scan() 是在设置内核输出终端，以及遍历“/aliases”节点下的所有的属性并挂入相应链表.

​	那么在执行完以上三个函数之后，dts将会在被展开成树状结构分布在内存中，等待后续的devices 和 driver 注册的时候再来调用。以下是他们的关系图

下面是大佬总结出来的图

```
setup_arch()
    |
    |------> setup_machine_fdt(__fdt_pointer) 
        |  输入设备树(DTB)首地址
        |
        |------> fixmap_remap_fdt()
        |      为fdt建立地址映射，在该函数的最后，顺便就调用memblock_reserve 保留了该段内存
        |
        |------> early_init_dt_scan()
            |------> early_init_dt_verify()
            |       检验头部，判断设备树有效性并且设置设备树的指针
            |
            |------> early_init_dt_scan_nodes()
                |
                |---->of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line)
                |     从设备树中读取chosen节点的信息,包括命令行 boot_command_line,initrd location及size
                |
                |----> of_scan_flat_dt(early_init_dt_scan_root, NULL)
                |     得到根节点的{size,address}-cells信息
                |
                |----> of_scan_flat_dt(early_init_dt_scan_memory, NULL)
                |     读出设备树的系统内存设置
                |
    |--------> arm64_memblock_init()
        |
        |------> early_init_fdt_scan_reserved_mem()
            |  分析dts中的节点，从而进行保留内存的动作
            |
            |------> __fdt_scan_reserved_mem()
                |  解析reserved-memory节点的内存
                |
                |----> status = of_get_flat_dt_prop(node, "status", NULL)
                |     获取设备树属性status，如果是okay则向下
                |
                |----> err = __reserved_mem_reserve_reg(node, uname)
                |     判断reg属性，如果没有此节点则保留该节点；
                |     继续扫描下一个节点节点
                |
            |------> fdt_init_reserved_mem()
            |      预留reserved-memory节点的内存
            |
    |--------> unflatten_device_tree()
        |     解析dtb文件，将DTB转换成节点是device_node的树状结构
        |
        |----> __unflatten_device_tree()
        |      第一次scan是为了得到保存所有node和property所需要的内存size；
        |      第二次调用才是具体填充每一个struct device_node和struct property结构体
        |
        |----> of_alias_scan()
        |    设置内核输出终端，以及遍历“/aliases”节点下的所有的属性并挂入相应链表
        |    
   |         
   |    在执行完unflatten_device_tree()后，DTS节点信息被解析出来，保存到allnodes链表中，allnodes在后面会被用到
   |
```

![img](./assets/pics/o_211223063913_dts_probe-20240712191318434.jpg)

https://www.cnblogs.com/lyndonlu/articles/15723743.html

####  总结

​	此段过程中，kernel首先根据bootarg的参数选择正确的device加载过程，然后通过unflatten_device_tree将一个blob文件转为内核中一棵由结构体构成的device tree。





