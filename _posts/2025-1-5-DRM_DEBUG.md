---
layout: post

title: DRM-DEBUG
categories: [DRM]
tags: [DRM,DEBUG]
typora-root-url: ..
---

上述文章只是有了个认识，但是对这块内容本身还是迷迷糊糊的。所以从两个方面来更清楚的认识，一个是驱动加载过程，一个是一条指令执行过程。

```
https://www.cnblogs.com/yaongtime/p/12940386.html
https://blog.csdn.net/czg13548930186/article/details/131132700
```

## 驱动加载过程

![CleanShot 2025-01-10 at 17.32.07](./assets/pics/CleanShot%202025-01-10%20at%2017.32.07.png)

如图，是这个驱动加载的整个过程，请先看一眼，下面如果有疑问对于该过程请回头看此图。

回到梦开始的地方，一个platform总线上的驱动的声明。

```
static struct platform_driver rockchip_drm_platform_driver = {
	.probe = rockchip_drm_platform_probe,
	.remove = rockchip_drm_platform_remove,
	.shutdown = rockchip_drm_platform_shutdown,
	.driver = {
		.name = "rockchip-drm",
		.of_match_table = rockchip_drm_dt_ids,
		.pm = &rockchip_drm_pm_ops,
	},
};
```

对应的dts。

```
 display_subsystem: display-subsystem {
  compatible = "rockchip,display-subsystem";
  memory-region = <&drm_logo>, <&drm_cubic_lut>;
  memory-region-names = "drm-logo", "drm-cubic-lut";
  ports = <&vop_out>;
  devfreq = <&dmc>;

  route {
   route_dsi0: route-dsi0 {
    status = "disabled";
    logo,uboot = "logo.bmp";
    logo,kernel = "logo_kernel.bmp";
    logo,mode = "center";
    charge_logo,mode = "center";
    connect = <&vp0_out_dsi0>;
   };
   route_dsi1: route-dsi1 {
    status = "disabled";
    logo,uboot = "logo.bmp";
    logo,kernel = "logo_kernel.bmp";
    logo,mode = "center";
    charge_logo,mode = "center";
    connect = <&vp0_out_dsi1>;
   };
...
  };
 }
```

这个驱动被加载的方式有所不同，采用了module_init这种方式。

```
static int __init rockchip_drm_init(void)
{
	int ret;

	num_rockchip_sub_drivers = 0;
#if IS_ENABLED(CONFIG_DRM_ROCKCHIP_VVOP)
	ADD_ROCKCHIP_SUB_DRIVER(vvop_platform_driver, CONFIG_DRM_ROCKCHIP_VVOP);
#else
	ADD_ROCKCHIP_SUB_DRIVER(vop_platform_driver, CONFIG_ROCKCHIP_VOP);
	ADD_ROCKCHIP_SUB_DRIVER(vop2_platform_driver, CONFIG_ROCKCHIP_VOP2);
	ADD_ROCKCHIP_SUB_DRIVER(rockchip_lvds_driver,
				CONFIG_ROCKCHIP_LVDS);
	ADD_ROCKCHIP_SUB_DRIVER(rockchip_dp_driver,
				CONFIG_ROCKCHIP_ANALOGIX_DP);
	ADD_ROCKCHIP_SUB_DRIVER(cdn_dp_driver, CONFIG_ROCKCHIP_CDN_DP);
	ADD_ROCKCHIP_SUB_DRIVER(dw_hdmi_rockchip_pltfm_driver,
				CONFIG_ROCKCHIP_DW_HDMI);
	ADD_ROCKCHIP_SUB_DRIVER(dw_mipi_dsi_driver,
				CONFIG_ROCKCHIP_DW_MIPI_DSI);
	ADD_ROCKCHIP_SUB_DRIVER(inno_hdmi_driver, CONFIG_ROCKCHIP_INNO_HDMI);
	ADD_ROCKCHIP_SUB_DRIVER(rockchip_tve_driver,
				CONFIG_ROCKCHIP_DRM_TVE);
	ADD_ROCKCHIP_SUB_DRIVER(rockchip_rgb_driver, CONFIG_ROCKCHIP_RGB);
#endif
	ret = platform_register_drivers(rockchip_sub_drivers,
					num_rockchip_sub_drivers);
	if (ret)
		return ret;

	ret = platform_driver_register(&rockchip_drm_platform_driver);
	if (ret)
		goto err_unreg_drivers;

	return 0;

err_unreg_drivers:
	platform_unregister_drivers(rockchip_sub_drivers,
				    num_rockchip_sub_drivers);
	return ret;
}
```

这个函数在干什么？

```
static struct platform_driver *rockchip_sub_drivers[MAX_ROCKCHIP_SUB_DRIVERS];
static int num_rockchip_sub_drivers;

#define ADD_ROCKCHIP_SUB_DRIVER(drv, cond) { \
	if (IS_ENABLED(cond) && \
	    !WARN_ON(num_rockchip_sub_drivers >= MAX_ROCKCHIP_SUB_DRIVERS)) \
		rockchip_sub_drivers[num_rockchip_sub_drivers++] = &drv; \
}
```

可以看到是根据config将所有驱动添加到这个数组里面。然后注册所有的驱动。当子模块match时probe被调用。

```
	ret = platform_register_drivers(rockchip_sub_drivers,
					num_rockchip_sub_drivers);
	if (ret)
		return ret;
		ret = platform_driver_register(&rockchip_drm_platform_driver);
	if (ret)
		goto err_unreg_drivers;

	return 0;
```

当这个device被match时。

```
display_subsystem: display-subsystem {
  compatible = "rockchip,display-subsystem";
```

调用如下probe函数。

```
static int rockchip_drm_platform_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct component_match *match = NULL;
	int ret;

	ret = rockchip_drm_platform_of_probe(dev);
#if !IS_ENABLED(CONFIG_DRM_ROCKCHIP_VVOP)
	if (ret)
		return ret;
#endif

	match = rockchip_drm_match_add(dev);
	if (IS_ERR(match))
		return PTR_ERR(match);

	ret = component_master_add_with_match(dev, &rockchip_drm_ops, match);
	if (ret < 0) {
		rockchip_drm_match_remove(dev);
		return ret;
	}
	dev->coherent_dma_mask = DMA_BIT_MASK(64);

	return 0;
}

```

```
static int rockchip_drm_platform_of_probe(struct device *dev)
{
	struct device_node *np = dev->of_node;
	struct device_node *port;
	bool found = false;
	int i;

	if (!np)
		return -ENODEV;
/* 死循环 */
	for (i = 0;; i++) {
		struct device_node *iommu;
/* 解析port节点，直到最后一个port解析完成跳出死循环 */
		port = of_parse_phandle(np, "ports", i);
		if (!port)
			break;

/* 为什么怎么操作？ */
/* 如果认为port是display-subsyste下面的节点，那么怎么操作确实难以理解 */
/* 观察设备树可以看到如下 */
/*
		ports = <&vop_out>;
*/
/* port是引用别的地方的，所以不确定是否port的parent是不是一个，所以每个都需要检查 */
/* 当port->parent被disable时直接跳过即可 */
/* 
vop: vop@fe040000 {
		compatible = "rockchip,rk3568-vop";
		...
		iommus = <&vop_mmu>;
		power-domains = <&power RK3568_PD_VO>;
		status = "disabled";

		vop_out: ports {
...
			vp0: port@0 {
				#address-cells = <1>;
				#size-cells = <0>;
*/
		if (!of_device_is_available(port->parent)) {
			of_node_put(port);
			continue;
		}
/* 看到这里的目的是找到iommus = <&vop_mmu>;的支持 */
		iommu = of_parse_phandle(port->parent, "iommus", 0);
		if (!iommu || !of_device_is_available(iommu->parent)) {
			DRM_DEV_DEBUG(dev,
				      "no iommu attached for %pOF, using non-iommu buffers\n",
				      port->parent);
			/*
			 * if there is a crtc not support iommu, force set all
			 * crtc use non-iommu buffer.
			 */
			 /* 如果有一个crtc不支持iommu，强制设置所有crtc使用非iommu缓冲区。 */
			is_support_iommu = false;
		}

		found = true;

		of_node_put(iommu);
		of_node_put(port);
	}

	if (i == 0) {
		DRM_DEV_ERROR(dev, "missing 'ports' property\n");
		return -ENODEV;
	}

	if (!found) {
		DRM_DEV_ERROR(dev,
			      "No available vop found for display-subsystem.\n");
		return -ENODEV;
	}

	return 0;
}
```

```
static struct component_match *rockchip_drm_match_add(struct device *dev)
{
	struct component_match *match = NULL;
	int i;
	/* num_rockchip_sub_drivers全局变量可以使用 */
	/* 这是最重要的一段代码 */
	for (i = 0; i < num_rockchip_sub_drivers; i++) {
		struct platform_driver *drv = rockchip_sub_drivers[i];
		struct device *p = NULL, *d;
		/* 首先找到这个driver例如vop2_platform_driver */
		/* 这是一个do_while循环 */
		do {
			/* 从null开始遍历所有platform总线上的device */
			/* 对每个device调用platform的match函数，注意此时并不会probe */
			/* 注意源码：		if (match(dev, data) && get_device(dev)) */
			/* 可以看到会增加该设备的引用计数，所以当找到怎么一个设备--可以匹配该driver的时候返回 */
			/* 在下面减少引用计数并p=d，意思为下次从这开始找 */
			d = bus_find_device(&platform_bus_type, p, &drv->driver,
					    (void *)platform_bus_type.match);
			put_device(p);
			p = d;

			if (!d)
				break;
			/* 建立连接，这个连接干啥用的不是很清楚 */
			device_link_add(dev, d, DL_FLAG_STATELESS);
			/* 添加到match */
			component_match_add(dev, &match, compare_dev, d);
		} while (true);
	}

	if (IS_ERR(match))
		rockchip_drm_match_remove(dev);

	return match ?: ERR_PTR(-ENODEV);
}
```

```
然后rockchip_drm_platform_probe中有这一句
ret = component_master_add_with_match(dev, &rockchip_drm_ops, match);
```

```
static const struct component_master_ops rockchip_drm_ops = {
	.bind = rockchip_drm_bind,
	.unbind = rockchip_drm_unbind,
};
```

上述已经很清楚了在rockchip_drm_init时，会把所有的driver注册上。在具体子驱动的probe中如下所示。

```
static int vop2_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;

	if (!dev->of_node) {
		DRM_DEV_ERROR(dev, "can't find vop2 devices\n");
		return -ENODEV;
	}
	return component_add(dev, &vop2_component_ops);
}

```

可以看到调用了component_add。

然后调用顺序是master->ops->bind ，然后是component->ops->bind。

目前的流程如下：

![CleanShot 2025-01-07 at 15.24.01](./assets/pics/CleanShot%202025-01-07%20at%2015.24.01.png)



那么，接下来首先先掉用rockchip_drm_bind函数。因为当子驱动probe的时候会使用component_add函数，此函数当所有就绪的时候会调用master->ops->bind函数，也就是rockchip_drm_bind。

首先是一段资源分配函数。需要对照dts看。

```
	display_subsystem: display-subsystem {
		compatible = "rockchip,display-subsystem";
		memory-region = <&drm_logo>, <&drm_cubic_lut>;
		memory-region-names = "drm-logo", "drm-cubic-lut";
		ports = <&vop_out>;
		devfreq = <&dmc>;

		route {
			route_dsi0: route-dsi0 {
				status = "disabled";
				logo,uboot = "logo.bmp";
				logo,kernel = "logo_kernel.bmp";
				logo,mode = "center";
				charge_logo,mode = "center";
				connect = <&vp0_out_dsi0>;
			};
...
	};
```

#### rockchip_drm_bind

```
static int rockchip_drm_bind(struct device *dev)
{
	struct drm_device *drm_dev;
	struct rockchip_drm_private *private;
	int ret;
	struct device_node *np = dev->of_node;
	struct device_node *parent_np;
	struct drm_crtc *crtc;
/* 分配drm_device */
	drm_dev = drm_dev_alloc(&rockchip_drm_driver, dev);
	if (IS_ERR(drm_dev))
		return PTR_ERR(drm_dev);
/* 将drm_device绑定到device上 */
	dev_set_drvdata(dev, drm_dev);

	private = devm_kzalloc(drm_dev->dev, sizeof(*private), GFP_KERNEL);
	if (!private) {
		ret = -ENOMEM;
		goto err_free;
	}

	mutex_init(&private->commit_lock);
	mutex_init(&private->ovl_lock);
	INIT_WORK(&private->commit_work, rockchip_drm_atomic_work);
	drm_dev->dev_private = private;

	private->dmc_support = false;
	/* 这段代码的意思是获取设备树上“devfreq”节点 */
	/*  可以看到有这个节点 		devfreq = <&dmc>;
  /*
 * devfreq_get_devfreq_by_phandle - Get the devfreq device from devicetree
 * @dev - instance to the given device
 * @index - index into list of devfreq
 *
 * return the instance of devfreq device
 */
  
  */
	private->devfreq = devfreq_get_devfreq_by_phandle(dev, 0);
	if (IS_ERR(private->devfreq)) {
	/* 如果有这个节点但是是defer，同样设置dmc_support */
	/* 现在设备树上没这个节点 */
		if (PTR_ERR(private->devfreq) == -EPROBE_DEFER) {
			parent_np = of_parse_phandle(np, "devfreq", 0);
			if (parent_np &&
			    of_device_is_available(parent_np)) {
				private->dmc_support = true;
				dev_warn(dev, "defer getting devfreq\n");
			} else {
				dev_info(dev, "dmc is disabled\n");
			}
		} else {
			dev_info(dev, "devfreq is not set\n");
		}
		private->devfreq = NULL;
		/* 如果就绪那么直接设为true */
	} else {
		private->dmc_support = true;
		dev_info(dev, "devfreq is ready\n");
	}
	
	/* 同样的套路设置这个hdmi的pll */
		/* 现在设备树上没这个节点 */
	private->hdmi_pll.pll = devm_clk_get(dev, "hdmi-tmds-pll");
	if (PTR_ERR(private->hdmi_pll.pll) == -ENOENT) {
		private->hdmi_pll.pll = NULL;
	} else if (PTR_ERR(private->hdmi_pll.pll) == -EPROBE_DEFER) {
		ret = -EPROBE_DEFER;
		goto err_free;
	} else if (IS_ERR(private->hdmi_pll.pll)) {
		dev_err(dev, "failed to get hdmi-tmds-pll\n");
		ret = PTR_ERR(private->hdmi_pll.pll);
		goto err_free;
	}
		/* 同样的套路设置这个default_pll */
			/* 现在设备树上没这个节点 */
	private->default_pll.pll = devm_clk_get(dev, "default-vop-pll");
	if (PTR_ERR(private->default_pll.pll) == -ENOENT) {
		private->default_pll.pll = NULL;
	} else if (PTR_ERR(private->default_pll.pll) == -EPROBE_DEFER) {
		ret = -EPROBE_DEFER;
		goto err_free;
	} else if (IS_ERR(private->default_pll.pll)) {
		dev_err(dev, "failed to get default vop pll\n");
		ret = PTR_ERR(private->default_pll.pll);
		goto err_free;
	}
/*
仅有kernel/arch/arm64/boot/dts/rockchip/rk3399.dtsi上面有这个属性
	display_subsystem: display-subsystem {
		compatible = "rockchip,display-subsystem";
		ports = <&vopl_out>, <&vopb_out>;
		clocks = <&cru PLL_VPLL>, <&cru PLL_CPLL>;
		clock-names = "hdmi-tmds-pll", "default-vop-pll";
		devfreq = <&dmc>;
	};


*/


/* 如果设备树没有对应的节点，那么应该返回的是ENOENT错误码 */
	INIT_LIST_HEAD(&private->psr_list);
	mutex_init(&private->psr_list_lock);

```

#### rockchip_drm_init_iommu

```

static int rockchip_drm_init_iommu(struct drm_device *drm_dev)
{
	struct rockchip_drm_private *private = drm_dev->dev_private;
	struct iommu_domain_geometry *geometry;
	u64 start, end;

	if (!is_support_iommu)
		return 0;
	/* 分配iommu资源 */
	/*
        Linux 的 IOMMU（Input/Output Memory Management Unit）子系统 是内核中用于管理硬件 IOMMU 的子系统。IOMMU 是一种硬件设备，主要用于在 DMA（直接内存访问）操作中进行地址转换和权限检查，提供以下关键功能：

    地址转换：

    将设备的虚拟地址（I/O 虚拟地址，简称 IOVA）映射到物理内存地址。
    使设备能够访问不连续的物理内存，同时对设备透明。
    内存保护：

    防止设备访问未授权的内存区域。
    提供 DMA 隔离，保护系统免受错误或恶意 DMA 操作的影响。
    共享虚拟内存（SVM）：

    支持设备与 CPU 共享相同的虚拟内存地址空间，简化编程模型。
	*/
	private->domain = iommu_domain_alloc(&platform_bus_type);
	if (!private->domain)
		return -ENOMEM;

	geometry = &private->domain->geometry;
	start = geometry->aperture_start;
	end = geometry->aperture_end;

	DRM_DEBUG("IOMMU context initialized (aperture: %#llx-%#llx)\n",
		  start, end);
		  /* gem中关于内存快的管理初始化，可以看到是从iommu拿到地址的范围 */
	drm_mm_init(&private->mm, start, end - start + 1);
	mutex_init(&private->mm_lock);
	/* 设置一个fault handler */
	iommu_set_fault_handler(private->domain, rockchip_drm_fault_handler,
				drm_dev);

	return 0;
}
```

#### drm_mode_config_init/rockchip_drm_mode_config_init

```
drm_mode_config_init 是 Linux DRM (Direct Rendering Manager) 子系统中用于初始化 drm_mode_config 结构的函数。这个函数是 DRM 核心框架的重要组成部分，负责设置显示硬件的模式配置结构，为驱动程序提供基础的模式设置功能。
在这个地方初始化

在这个地方进行基础的初始化
void rockchip_drm_mode_config_init(struct drm_device *dev)
{
	dev->mode_config.min_width = 0;
	dev->mode_config.min_height = 0;

	/*
	 * set max width and height as default value(4096x4096).
	 * this value would be used to check framebuffer size limitation
	 * at drm_mode_addfb().
	 */
	dev->mode_config.max_width = 8192;
	dev->mode_config.max_height = 8192;
	dev->mode_config.async_page_flip = true;

	dev->mode_config.funcs = &rockchip_drm_mode_config_funcs;
	dev->mode_config.helper_private = &rockchip_mode_config_helpers;
}
```

#### rockchip_drm_create_properties

```
创建properties
static int rockchip_drm_create_properties(struct drm_device *dev)
{
	struct drm_property *prop;
	struct rockchip_drm_private *private = dev->dev_private;

	prop = drm_property_create_range(dev, DRM_MODE_PROP_ATOMIC,
					 "EOTF", 0, 5);
	if (!prop)
		return -ENOMEM;
	private->eotf_prop = prop;

	prop = drm_property_create_range(dev, DRM_MODE_PROP_ATOMIC,
					 "COLOR_SPACE", 0, 12);
	if (!prop)
		return -ENOMEM;
	private->color_space_prop = prop;

	prop = drm_property_create_range(dev, DRM_MODE_PROP_ATOMIC,
					 "GLOBAL_ALPHA", 0, 255);
	if (!prop)
		return -ENOMEM;
	private->global_alpha_prop = prop;

	prop = drm_property_create_range(dev, DRM_MODE_PROP_ATOMIC,
					 "BLEND_MODE", 0, 1);
	if (!prop)
		return -ENOMEM;
	private->blend_mode_prop = prop;

	prop = drm_property_create_range(dev, DRM_MODE_PROP_ATOMIC,
					 "ALPHA_SCALE", 0, 1);
	if (!prop)
		return -ENOMEM;
	private->alpha_scale_prop = prop;

	prop = drm_property_create_range(dev, DRM_MODE_PROP_ATOMIC,
					 "ASYNC_COMMIT", 0, 1);
	if (!prop)
		return -ENOMEM;
	private->async_commit_prop = prop;

	prop = drm_property_create_range(dev, DRM_MODE_PROP_ATOMIC,
					 "SHARE_ID", 0, UINT_MAX);
	if (!prop)
		return -ENOMEM;
	private->share_id_prop = prop;

	prop = drm_property_create_range(dev, DRM_MODE_PROP_ATOMIC,
					 "CONNECTOR_ID", 0, 0xf);
	if (!prop)
		return -ENOMEM;
	private->connector_id_prop = prop;

	return drm_mode_create_tv_properties(dev, 0, NULL);
}
```

然后调用bind_all函数。这个函数会调用其他的bind函数。流程如下图所示。

![CleanShot 2025-01-09 at 15.12.20](./assets/pics/CleanShot%202025-01-09%20at%2015.12.20.png)

## vop

### vop2_bind

![CleanShot 2025-01-09 at 15.28.04](./assets/pics/CleanShot%202025-01-09%20at%2015.28.04.png)

这个是win的定义，rk手册上的。然后下面是一段注释也是关于这方面。

```
/*
 * VOP2 architecture
 *
 +----------+   +-------------+
 |  Cluster |   | Sel 1 from 6
 |  window0 |   |    Layer0   |              +---------------+    +-------------+    +-----------+
 +----------+   +-------------+              |N from 6 layers|    |             |    | 1 from 3  |
 +----------+   +-------------+              |   Overlay0    |    | Video Port0 |    |    RGB    |
 |  Cluster |   | Sel 1 from 6|              |               |    |             |    +-----------+
 |  window1 |   |    Layer1   |              +---------------+    +-------------+
 +----------+   +-------------+                                                      +-----------+
 +----------+   +-------------+                               +-->                   | 1 from 3  |
 |  Esmart  |   | Sel 1 from 6|              +---------------+    +-------------+    |   LVDS    |
 |  window0 |   |   Layer2    |              |N from 6 Layers     |             |    +-----------+
 +----------+   +-------------+              |   Overlay1    +    | Video Port1 | +--->
 +----------+   +-------------+   -------->  |               |    |             |    +-----------+
 |  Esmart  |   | Sel 1 from 6|   -------->  +---------------+    +-------------+    | 1 from 3  |
 |  Window1 |   |   Layer3    |                               +-->                   |   MIPI    |
 +----------+   +-------------+                                                      +-----------+
 +----------+   +-------------+              +---------------+    +-------------+
 |  Smart   |   | Sel 1 from 6|              |N from 6 Layers|    |             |    +-----------+
 |  Window0 |   |    Layer4   |              |   Overlay2    |    | Video Port2 |    | 1 from 3  |
 +----------+   +-------------+              |               |    |             |    |   HDMI    |
 +----------+   +-------------+              +---------------+    +-------------+    +-----------+
 |  Smart   |   | Sel 1 from 6|                                                      +-----------+
 |  Window1 |   |    Layer5   |                                                      |  1 from 3 |
 +----------+   +-------------+                                                      |    eDP    |
 *                                                                                   +-----------+
 */
```



```

static int vop2_bind(struct device *dev, struct device *master, void *data)
{
	struct platform_device *pdev = to_platform_device(dev);
	const struct vop2_data *vop2_data;
	struct drm_device *drm_dev = data;
	struct vop2 *vop2;
	struct resource *res;
	size_t alloc_size;
	int ret, i;
	int num_wins = 0;
	struct device_node *vop_out_node;
	
	/* 这句代码很关键，因为下面依赖的很多数据都依赖于此 */
	/*
	const void *of_device_get_match_data(const struct device *dev)
{
	const struct of_device_id *match;

	match = of_match_device(dev->driver->of_match_table, dev);
	if (!match)
		return NULL;

	return match->data;
}
可以看到实际上是找到device对应的driver对应的match_table
struct platform_driver vop2_platform_driver = {
	.probe = vop2_probe,
	.remove = vop2_remove,
	.driver = {
		.name = "rockchip-vop2",
		.of_match_table = of_match_ptr(vop2_dt_match),
	},
};
	
	static const struct vop2_data rk3568_vop = {
	.version = VOP_VERSION(0x40, 0x15),
	.nr_vps = 3,
	.nr_mixers = 5,
	.nr_gammas = 1,
	.max_input = { 4096, 2304 },
	.max_output = { 4096, 2304 },
	.ctrl = &rk3568_vop_ctrl,
	.grf_ctrl = &rk3568_grf_ctrl,
	.axi_intr = rk3568_vop_axi_intr,
	.nr_axi_intr = ARRAY_SIZE(rk3568_vop_axi_intr),
	.vp = rk3568_vop_video_ports,
	.wb = &rk3568_vop_wb_data,
	.layer = rk3568_vop_layers,
	.nr_layers = ARRAY_SIZE(rk3568_vop_layers),
	.win = rk3568_vop_win_data,
	.win_size = ARRAY_SIZE(rk3568_vop_win_data),
};

static const struct of_device_id vop2_dt_match[] = {
	{ .compatible = "rockchip,rk3568-vop",
	  .data = &rk3568_vop },
	{},
};
实际上就是这一串东西
	
	*/
	vop2_data = of_device_get_match_data(dev);
	if (!vop2_data)
		return -ENODEV;

/* 这一部分的原始数据如下
/*
 * rk3568 vop with 2 cluster, 2 esmart win, 2 smart win.
 * Every cluster can work as 4K win or split into two win.
 * All win in cluster support AFBCD.
 *
 * Every esmart win and smart win support 4 Multi-region.
 *
 * Scale filter mode:
 *
 * * Cluster:  bicubic for horizontal scale up, others use bilinear
 * * ESmart:
 *    * nearest-neighbor/bilinear/bicubic for scale up
 *    * nearest-neighbor/bilinear/average for scale down
 *
 *
 * @TODO describe the wind like cpu-map dt nodes;
 */
static const struct vop2_win_data rk3568_vop_win_data[] = {
	{
	  .name = "Smart0-win0",
	  .phys_id = ROCKCHIP_VOP2_SMART0,
	  .base = 0x400,
	  .formats = formats_win_lite,
	  .nformats = ARRAY_SIZE(formats_win_lite),
	  .format_modifiers = format_modifiers,
	  .layer_sel_id = 3,
	  .supported_rotations = DRM_MODE_REFLECT_Y,
	  .hsu_filter_mode = VOP2_SCALE_UP_BIC,
	  .hsd_filter_mode = VOP2_SCALE_DOWN_BIL,
	  .vsu_filter_mode = VOP2_SCALE_UP_BIL,
	  .vsd_filter_mode = VOP2_SCALE_DOWN_BIL,
	  .regs = &rk3568_esmart_win_data,
	  .area = rk3568_area_data,
	  .area_size = ARRAY_SIZE(rk3568_area_data),
	  .type = DRM_PLANE_TYPE_PRIMARY,
	  .max_upscale_factor = 8,
	  .max_downscale_factor = 8,
	  .dly = { 20, 47, 41 },
	  .feature = WIN_FEATURE_MULTI_AREA,
	},
....
};


*/

/* 统计win的个数 */
	for (i = 0; i < vop2_data->win_size; i++) {
		const struct vop2_win_data *win_data = &vop2_data->win[i];

		num_wins += win_data->area_size + 1;
	}

	/* Allocate vop2 struct and its vop2_win array */
	/* 根据win的个数分配空间 */
	alloc_size = sizeof(*vop2) + sizeof(*vop2->win) * num_wins;
	vop2 = devm_kzalloc(dev, alloc_size, GFP_KERNEL);
	if (!vop2)
		return -ENOMEM;

	vop2->dev = dev;
	vop2->data = vop2_data;
	vop2->drm_dev = drm_dev;
	vop2->version = vop2_data->version;

	dev_set_drvdata(dev, vop2);
	/* 这个属性好像android用到 */
	vop2->support_multi_area = of_property_read_bool(dev->of_node, "support-multi-area");
	/* 这几个属性没有找到引用 */
	vop2->disable_afbc_win = of_property_read_bool(dev->of_node, "disable-afbc-win");
	vop2->disable_win_move = of_property_read_bool(dev->of_node, "disable-win-move");
	vop2->skip_ref_fb = of_property_read_bool(dev->of_node, "skip-ref-fb");

	/* 解析如下 */
	ret = vop2_win_init(vop2);
	if (ret)
		return ret;

/* 一些资源操作，从设备树中获取并映射 */
	res = platform_get_resource_byname(pdev, IORESOURCE_MEM, "regs");
	if (!res) {
		DRM_DEV_ERROR(vop2->dev, "failed to get vop2 register byname\n");
		return -EINVAL;
	}
	vop2->regs = devm_ioremap_resource(dev, res);
	if (IS_ERR(vop2->regs))
		return PTR_ERR(vop2->regs);
	vop2->len = resource_size(res);

	vop2->regsbak = devm_kzalloc(dev, vop2->len, GFP_KERNEL);
	if (!vop2->regsbak)
		return -ENOMEM;

	res = platform_get_resource_byname(pdev, IORESOURCE_MEM, "gamma_lut");
	if (res) {
		vop2->lut_regs = devm_ioremap_resource(dev, res);
		if (IS_ERR(vop2->lut_regs))
			return PTR_ERR(vop2->lut_regs);
	}

	vop2->grf = syscon_regmap_lookup_by_phandle(dev->of_node, "rockchip,grf");

	vop2->hclk = devm_clk_get(vop2->dev, "hclk_vop");
	if (IS_ERR(vop2->hclk)) {
		DRM_DEV_ERROR(vop2->dev, "failed to get hclk source\n");
		return PTR_ERR(vop2->hclk);
	}
	vop2->aclk = devm_clk_get(vop2->dev, "aclk_vop");
	if (IS_ERR(vop2->aclk)) {
		DRM_DEV_ERROR(vop2->dev, "failed to get aclk source\n");
		return PTR_ERR(vop2->aclk);
	}

	vop2->irq = platform_get_irq(pdev, 0);
	if (vop2->irq < 0) {
		DRM_DEV_ERROR(dev, "cannot find irq for vop2\n");
		return vop2->irq;
	}
/* 
	遍历每个ports 
			vop_out: ports {
			#address-cells = <1>;
			#size-cells = <0>;

			vp0: port@0 {
				#address-cells = <1>;
				#size-cells = <0>;
				reg = <0>;

				vp0_out_dsi0: endpoint@0 {
					reg = <0>;
					remote-endpoint = <&dsi0_in_vp0>;
				};

				vp0_out_dsi1: endpoint@1 {
					reg = <1>;
					remote-endpoint = <&dsi1_in_vp0>;
				};

				vp0_out_edp: endpoint@2 {
					reg = <2>;
					remote-endpoint = <&edp_in_vp0>;
				};

				vp0_out_hdmi: endpoint@3 {
					reg = <3>;
					remote-endpoint = <&hdmi_in_vp0>;
				};
			};
			...
			}
*/
	vop_out_node = of_get_child_by_name(dev->of_node, "ports");
	if (vop_out_node) {
		struct device_node *child;

		for_each_child_of_node(vop_out_node, child) {
			u32 plane_mask = 0;
			u32 primary_plane_phy_id = 0;
			u32 vp_id = 0;
	/* 这几个存疑，因为目前我用的代码并没有这个属性 */
	/* 网上说arch/arm64/boot/dts/rockchip/rk3588-evb.dtsi 文件里面存在但是没有找到 */
	/*
	但是在运行时如下文件夹内存在 
	root@linaro-alip:/proc/device-tree/vop@fe040000/ports/port@0# ls
'#address-cells'   endpoint@1   name      rockchip,plane-mask
 cursor-win-id     endpoint@2   phandle   rockchip,primary-plane
 endpoint@0        endpoint@3   reg      '#size-cells'
 使用hexdump rockchip,plane-mask
	0000000 0000 2a00                              
0000004
	使用hexdump
	0000000 0000 0500                              
0000004
	*/
	
			of_property_read_u32(child, "rockchip,plane-mask", &plane_mask);
			of_property_read_u32(child, "rockchip,primary-plane", &primary_plane_phy_id);
			of_property_read_u32(child, "reg", &vp_id);
/* 这个vps是 video port 有个文档称为 Video Output Processor */
			vop2->vps[vp_id].plane_mask = plane_mask;
			if (plane_mask)
				vop2->vps[vp_id].primary_plane_phy_id = primary_plane_phy_id;
			else
				vop2->vps[vp_id].primary_plane_phy_id = ROCKCHIP_VOP2_PHY_ID_INVALID;

			DRM_DEV_INFO(dev, "vp%d assign plane mask: 0x%x, primary plane phy id: %d\n",
				     vp_id, vop2->vps[vp_id].plane_mask,
				     vop2->vps[vp_id].primary_plane_phy_id);
		}
	}

	spin_lock_init(&vop2->reg_lock);
	spin_lock_init(&vop2->irq_lock);
	mutex_init(&vop2->vop2_lock);
	/* 申请irq */
	ret = devm_request_irq(dev, vop2->irq, vop2_isr, IRQF_SHARED, dev_name(dev), vop2);
	if (ret)
		return ret;

	ret = vop2_create_crtc(vop2);
	if (ret)
		return ret;
	ret = vop2_gamma_init(vop2);
	if (ret)
		return ret;
	vop2_cubic_lut_init(vop2);
	vop2_wb_connector_init(vop2);
	pm_runtime_enable(&pdev->dev);

	return 0;
}
```

![CleanShot 2025-01-09 at 15.32.57](./assets/pics/CleanShot%202025-01-09%20at%2015.32.57.png)

#### vop2_win_init

```
static int vop2_win_init(struct vop2 *vop2)
{
	const struct vop2_data *vop2_data = vop2->data;
	const struct vop2_layer_data *layer_data;
	struct drm_prop_enum_list *plane_name_list;
	struct vop2_win *win;
	struct vop2_layer *layer;
	struct drm_property *prop;
	char name[DRM_PROP_NAME_LEN];
	unsigned int num_wins = 0;
	uint8_t plane_id = 0;
	unsigned int i, j;
	/* 遍历所有win data */
	for (i = 0; i < vop2_data->win_size; i++) {
		const struct vop2_win_data *win_data = &vop2_data->win[i];
	  /* 将数据拷贝到刚刚申请的空间中 */
	  /* 为什么不能用memcpy函数直接拷贝这整片空间？ */
		win = &vop2->win[num_wins];
		win->name = win_data->name;
		win->regs = win_data->regs;
		win->offset = win_data->base;
		win->type = win_data->type;
		win->formats = win_data->formats;
		win->nformats = win_data->nformats;
		win->format_modifiers = win_data->format_modifiers;
		win->supported_rotations = win_data->supported_rotations;
		win->max_upscale_factor = win_data->max_upscale_factor;
		win->max_downscale_factor = win_data->max_downscale_factor;
		win->hsu_filter_mode = win_data->hsu_filter_mode;
		win->hsd_filter_mode = win_data->hsd_filter_mode;
		win->vsu_filter_mode = win_data->vsu_filter_mode;
		win->vsd_filter_mode = win_data->vsd_filter_mode;
		win->dly = win_data->dly;
		win->feature = win_data->feature;
		win->phys_id = win_data->phys_id;
		win->layer_sel_id = win_data->layer_sel_id;
		win->win_id = i;
		win->plane_id = plane_id++;
		win->area_id = 0;
		win->zpos = i;
		win->vop2 = vop2;

		num_wins++;
		/* 是否支持多区域窗口 */
		if (!vop2->support_multi_area)
			continue;
		/* 每片area都创建一个win结构体 */
		for (j = 0; j < win_data->area_size; j++) {
			struct vop2_win *area = &vop2->win[num_wins];
			const struct vop2_win_regs *regs = win_data->area[j];

			area->parent = win;
			area->offset = win->offset;
			area->regs = regs;
			area->type = DRM_PLANE_TYPE_OVERLAY;
			area->formats = win->formats;
			area->feature = win->feature;
			area->nformats = win->nformats;
			area->format_modifiers = win->format_modifiers;
			area->max_upscale_factor = win_data->max_upscale_factor;
			area->max_downscale_factor = win_data->max_downscale_factor;
			area->supported_rotations = win_data->supported_rotations;
			area->hsu_filter_mode = win_data->hsu_filter_mode;
			area->hsd_filter_mode = win_data->hsd_filter_mode;
			area->vsu_filter_mode = win_data->vsu_filter_mode;
			area->vsd_filter_mode = win_data->vsd_filter_mode;

			area->vop2 = vop2;
			area->win_id = i;
			area->phys_id = win->phys_id;
			area->area_id = j + 1;
			area->plane_id = plane_id++;
			area->layer_sel_id = -1;
			snprintf(name, min(sizeof(name), strlen(win->name)), "%s", win->name);
			snprintf(name, sizeof(name), "%s%d", name, area->area_id);
			area->name = devm_kstrdup(vop2->dev, name, GFP_KERNEL);
			num_wins++;
		}
	}

	vop2->registered_num_wins = num_wins;
	/* 创建layer */
	for (i = 0; i < vop2_data->nr_layers; i++) {
		layer = &vop2->layers[i];
		layer_data = &vop2_data->layer[i];
		layer->id = layer_data->id;
		layer->regs = layer_data->regs;
	}
 /* 设置plane的名字 */
	plane_name_list = devm_kzalloc(vop2->dev,
				       vop2->registered_num_wins * sizeof(*plane_name_list),
				       GFP_KERNEL);
	if (!plane_name_list) {
		DRM_DEV_ERROR(vop2->dev, "failed to alloc memory for plane_name_list\n");
		return -ENOMEM;
	}

	for (i = 0; i < vop2->registered_num_wins; i++) {
		win = &vop2->win[i];
		plane_name_list[i].type = win->plane_id;
		plane_name_list[i].name = win->name;
	}

	vop2->plane_name_list = plane_name_list;

	prop = drm_property_create_object(vop2->drm_dev,
					  DRM_MODE_PROP_ATOMIC | DRM_MODE_PROP_IMMUTABLE,
					  "SOC_ID", DRM_MODE_OBJECT_CRTC);
	vop2->soc_id_prop = prop;

	prop = drm_property_create_object(vop2->drm_dev,
					  DRM_MODE_PROP_ATOMIC | DRM_MODE_PROP_IMMUTABLE,
					  "PORT_ID", DRM_MODE_OBJECT_CRTC);
	vop2->vp_id_prop = prop;

	vop2->aclk_prop = drm_property_create_range(vop2->drm_dev, 0, "ACLK", 0, UINT_MAX);
	vop2->bg_prop = drm_property_create_range(vop2->drm_dev, 0, "BACKGROUND", 0, UINT_MAX);

	vop2->line_flag_prop = drm_property_create_range(vop2->drm_dev, 0, "LINE_FLAG1", 0, UINT_MAX);

	if (!vop2->soc_id_prop || !vop2->vp_id_prop || !vop2->aclk_prop || !vop2->bg_prop ||
	    !vop2->line_flag_prop) {
		DRM_DEV_ERROR(vop2->dev, "failed to create soc_id/vp_id/aclk property\n");
		return -ENOMEM;
	}

	return 0;
}

```



#### vop2_create_crtc

![CleanShot 2025-01-09 at 21.20.22](./assets/pics/CleanShot%202025-01-09%20at%2021.20.22.png)

```
static int vop2_create_crtc(struct vop2 *vop2)
{
	const struct vop2_data *vop2_data = vop2->data;
	struct drm_device *drm_dev = vop2->drm_dev;
	struct device *dev = vop2->dev;
	struct drm_plane *plane;
	struct drm_plane *cursor = NULL;
	struct drm_crtc *crtc;
	struct device_node *port;
	struct vop2_win *win = NULL;
	struct vop2_video_port *vp;
	const struct vop2_video_port_data *vp_data;
	uint32_t possible_crtcs;
	uint64_t soc_id;
	uint32_t registered_num_crtcs = 0;
	uint32_t plane_mask = 0;
	char dclk_name[9];
	int i = 0, j = 0, k = 0;
	int ret = 0;
	bool be_used_for_primary_plane = false;
	bool find_primary_plane = false;
	bool bootloader_initialized = false;

	/* all planes can attach to any crtc */

	/* 为什么怎么算？ */
	/* 先看这个字段的定义 */
	/*
	struct drm_plane {
	/** @dev: DRM device this plane belongs to */
	struct drm_device *dev;
	/**
	 * @possible_crtcs: pipes this plane can be bound to constructed from
	 * drm_crtc_mask()
	 */
	uint32_t possible_crtcs;
	...
	}
	
	/**
 * drm_crtc_mask - find the mask of a registered CRTC
 * @crtc: CRTC to find mask for
 *
 * Given a registered CRTC, return the mask bit of that CRTC for the
 * &drm_encoder.possible_crtcs and &drm_plane.possible_crtcs fields.
 */
static inline uint32_t drm_crtc_mask(const struct drm_crtc *crtc)
{
	return 1 << drm_crtc_index(crtc);
}
	
	
	/**
 * drm_crtc_index - find the index of a registered CRTC
 * @crtc: CRTC to find index for
 *
 * Given a registered CRTC, return the index of that CRTC within a DRM
 * device's list of CRTCs.
 */
static inline unsigned int drm_crtc_index(const struct drm_crtc *crtc)
{
	return crtc->index;
}

	可以看到每一位代表一个ctrc，那么3个ctrc就是 0x 111 也就是 0x1000-0x1 也就是8-1也就是 1<<3 -1
	*/
	possible_crtcs = (1 << vop2_data->nr_vps) - 1;

	/*
	 * We set plane_mask from dts or bootloader
	 * if all the plane_mask are zero, that means
	 * the bootloader don't initialized the vop, or
	 * something is wrong, the kernel will try to
	 * initial all the vp.
	 */
	for (i = 0; i < vop2_data->nr_vps; i++) {
		vp = &vop2->vps[i];
		if (vp->plane_mask) {
			bootloader_initialized = true;
			break;
		}
	}

	/*
	 * Create primary plane for eache crtc first, since we need
	 * to pass them to drm_crtc_init_with_planes, which sets the
	 * "possible_crtcs" to the newly initialized crtc.
	 
	 首先为每个crTC创建主平面，因为我们需要将它们传递给spel_crTC_init_with_planes，它将“possible_crtcs”设置为新初始化的crTC。
	 上文说了vps 是 video ports ，是crtc对应的硬件，所以为每个vps创建crtc
	 */
	for (i = 0; i < vop2_data->nr_vps; i++) {
		vp_data = &vop2_data->vp[i];
		vp = &vop2->vps[i];
		vp->vop2 = vop2;
		vp->id = vp_data->id;
		vp->regs = vp_data->regs;
		vp->cursor_win_id = -1;
		if (vop2->disable_win_move)
			possible_crtcs = BIT(registered_num_crtcs);

		/*
		 * we assume a vp with a zere plane_mask(set from dts or bootloader)
		 * as unused.
		 */
		if (!vp->plane_mask && bootloader_initialized)
			continue;

		if (vop2_soc_is_rk3566())
			soc_id = vp_data->soc_id[1];
		else
			soc_id = vp_data->soc_id[0];

		snprintf(dclk_name, sizeof(dclk_name), "dclk_vp%d", vp->id);
		vp->dclk = devm_clk_get(vop2->dev, dclk_name);
		if (IS_ERR(vp->dclk)) {
			DRM_DEV_ERROR(vop2->dev, "failed to get %s\n", dclk_name);
			return PTR_ERR(vp->dclk);
		}

		crtc = &vp->crtc;

		port = of_graph_get_port_by_id(dev->of_node, i);
		if (!port) {
			DRM_DEV_ERROR(vop2->dev, "no port node found for video_port%d\n", i);
			return -ENOENT;
		}
		crtc->port = port;
		of_property_read_u32(port, "cursor-win-id", &vp->cursor_win_id);

		plane_mask = vp->plane_mask;
		if (vop2_soc_is_rk3566()) {
			if ((vp->plane_mask & RK3566_MIRROR_PLANE_MASK) &&
			    (vp->plane_mask & ~RK3566_MIRROR_PLANE_MASK)) {
				plane_mask &= ~RK3566_MIRROR_PLANE_MASK;
			}
		}

		if (vp->primary_plane_phy_id >= 0) {
			win = vop2_find_win_by_phys_id(vop2, vp->primary_plane_phy_id);
			if (win) {
				find_primary_plane = true;
				win->type = DRM_PLANE_TYPE_PRIMARY;
			}
		} else {
			while (j < vop2->registered_num_wins) {
				be_used_for_primary_plane = false;
				win = &vop2->win[j];
				j++;

				if (win->parent || (win->feature & WIN_FEATURE_CLUSTER_SUB))
					continue;

				if (win->type != DRM_PLANE_TYPE_PRIMARY)
					continue;

				for (k = 0; k < vop2_data->nr_vps; k++) {
					if (win->phys_id == vop2->vps[k].primary_plane_phy_id) {
						be_used_for_primary_plane = true;
						break;
					}
				}

				if (be_used_for_primary_plane)
					continue;

				find_primary_plane = true;
				break;
			}

			if (find_primary_plane)
				vp->primary_plane_phy_id = win->phys_id;
		}

		if (!find_primary_plane) {
			DRM_DEV_ERROR(vop2->dev, "No primary plane find for video_port%d\n", i);
			break;
		}

		if (vop2_plane_init(vop2, win, possible_crtcs)) {
			DRM_DEV_ERROR(vop2->dev, "failed to init primary plane\n");
			break;
		}
		plane = &win->base;

		/* some times we want a cursor window for some vp */
		if (vp->cursor_win_id >= 0) {
			cursor = vop2_cursor_plane_init(vp, possible_crtcs);
			if (!cursor)
				DRM_WARN("failed to init cursor plane for vp%d\n", vp->id);
			else
				DRM_DEV_INFO(vop2->dev, "%s as cursor plane for vp%d\n",
					     cursor->name, vp->id);
		} else {
			cursor = NULL;
		}

		ret = drm_crtc_init_with_planes(drm_dev, crtc, plane, cursor, &vop2_crtc_funcs,
						"video_port%d", vp->id);
		if (ret) {
			DRM_DEV_ERROR(vop2->dev, "crtc init for video_port%d failed\n", i);
			return ret;
		}

		drm_crtc_helper_add(crtc, &vop2_crtc_helper_funcs);

		drm_flip_work_init(&vp->fb_unref_work, "fb_unref", vop2_fb_unref_worker);

		init_completion(&vp->dsp_hold_completion);
		init_completion(&vp->line_flag_completion);
		rockchip_register_crtc_funcs(crtc, &private_crtc_funcs);
		soc_id = vop2_soc_id_fixup(soc_id);
		drm_object_attach_property(&crtc->base, vop2->soc_id_prop, soc_id);
		drm_object_attach_property(&crtc->base, vop2->vp_id_prop, vp->id);
		drm_object_attach_property(&crtc->base, vop2->aclk_prop, 0);
		drm_object_attach_property(&crtc->base, vop2->bg_prop, 0);
		drm_object_attach_property(&crtc->base, vop2->line_flag_prop, 0);
		drm_object_attach_property(&crtc->base,
					   drm_dev->mode_config.tv_left_margin_property, 100);
		drm_object_attach_property(&crtc->base,
					   drm_dev->mode_config.tv_right_margin_property, 100);
		drm_object_attach_property(&crtc->base,
					   drm_dev->mode_config.tv_top_margin_property, 100);
		drm_object_attach_property(&crtc->base,
					   drm_dev->mode_config.tv_bottom_margin_property, 100);
		vop2_crtc_create_plane_mask_property(vop2, crtc, plane_mask);
		registered_num_crtcs++;
	}

	/*
	 * change the unused primary window to overlay window
	 */
	for (j = 0; j < vop2->registered_num_wins; j++) {
		win = &vop2->win[j];
		be_used_for_primary_plane = false;

		for (k = 0; k < vop2_data->nr_vps; k++) {
			if (vop2->vps[k].primary_plane_phy_id == win->phys_id) {
				be_used_for_primary_plane = true;
				break;
			}
		}

		if (win->type == DRM_PLANE_TYPE_PRIMARY &&
		    !be_used_for_primary_plane)
			win->type = DRM_PLANE_TYPE_OVERLAY;
	}

	/*
	 * create overlay planes of the leftover overlay win
	 * Create drm_planes for overlay windows with possible_crtcs restricted
	 */
	for (j = 0; j < vop2->registered_num_wins; j++) {
		win = &vop2->win[j];

		if (win->type != DRM_PLANE_TYPE_OVERLAY)
			continue;
		/*
		 * Only dual display(which need two crtcs) need mirror win
		 */
		if (registered_num_crtcs < 2 && vop2_is_mirror_win(win))
			continue;

		if (vop2->disable_win_move) {
			crtc = vop2_find_crtc_by_plane_mask(vop2, win->phys_id);
			if (crtc)
				possible_crtcs = drm_crtc_mask(crtc);
			else
				possible_crtcs = (1 << vop2_data->nr_vps) - 1;
		}

		ret = vop2_plane_init(vop2, win, possible_crtcs);
		if (ret)
			DRM_WARN("failed to init overlay plane %s\n", win->name);
	}

	return 0;
}

```

##### **2.1 变量初始化**

- `vop2_data`：包含 VOP2 硬件的描述信息（如支持的 VP 数量）。
- `possible_crtcs`：表示哪些 CRTC 可以与 Plane 关联，初始值是支持所有 VP 的位掩码。
- `bootloader_initialized`：检查 Bootloader 是否已经初始化了 VOP 的 `plane_mask`。

##### **2.2 检查 Bootloader 初始化状态**

```
c


复制代码
for (i = 0; i < vop2_data->nr_vps; i++) {
    vp = &vop2->vps[i];
    if (vp->plane_mask) {
        bootloader_initialized = true;
        break;
    }
}
```

- 如果 `plane_mask` 全为零，说明 Bootloader 没有初始化 VOP，内核需要手动初始化。

------

##### **2.3 为每个 VP 创建 Primary Plane 和 CRTC**

```
c


复制代码
for (i = 0; i < vop2_data->nr_vps; i++) {
    vp = &vop2->vps[i];
    ...
    if (!vp->plane_mask && bootloader_initialized)
        continue; // 跳过未初始化的 VP
```

- 遍历每个 VP，为其创建 Primary Plane 和 CRTC。
- 如果 `plane_mask` 为零且 Bootloader 已初始化，则跳过该 VP。

##### **Primary Plane 的分配**

```
c


复制代码
if (vp->primary_plane_phy_id >= 0) {
    win = vop2_find_win_by_phys_id(vop2, vp->primary_plane_phy_id);
    if (win) {
        find_primary_plane = true;
        win->type = DRM_PLANE_TYPE_PRIMARY;
    }
} else {
    while (j < vop2->registered_num_wins) {
        ...
        if (win->type != DRM_PLANE_TYPE_PRIMARY)
            continue;

        find_primary_plane = true;
        break;
    }
}
```

- 如果 `primary_plane_phy_id` 有效，则直接分配对应的窗口（Window）。
- 否则，从未被分配的窗口中查找一个作为 Primary Plane。

##### **初始化 Primary Plane**

```
c


复制代码
if (vop2_plane_init(vop2, win, possible_crtcs)) {
    DRM_DEV_ERROR(vop2->dev, "failed to init primary plane\n");
    break;
}
```

- 调用 `vop2_plane_init` 初始化 Primary Plane。
- 如果失败，记录错误并退出。

##### **创建 Cursor Plane**

```
c


复制代码
if (vp->cursor_win_id >= 0) {
    cursor = vop2_cursor_plane_init(vp, possible_crtcs);
    if (!cursor)
        DRM_WARN("failed to init cursor plane for vp%d\n", vp->id);
}
```

- 如果 `cursor_win_id` 有效，则为 VP 创建 Cursor Plane。
- Cursor Plane 是一个特殊的图层，用于显示鼠标指针或其他小型图像。

##### **创建 CRTC**

```
c


复制代码
ret = drm_crtc_init_with_planes(drm_dev, crtc, plane, cursor, &vop2_crtc_funcs,
                                "video_port%d", vp->id);
```

- 调用 DRM 框架的 `drm_crtc_init_with_planes`，将 Primary Plane 和 Cursor Plane 绑定到 CRTC。
- 设置 CRTC 的名称（如 `video_port0`）。

------

##### **2.4 剩余窗口的分配**

```
c


复制代码
for (j = 0; j < vop2->registered_num_wins; j++) {
    win = &vop2->win[j];
    ...
    if (win->type == DRM_PLANE_TYPE_PRIMARY &&
        !be_used_for_primary_plane)
        win->type = DRM_PLANE_TYPE_OVERLAY;
}
```

- 将未分配的 Primary Plane 转换为 Overlay Plane。
- Overlay Plane 是一种辅助图层，用于显示叠加内容。

------

##### **2.5 创建 Overlay Plane**

```
c


复制代码
for (j = 0; j < vop2->registered_num_wins; j++) {
    win = &vop2->win[j];

    if (win->type != DRM_PLANE_TYPE_OVERLAY)
        continue;

    ret = vop2_plane_init(vop2, win, possible_crtcs);
    if (ret)
        DRM_WARN("failed to init overlay plane %s\n", win->name);
}
```

- 遍历所有剩余的窗口，为 Overlay Plane 创建 DRM 对象。



这一部分同样可以把何小龙大佬的图放上，明确目前代码的位置。

![在这里插入图片描述](./assets/pics/6bd2e8d60e47903494494177e56938d4-6501159.png)

和rockchip的如下图，可以更清楚的展示。

![CleanShot 2025-01-10 at 17.29.56](./assets/pics/CleanShot%202025-01-10%20at%2017.29.56.png)

![CleanShot 2025-01-10 at 17.30.14](./assets/pics/CleanShot%202025-01-10%20at%2017.30.14.png)

![CleanShot 2025-01-10 at 17.31.17](./assets/pics/CleanShot%202025-01-10%20at%2017.31.17.png)



## VOP FUN/操作

TODO



## MIPI DSI

根据上图，可以看到vop这个硬件对应的软件描述已经在vop_xxx中描述了，那么VOP的下层就是各种各样的现实接口比如MIPI DSI，所以kms的encoder与connector需要这里描述，然后交给panel显示。

TODO
