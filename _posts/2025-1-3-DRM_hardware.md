---
layout: post

title: DRM-Hardware
categories: [DRM]
tags: [DRM,hardware,dts]
typora-root-url: ..
---

本章内容为drm子系统管理的硬件以及与软件间的关系。一开始的简介里面有简单的介绍，当时只是简单知道这个概念，现在在阅读plane的源代码的时候发现很多概念里不清楚。故而探讨软件硬件之间的关系，加深理解这个流程。很多基础概念来自于何小龙大佬的博客。

## Objects

在开始编写 DRM 驱动程序之前，我有必要对 DRM 内部的 Objects 进行一番介绍。因为这些 Objects 是 DRM 框架的核心，它们缺一不可。

![在这里插入图片描述](./assets/pics/dcaf498811af9864c547cf527566c572.png)

上图蓝色部分则是对物理硬件的抽象，黄色部分则是对软件的抽象。虚线以上的为 drm_mode_object，虚线以下为 drm_gem_object。
这里的意思是，暂且理解为，黄色部分并不作为驱动在这个系统中，类似于脚手架为了完成目的。然后蓝色的部分相当于硬件，为什么crtc和plane在一起，而connector与encoder在一起，请看下图。

![在这里插入图片描述](./assets/pics/6bd2e8d60e47903494494177e56938d4.png)

可以看到如上图所示，crtc与plane作为display controller这个硬件的驱动被加载，然后connector与encoder作为dsi controller这个硬件的驱动加载。然后看lcd部分。

**drm_panel**
drm_panel 不属于 objects 的范畴，它只是一堆回调函数的集合。但它的存在降低了 LCD 驱动与 encoder 驱动之间的耦合度。

耦合的产生：
（1）connector 的主要作用就是获取显示参数，所以会在 LCD 驱动中去构造 connector object。但是 connector 初始化时需要 attach 上一个 encoder object，而这个 encoder object 往往是在另一个硬件驱动中生成的，为了访问该 encoder object，势必会产生一部分耦合的代码。
（2）encoder 除了扮演信号转换的角色，还担任着通知显示设备休眠唤醒的角色。因此，当 encoder 通知 LCD 驱动执行相应的 enable/disable 操作时，就一定会调用 LCD 驱动导出的全局函数，这也必然会产生一部分的耦合代码。

为了解决该耦合的问题，DRM 子系统为开发人员提供了 drm_panel 结构体，该结构体封装了 connector & encoder 对 LCD 访问的常用接口。



于是，原来的 Encoder 驱动和 LCD 驱动之间的耦合，就转变成了上图中 Encoder 驱动与 drm_panel、drm_panel 与 LCD 驱动之间的“耦合”，从而实现了 Encoder 驱动与 LCD 驱动之间的解耦合。

小技巧：

为了方便驱动程序设计，通常都将 encoder 与 connector 放在同一个驱动中初始化，即 encoder 在哪，connector 就在哪。
**上述想法正确吗？**

```
kernel/drivers/gpu/drm/mediatek/mtk_drm_crtc.c
```

```
int mtk_plane_init(struct drm_device *dev, struct drm_plane *plane,
		   unsigned long possible_crtcs, enum drm_plane_type type)
{
	int err;

	err = drm_universal_plane_init(dev, plane, possible_crtcs,
				       &mtk_plane_funcs, formats,
				       ARRAY_SIZE(formats), NULL, type, NULL);
	if (err) {
		DRM_ERROR("failed to initialize plane\n");
		return err;
	}

	drm_plane_helper_add(plane, &mtk_plane_helper_funcs);

	return 0;
}

int mtk_drm_crtc_create(struct drm_device *drm_dev,
			const enum mtk_ddp_comp_id *path, unsigned int path_len)
{
...

	for (zpos = 0; zpos < mtk_crtc->layer_nr; zpos++) {
		type = (zpos == 0) ? DRM_PLANE_TYPE_PRIMARY :
				(zpos == 1) ? DRM_PLANE_TYPE_CURSOR :
						DRM_PLANE_TYPE_OVERLAY;
		ret = mtk_plane_init(drm_dev, &mtk_crtc->planes[zpos],
				     BIT(pipe), type);
		if (ret)
			goto unprepare;
	}

...

unprepare:
	while (--i >= 0)
		clk_unprepare(mtk_crtc->ddp_comp[i]->clk);

	return ret;
}
```

根据上述代码暂且肤浅的认为在probe的时候顺便把crtc probe了。所以说何大佬的图片并没有问题，可以那么看待整个软硬件体系。

## VOP

那么rk中的vop又是什么？

数据文档中：

```
VOP2 is the display interface from memory frame buffer to display device. VOP is connected to an AHB bus through an AHB slave and AXI bus through an AXI master. The register setting is configured through the AHB slave interface and the display frame data is read through the AXI master interface.VOP2 supports the following features:
```

dt-bingding中：

```
description:
  VOP2 (Video Output Processor v2) is the display controller for the Rockchip
  series of SoCs which transfers the image data from a video memory buffer to
  an external LCD interface.
```

那么从软件中，从device tree加载到驱动加载。

```
static const struct of_device_id vop2_dt_match[] = {
	{
		.compatible = "rockchip,rk3566-vop",
		.data = &rk3566_vop,
	}, {
		.compatible = "rockchip,rk3568-vop",
		.data = &rk3568_vop,
	}, {
		.compatible = "rockchip,rk3588-vop",
		.data = &rk3588_vop
	}, {
	},
};
MODULE_DEVICE_TABLE(of, vop2_dt_match);

static int vop2_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;

	return component_add(dev, &vop2_component_ops);
}

static void vop2_remove(struct platform_device *pdev)
{
	component_del(&pdev->dev, &vop2_component_ops);
}

struct platform_driver vop2_platform_driver = {
	.probe = vop2_probe,
	.remove_new = vop2_remove,
	.driver = {
		.name = "rockchip-vop2",
		.of_match_table = vop2_dt_match,
	},
};
```

可以看到用了component_add将vop2_component_ops逐个加载。

```
const struct component_ops vop2_component_ops = {
	.bind = vop2_bind,
	.unbind = vop2_unbind,
};
```

```

static int vop2_bind(struct device *dev, struct device *master, void *data)
{
/* 获得资源分配资源 */
...
	/* vop 硬件对应 crtc */
	ret = vop2_create_crtcs(vop2);
	if (ret)
		return ret;
	
	/* vop 硬件对应 encoder */
	ret = vop2_find_rgb_encoder(vop2);
	if (ret >= 0) {
		vop2->rgb = rockchip_rgb_init(dev, &vop2->vps[ret].crtc,
					      vop2->drm, ret);
		if (IS_ERR(vop2->rgb)) {
			if (PTR_ERR(vop2->rgb) == -EPROBE_DEFER) {
				ret = PTR_ERR(vop2->rgb);
				goto err_crtcs;
			}
			vop2->rgb = NULL;
		}
	}
...
}
```

可以看到crct与plane在一层次，如上图所示。

```
/* 可以看到在创建crtc时初始化了plane */
static int vop2_create_crtcs(struct vop2 *vop2)
{
...
		ret = vop2_plane_init(vop2, win, possible_crtcs);
		if (ret) {
			drm_err(vop2->drm, "failed to init plane %s: %d\n",
				win->data->name, ret);
			return ret;
		}
	}

	for (i = 0; i < vop2_data->nr_vps; i++) {
		vp = &vop2->vps[i];

		if (!vp->crtc.port)
			continue;

		plane = &vp->primary_plane->base;

		ret = drm_crtc_init_with_planes(drm, &vp->crtc, plane, NULL,
						&vop2_crtc_funcs,
						"video_port%d", vp->id);
		if (ret) {
			drm_err(vop2->drm, "crtc init for video_port%d failed\n", i);
			return ret;
		}

		drm_crtc_helper_add(&vp->crtc, &vop2_crtc_helper_funcs);

...
}
```

可以看到在这里初始化encoder的时候，初始化了connector。可以认为他俩在一个层次对应上图。

```
struct rockchip_rgb *rockchip_rgb_init(struct device *dev,
				       struct drm_crtc *crtc,
				       struct drm_device *drm_dev,
				       int video_port)
{
...

	encoder = &rgb->encoder.encoder;
	encoder->possible_crtcs = drm_crtc_mask(crtc);

	ret = drm_simple_encoder_init(drm_dev, encoder, DRM_MODE_ENCODER_NONE);
	if (ret < 0) {
		DRM_DEV_ERROR(drm_dev->dev,
			      "failed to initialize encoder: %d\n", ret);
		return ERR_PTR(ret);
	}

	drm_encoder_helper_add(encoder, &rockchip_rgb_encoder_helper_funcs);

	if (panel) {
		bridge = drm_panel_bridge_add_typed(panel,
						    DRM_MODE_CONNECTOR_LVDS);
		if (IS_ERR(bridge))
			return ERR_CAST(bridge);
	}

	rgb->bridge = bridge;

	ret = drm_bridge_attach(encoder, rgb->bridge, NULL,
				DRM_BRIDGE_ATTACH_NO_CONNECTOR);
	if (ret)
		goto err_free_encoder;

	connector = &rgb->connector;
	connector = drm_bridge_connector_init(rgb->drm_dev, encoder);
	if (IS_ERR(connector)) {
		DRM_DEV_ERROR(drm_dev->dev,
			      "failed to initialize bridge connector: %pe\n",
			      connector);
		ret = PTR_ERR(connector);
		goto err_free_encoder;
	}

	rgb->encoder.crtc_endpoint_id = endpoint_id;

	ret = drm_connector_attach_encoder(connector, encoder);
...
}
EXPORT_SYMBOL_GPL(rockchip_rgb_init);
```

从这个驱动来看，vop是什么？vop是 display controller+dsi controller。但是肯定不仅仅那么局限。单独合并两个硬件然后取个名字，太无聊，应该是在这个基础上做了什么优化或者添加了什么功能，也就是rk认为vop比display controller+dsi controller优秀，往后再看。

然后看vop的dts，如果想要看明白vop的dts，那么先应该关注这个文档。

```
kernel/Documentation/devicetree/bindings/media/video-interfaces.txt
```

```
	display_subsystem: display-subsystem {
		compatible = "rockchip,display-subsystem";
		ports = <&vop_out>;
	};
	
vop: vop@fe040000 {
  compatible = "rockchip,rk3568-vop";
  reg = <0x0 0xfe040000 0x0 0x3000>, <0x0 0xfe044000 0x0 0x1000>;
  reg-names = "regs", "gamma_lut";
  rockchip,grf = <&grf>;
  interrupts = <0 148 4>;
  clocks = <&cru 221>, <&cru 222>, <&cru 223>, <&cru 224>, <&cru 225>;
  clock-names = "aclk_vop", "hclk_vop", "dclk_vp0", "dclk_vp1", "dclk_vp2";
  iommus = <&vop_mmu>;
  power-domains = <&power 9>;
  status = "disabled";

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

   vp1: port@1 {
    #address-cells = <1>;
    #size-cells = <0>;
    reg = <1>;

    vp1_out_dsi0: endpoint@0 {
     reg = <0>;
     /* 可以看到连接到了dsi0_in_vp1 */
     remote-endpoint = <&dsi0_in_vp1>;
    };

...

dsi0: dsi@fe060000 {
  compatible = "rockchip,rk3568-mipi-dsi";
...

  ports {
   #address-cells = <1>;
   #size-cells = <0>;

   dsi0_in: port@0 {
    reg = <0>;
    #address-cells = <1>;
    #size-cells = <0>;

    dsi0_in_vp0: endpoint@0 {
     reg = <0>;
     remote-endpoint = <&vp0_out_dsi0>;
     status = "disabled";
    };

    dsi0_in_vp1: endpoint@1 {
     reg = <1>;
     remote-endpoint = <&vp1_out_dsi0>;
     status = "disabled";
    };
   };
  };
 };

```

可以看到rk3566用的是3568的核心配置，这里并没有vop-big之类的，而是仅有一个rockchip,rk3568-vop，可能就是网上说的vop2.



```
https://www.cnblogs.com/zyly/p/17778173.html#_label0
https://www.cnblogs.com/zyly/p/17686403.html#_label1_0
https://blog.csdn.net/u012080932/article/details/114385873
```

## mipi

TODO

### DPHY？TODO

