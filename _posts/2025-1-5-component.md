---
layout: post

title: component
categories: [component]
tags: [component]
typora-root-url: ..
---

```
 * This is work in progress.  We gather up the component devices into a list,
 * and bind them when instructed.  At the moment, we're specific to the DRM
 * subsystem, and only handles one master device, but this doesn't have to be
 * the case.

这是正在进行的工作。我们将组件设备收集到一个列表中，并根据指示将它们绑定。目前，我们特定于DRM子系统，并且仅处理一个主设备，但情况并非如此。
```

## debugfs

在component中有个debugfs的导出。

```

#ifdef CONFIG_DEBUG_FS

static struct dentry *component_debugfs_dir;

static int component_devices_show(struct seq_file *s, void *data)
{
	struct master *m = s->private;
	struct component_match *match = m->match;
	size_t i;

	mutex_lock(&component_mutex);
	seq_printf(s, "%-40s %20s\n", "master name", "status");
	seq_puts(s, "-------------------------------------------------------------\n");
	seq_printf(s, "%-40s %20s\n\n",
		   dev_name(m->dev), m->bound ? "bound" : "not bound");

	seq_printf(s, "%-40s %20s\n", "device name", "status");
	seq_puts(s, "-------------------------------------------------------------\n");
	for (i = 0; i < match->num; i++) {
		struct component *component = match->compare[i].component;

		seq_printf(s, "%-40s %20s\n",
			   component ? dev_name(component->dev) : "(unknown)",
			   component ? (component->bound ? "bound" : "not bound") : "not registered");
	}
	mutex_unlock(&component_mutex);

	return 0;
}

static int component_devices_open(struct inode *inode, struct file *file)
{
	return single_open(file, component_devices_show, inode->i_private);
}
/* 可以看到最终调用了component_devices_show */
static const struct file_operations component_devices_fops = {
	.open = component_devices_open,
	.read = seq_read,
	.llseek = seq_lseek,
	.release = single_release,
};

/* 可以看到注册为了device_component文件夹 */
static int __init component_debug_init(void)
{
	component_debugfs_dir = debugfs_create_dir("device_component", NULL);

	return 0;
}

/* 可以看到在core_initcall中被init */
core_initcall(component_debug_init);

/* 如果debug生效那么在component_master_add_with_match被调用，如果不生效则为空函数 */
static void component_master_debugfs_add(struct master *m)
{
	m->dentry = debugfs_create_file(dev_name(m->dev), 0444,
					component_debugfs_dir,
					m, &component_devices_fops);
}

static void component_master_debugfs_del(struct master *m)
{
	debugfs_remove(m->dentry);
	m->dentry = NULL;
}

#else

static void component_master_debugfs_add(struct master *m)
{ }

static void component_master_debugfs_del(struct master *m)
{ }

#endif
```

输出如下与component_devices_show对应。vop，hdmi，dsi被挂在到display-subsystem，display-subsystem为master。

```
root@linaro-alip:/sys/kernel/debug/device_component# cat display-subsystem  
master name                                            status
-------------------------------------------------------------
display-subsystem                                       bound

device name                                            status
-------------------------------------------------------------
fe040000.vop                                            bound
fe0a0000.hdmi                                           bound
fe070000.dsi                                            bound
```

## match

```
struct component_match *match = NULL;
component_match_add(dev, &match, compare_dev, d);

int (*compare)(struct device *, void *)
static inline void component_match_add(struct device *master,
	struct component_match **matchptr,
	int (*compare)(struct device *, void *), void *compare_data)
{
	component_match_add_release(master, matchptr, NULL, compare,
				    compare_data);
}

```

```
/*
 * Add a component to be matched, with a release function.
 *
 * The match array is first created or extended if necessary.
 */
void component_match_add_release(struct device *master,
/* 可以看到这里使用的双重指针传递，因为需要在这个里面分配空间 */
	struct component_match **matchptr,
	void (*release)(struct device *, void *),
	int (*compare)(struct device *, void *), void *compare_data)
{
/* 在这里已经拿到了调用方声明的指针 */
	struct component_match *match = *matchptr;

	if (IS_ERR(match))
		return;

	if (!match) {
	/* 分配空间 */
		match = devres_alloc(devm_component_match_release,
				     sizeof(*match), GFP_KERNEL);
		if (!match) {
			*matchptr = ERR_PTR(-ENOMEM);
			return;
		}
/* 将资源与设备绑定 可以自动释放 */
/* 可以看到是将资源放到链表里面 */
/* 	list_add_tail(&node->entry, &dev->devres_head); */
		devres_add(master, match);

		*matchptr = match;
	}

/* 如果num==allc那么表示 表示未有分配，一开始应该是0 */
	if (match->num == match->alloc) {
	/* 那么应该分配多一个的空间，这个大小是16 */
		size_t new_size = match->alloc + 16;
		int ret;
/* 分配struct component_match_array *new; */
		ret = component_match_realloc(master, match, new_size);
		if (ret) {
			*matchptr = ERR_PTR(ret);
			return;
		}
	}

/* 然后设置这些东西 */
	match->compare[match->num].compare = compare;
	match->compare[match->num].release = release;
	match->compare[match->num].data = compare_data;
	match->compare[match->num].component = NULL;
	match->num++;
}
EXPORT_SYMBOL(component_match_add_release);
```

```
static int component_match_realloc(struct device *dev,
	struct component_match *match, size_t num)
{
	struct component_match_array *new;

	if (match->alloc == num)
		return 0;

	new = kmalloc_array(num, sizeof(*new), GFP_KERNEL);
	if (!new)
		return -ENOMEM;
/* 可以看到component的方法不是通过链表分配而是重新申请一个新的数组的方式。 */
	if (match->compare) {
		memcpy(new, match->compare, sizeof(*new) *
					    min(match->num, num));
		kfree(match->compare);
	}
	match->compare = new;
	match->alloc = num;

	return 0;
}
```



## masters

```
/* 上述已经将device添加到了match中这个空间和dev关联 */
/* 然后调用如下函数 */
ret = component_master_add_with_match(dev, &rockchip_drm_ops, match);
```

```
int component_master_add_with_match(struct device *dev,
	const struct component_master_ops *ops,
	struct component_match *match)
{
	struct master *master;
	int ret;

	/* Reallocate the match array for its true size */
	ret = component_match_realloc(dev, match, match->num);
	if (ret)
		return ret;

	master = kzalloc(sizeof(*master), GFP_KERNEL);
	if (!master)
		return -ENOMEM;

	master->dev = dev;
	master->ops = ops;
	/* 看到这里match与master已经匹配起来了 */
	/* 但是这里的match不一定全 */
	master->match = match;
	/* 添加debugfs */
	component_master_debugfs_add(master);
	/* Add to the list of available masters. */
	mutex_lock(&component_mutex);
	list_add(&master->node, &masters);
/* 所以使用这个函数搜集所有满足的component */
	ret = try_to_bring_up_master(master, NULL);

	if (ret < 0)
		free_master(master);

	mutex_unlock(&component_mutex);

	return ret < 0 ? ret : 0;
}
```

```
static int try_to_bring_up_masters(struct component *component)
{
	struct master *m;
	int ret = 0;
/* 这个函数会遍历所有master然后找到component对应的master，如果对应不上 */
/*

	if (component && component->master != master) {
		dev_dbg(master->dev, "master is not for this component (%s)\n",
			dev_name(component->dev));
		return 0;
	}

*/
	list_for_each_entry(m, &masters, node) {
		if (!m->bound) {
			ret = try_to_bring_up_master(m, component);
			if (ret != 0)
				break;
		}
	}

	return ret;
}
```

```
/*
 * Try to bring up a master.  If component is NULL, we're interested in
 * this master, otherwise it's a component which must be present to try
 * and bring up the master.
 *
 * Returns 1 for successful bringup, 0 if not ready, or -ve errno.
 try_to_bring_up_master 的目的是尝试初始化或“激活”一个主设备（master）。

它会检查主设备是否具备所有必要的组件（components）。
如果传入了一个特定的组件参数，则还会验证该组件是否属于这个主设备。
在所有条件满足的情况下，调用主设备的绑定（bind）操作完成初始化。
可以看到如果component为null则初始化这个master
 */
static int try_to_bring_up_master(struct master *master,
	struct component *component)
{
	int ret;

	dev_dbg(master->dev, "trying to bring up master\n");
/* 主要是这个 */
	if (find_components(master)) {
		dev_dbg(master->dev, "master has incomplete components\n");
		return 0;
	}

	if (component && component->master != master) {
		dev_dbg(master->dev, "master is not for this component (%s)\n",
			dev_name(component->dev));
		return 0;
	}

	if (!devres_open_group(master->dev, NULL, GFP_KERNEL))
		return -ENOMEM;
/* 如果找到所有的component则调用bind函数 */
	/* Found all components */
	ret = master->ops->bind(master->dev);
	if (ret < 0) {
		devres_release_group(master->dev, NULL);
		if (ret != -EPROBE_DEFER)
			dev_info(master->dev, "master bind failed: %d\n", ret);
		return ret;
	}

	master->bound = true;
	return 1;
}
```

## component

```
int component_add(struct device *dev, const struct component_ops *ops)
{
	struct component *component;
	int ret;

	component = kzalloc(sizeof(*component), GFP_KERNEL);
	if (!component)
		return -ENOMEM;

	component->ops = ops;
	component->dev = dev;

	dev_dbg(dev, "adding component (ops %ps)\n", ops);

	mutex_lock(&component_mutex);
	list_add_tail(&component->node, &component_list);
/* 可以看到，component并不会指定master 而是放到全局链表里面*/
	ret = try_to_bring_up_masters(component);
	if (ret < 0) {
		if (component->master)
			remove_component(component->master, component);
		list_del(&component->node);

		kfree(component);
	}
	mutex_unlock(&component_mutex);

	return ret < 0 ? ret : 0;
}
EXPORT_SYMBOL_GPL(component_add);
```

```
static int find_components(struct master *master)
{
/* 已经关联起来了master与match */
	struct component_match *match = master->match;
	size_t i;
	int ret = 0;

	/*
	 * Scan the array of match functions and attach
	 * any components which are found to this master.
	 */
	for (i = 0; i < match->num; i++) {
		struct component_match_array *mc = &match->compare[i];
		struct component *c;

		dev_dbg(master->dev, "Looking for component %zu\n", i);

		if (match->compare[i].component)
			continue;
/* 如果这个component不存在那么从全局componet的链表中找到component */
		c = find_component(master, mc->compare, mc->data);
		if (!c) {
			ret = -ENXIO;
			break;
		}

		dev_dbg(master->dev, "found component %s, duplicate %u\n", dev_name(c->dev), !!c->master);

		/* Attach this component to the master */
		/* 将component与master关联起来 */
		match->compare[i].duplicate = !!c->master;
		match->compare[i].component = c;
		c->master = master;
	}
	return ret;
}
```

```
static struct component *find_component(struct master *master,
	int (*compare)(struct device *, void *), void *compare_data)
{
	struct component *c;

	list_for_each_entry(c, &component_list, node) {
		if (c->master && c->master != master)
			continue;

		if (compare(c->dev, compare_data))
			return c;
	}

	return NULL;
}
```

## 如何匹配component？

![CleanShot 2025-01-07 at 15.02.05](./assets/pics/CleanShot%202025-01-07%20at%2015.02.05.png)

## 何时调用master的bind函数？

有两个动作其实，一个添加master，一个是添加componet。其中都会到如下函数。

```
static int try_to_bring_up_masters(struct component *component)
{
	struct master *m;
	int ret = 0;

	list_for_each_entry(m, &masters, node) {
		if (!m->bound) {
			ret = try_to_bring_up_master(m, component);
			if (ret != 0)
				break;
		}
	}

	return ret;
}
```

```
/*
 * Try to bring up a master.  If component is NULL, we're interested in
 * this master, otherwise it's a component which must be present to try
 * and bring up the master.
 *
 * Returns 1 for successful bringup, 0 if not ready, or -ve errno.
 */
static int try_to_bring_up_master(struct master *master,
	struct component *component)
{
	int ret;

	dev_dbg(master->dev, "trying to bring up master\n");

	if (find_components(master)) {
		dev_dbg(master->dev, "master has incomplete components\n");
		return 0;
	}

	if (component && component->master != master) {
		dev_dbg(master->dev, "master is not for this component (%s)\n",
			dev_name(component->dev));
		return 0;
	}

	if (!devres_open_group(master->dev, NULL, GFP_KERNEL))
		return -ENOMEM;
/* 当找到所有match对应的component的时候调用master的bind */
	/* Found all components */
	ret = master->ops->bind(master->dev);
	if (ret < 0) {
		devres_release_group(master->dev, NULL);
		if (ret != -EPROBE_DEFER)
			dev_info(master->dev, "master bind failed: %d\n", ret);
		return ret;
	}

	master->bound = true;
	return 1;
}
```

## 何时调用component的bind函数？

```
int component_bind_all(struct device *master_dev, void *data)
{
	struct master *master;
	struct component *c;
	size_t i;
	int ret = 0;

	WARN_ON(!mutex_is_locked(&component_mutex));

	master = __master_find(master_dev, NULL);
	if (!master)
		return -EINVAL;

	/* Bind components in match order */
	for (i = 0; i < master->match->num; i++)
		if (!master->match->compare[i].duplicate) {
			c = master->match->compare[i].component;
			ret = component_bind(c, master, data);
			if (ret)
				break;
		}

	if (ret != 0) {
		for (; i > 0; i--)
			if (!master->match->compare[i - 1].duplicate) {
				c = master->match->compare[i - 1].component;
				component_unbind(c, master, data);
			}
	}

	return ret;
}
EXPORT_SYMBOL_GPL(component_bind_all);

static int component_bind(struct component *component, struct master *master,
	void *data)
{
	int ret;

	/*
	 * Each component initialises inside its own devres group.
	 * This allows us to roll-back a failed component without
	 * affecting anything else.
	 */
	if (!devres_open_group(master->dev, NULL, GFP_KERNEL))
		return -ENOMEM;
/* 不太清楚devres_open_group干啥的 */
	/*
	 * Also open a group for the device itself: this allows us
	 * to release the resources claimed against the sub-device
	 * at the appropriate moment.
	 */
	if (!devres_open_group(component->dev, component, GFP_KERNEL)) {
		devres_release_group(master->dev, NULL);
		return -ENOMEM;
	}

	dev_dbg(master->dev, "binding %s (ops %ps)\n",
		dev_name(component->dev), component->ops);

	ret = component->ops->bind(component->dev, master->dev, data);
	if (!ret) {
		component->bound = true;

		/*
		 * Close the component device's group so that resources
		 * allocated in the binding are encapsulated for removal
		 * at unbind.  Remove the group on the DRM device as we
		 * can clean those resources up independently.
		 */
		devres_close_group(component->dev, NULL);
		devres_remove_group(master->dev, NULL);

		dev_info(master->dev, "bound %s (ops %ps)\n",
			 dev_name(component->dev), component->ops);
	} else {
		devres_release_group(component->dev, NULL);
		devres_release_group(master->dev, NULL);

		if (ret != -EPROBE_DEFER)
			dev_err(master->dev, "failed to bind %s (ops %ps): %d\n",
				dev_name(component->dev), component->ops, ret);
	}

	return ret;
}
```

## 总结

所以说componet如何使用？

1.通过component_match_add生成match，写好比较的compare函数用于匹配component。

2.通过component_master_add_with_match生成master，并和上述match关联。

3.在驱动里面通过componet_add添加componet。

4.调用component_bind_all，一般是在master的ops的bind函数。

```
https://www.cnblogs.com/zyly/p/17678424.html
https://blog.csdn.net/shikivs/article/details/103591971
https://blog.csdn.net/bulinsheng/article/details/108680019
https://www.kernel.org/doc/html/v6.1/driver-api/component.html
```

