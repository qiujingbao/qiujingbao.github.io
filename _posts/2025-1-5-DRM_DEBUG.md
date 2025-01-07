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





