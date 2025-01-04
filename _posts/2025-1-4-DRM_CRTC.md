---
layout: post

title: DRM-CRTC
categories: [DRM]
tags: [DRM,CRTC]
typora-root-url: ..
---

```
/**
 * DOC: overview
 *
 * A CRTC represents the overall display pipeline. It receives pixel data from
 * &drm_plane and blends them together. The &drm_display_mode is also attached
 * to the CRTC, specifying display timings. On the output side the data is fed
 * to one or more &drm_encoder, which are then each connected to one
 * &drm_connector.
 *
 * To create a CRTC, a KMS drivers allocates and zeroes an instances of
 * &struct drm_crtc (possibly as part of a larger structure) and registers it
 * with a call to drm_crtc_init_with_planes().
 *
 * The CRTC is also the entry point for legacy modeset operations, see
 * &drm_crtc_funcs.set_config, legacy plane operations, see
 * &drm_crtc_funcs.page_flip and &drm_crtc_funcs.cursor_set2, and other legacy
 * operations like &drm_crtc_funcs.gamma_set. For atomic drivers all these
 * features are controlled through &drm_property and
 * &drm_mode_config_funcs.atomic_check and &drm_mode_config_funcs.atomic_check.
 */

```

```
DRM 子系统中 CRTC 的概述
在 DRM（Direct Rendering Manager）子系统中，CRTC（Cathode Ray Tube Controller，显像管控制器）是显示管线的核心组件。以下是根据文档内容的详细分析：

1. CRTC 的作用
像素数据处理：
CRTC 从一个或多个 drm_plane 获取像素数据。
它将这些像素数据混合（blend）后生成最终的输出帧。
显示时序：
每个 CRTC 都关联一个 drm_display_mode，该结构定义了显示时序，例如分辨率、刷新率和同步参数。
输出到编码器：
处理后的数据被传递到一个或多个 drm_encoder，这些编码器再连接到 drm_connector，将数据输出到物理显示设备。
2. 创建和初始化
驱动的职责：
KMS（Kernel Mode Setting，内核模式设置）驱动通过分配并初始化 struct drm_crtc 来创建一个 CRTC 实例。
使用 drm_crtc_init_with_planes() 函数完成初始化。
集成方式：
CRTC 可以作为驱动程序分配的更大结构的一部分，以便支持自定义功能和扩展能力。
3. 传统操作（Legacy Operations）
模式设置（Modesetting）：
传统模式设置操作通过 drm_crtc_funcs.set_config 函数管理。
Plane 操作：
传统的 Plane 管理包括：
drm_crtc_funcs.page_flip：用于翻转显示缓冲区。
drm_crtc_funcs.cursor_set2：用于硬件光标的管理。
其他功能：
例如伽马校正（Gamma Correction）通过 drm_crtc_funcs.gamma_set 实现。
4. 原子操作（Atomic Operations）
对于使用原子 API 的现代驱动：

控制机制：
所有功能通过 drm_property 管理。
验证与应用：
使用 drm_mode_config_funcs.atomic_check 等函数验证和应用原子更新。
```

对比plane，crtc代表了整个显示管道，有数据混合，显示时序，输出到编码器的功能。而plane代表了一个图像源，含有很多参数指导这个图像如何处理，crtc是真正处理这个。

利用modetest得到如下信息

```
CRTCs:
id      fb      pos     size
85      0       (0,0)   (0x0)
   0 0 0 0 0 0 0 0 0 0 flags: ; type: 
  props:
...
        28 CUBIC_LUT:
                flags: blob
                blobs:

                value:
        29 CUBIC_LUT_SIZE:
                flags: immutable range
                values: 0 4294967295
                value: 729
115     169     (0,0)   (720x1280)
   0 720 728 744 752 1280 1288 1304 1312 59197 flags: nhsync, nvsync; type: 
  props:
        54 ACLK:
                flags: range
                values: 0 4294967295
                value: 500000
        55 BACKGROUND:
                flags: range
                values: 0 4294967295
                value: 0
...
可以看到有两个crtc，然后每个crtc都有一堆属性
Planes:
id      crtc    fb      CRTC x,y        x,y     gamma size      possible crtcs
57      0       0       0,0             0,0     0               0x00000001
71      0       0       0,0             0,0     0               0x00000001
87      115     169     0,0             0,0     0               0x00000002
101     115     164     0,0             0,0     0               0x00000002
117     0       0       0,0             0,0     0               0x00000001
  formats: XR24 AR24 XB24 AB24 RG24 BG24 RG16 BG16 NV12 NV16 NV24 NA12 NA16 NA24 YVYU VYUY

131     0       0       0,0             0,0     0               0x00000002
  formats: XR24 AR24 XB24 AB24 RG24 BG24 RG16 BG16 NV12 NV16 NV24 NA12 NA16 NA24 YVYU VYUY

```

如果想弄懂打印的什么东西如何和内核的数据结构关联起来，我们需要看libdrm源码。modetest的源码如下位置

```
https://gitlab.freedesktop.org/mesa/drm/-/blob/main/tests/modetest/modetest.c?ref_type=heads
```

```
int main(int argc, char **argv)
{
...
		case 'p':
			crtcs = 1;
			planes = 1;
...
	dump_resource(&dev, crtcs);
	dump_resource(&dev, planes);
...
}

#define dump_resource(dev, res) if (res) dump_##res(dev)
所以会被替换成如下函数
dump_crtcs(dev)
dump_planes(dev)

static void dump_crtcs(struct device *dev)
{
	int i;
	uint32_t j;

	printf("CRTCs:\n");
	printf("id\tfb\tpos\tsize\n");
	for (i = 0; i < dev->resources->count_crtcs; i++) {
	/* 对于每个crtc */
		struct crtc *_crtc = &dev->resources->crtcs[i];
		drmModeCrtc *crtc = _crtc->crtc;
		if (!crtc)
			continue;
	/* 编号 115     169     (0,0)   (720x1280) */
		printf("%d\t%d\t(%d,%d)\t(%dx%d)\n",
		       crtc->crtc_id,
		       crtc->buffer_id,
		       crtc->x, crtc->y,
		       crtc->width, crtc->height);
		dump_mode(&crtc->mode, 0);
/* 然后打印的是属性 */
		if (_crtc->props) {
			printf("  props:\n");
			for (j = 0; j < _crtc->props->count_props; j++)
				dump_prop(dev, _crtc->props_info[j],
					  _crtc->props->props[j],
					  _crtc->props->prop_values[j]);
		} else {
			printf("  no properties found\n");
		}
	}
	printf("\n");
}

static void dump_mode(drmModeModeInfo *mode, int index)
{
/*    0 720 728 744 752 1280 1288 1304 1312 59197 flags: nhsync, nvsync; type:  */
	printf("  #%i %s %.2f %d %d %d %d %d %d %d %d %d",
	       index,
	       mode->name,
	       mode_vrefresh(mode),
	       mode->hdisplay,
	       mode->hsync_start,
	       mode->hsync_end,
	       mode->htotal,
	       mode->vdisplay,
	       mode->vsync_start,
	       mode->vsync_end,
	       mode->vtotal,
	       mode->clock);

	printf(" flags: ");
	mode_flag_str(mode->flags);
	printf("; type: ");
	mode_type_str(mode->type);
	printf("\n");
}


对于plane
static void dump_planes(struct device *dev)
{
	unsigned int i, j;

	printf("Planes:\n");
	printf("id\tcrtc\tfb\tCRTC x,y\tx,y\tgamma size\tpossible crtcs\n");

	for (i = 0; i < dev->resources->count_planes; i++) {
		struct plane *plane = &dev->resources->planes[i];
		drmModePlane *ovr = plane->plane;
		if (!ovr)
			continue;

		printf("%d\t%d\t%d\t%d,%d\t\t%d,%d\t%-8d\t0x%08x\n",
		       ovr->plane_id, ovr->crtc_id, ovr->fb_id,
		       ovr->crtc_x, ovr->crtc_y, ovr->x, ovr->y,
		       ovr->gamma_size, ovr->possible_crtcs);

		if (!ovr->count_formats)
			continue;

		printf("  formats:");
		for (j = 0; j < ovr->count_formats; j++)
			dump_fourcc(ovr->formats[j]);
		printf("\n");

		if (plane->props) {
			printf("  props:\n");
			for (j = 0; j < plane->props->count_props; j++)
				dump_prop(dev, plane->props_info[j],
					  plane->props->props[j],
					  plane->props->prop_values[j]);
		} else {
			printf("  no properties found\n");
		}
	}
	printf("\n");

	return;
}
```

这里主要的是possible_crtcs，指明了plane属于的ctct，那么为什么系统中有两个crtc？

回想上述何小龙大佬的框图，crct对应的是display controller，那么这个display controller支持输出几路信号那么就应该有几个crtc（猜测），根据上述设备树VOP配置时ports有两个port所以有两个port，那么如何证明？应该在rk的vop初始代码中寻找答案。本文暂不涉及。



## obj

| **字段名**                | **类型**                               | **描述**                                                     |
| ------------------------- | -------------------------------------- | ------------------------------------------------------------ |
| `dev`                     | `struct drm_device *`                  | 父 DRM 设备的指针。                                          |
| `port`                    | `struct device_node *`                 | 用于 `drm_of_find_possible_crtcs()` 的设备树节点。           |
| `head`                    | `struct list_head`                     | 所有 CRTC 的链表头，链接到 `drm_mode_config.crtc_list`。     |
| `name`                    | `char *`                               | CRTC 的可读名称，可由驱动覆盖。                              |
| `mutex`                   | `struct drm_modeset_lock`              | 用于保护 CRTC 状态的锁，包括模式、DPMS 状态等。对于原子驱动程序，这个锁还保护 `state`。 |
| `base`                    | `struct drm_mode_object`               | 基础 KMS 对象，用于 ID 跟踪等功能。                          |
| `primary`                 | `struct drm_plane *`                   | CRTC 的主平面，仅用于传统 IOCTL（如 `SETCRTC` 和 `PAGE_FLIP`）。 |
| `cursor`                  | `struct drm_plane *`                   | CRTC 的光标平面，仅用于传统 IOCTL（如 `SETCURSOR` 和 `SETCURSOR2`）。 |
| `index`                   | `unsigned`                             | CRTC 在 `mode_config.list` 中的位置，可用作数组索引。        |
| `cursor_x`                | `int`                                  | 光标的当前 x 坐标（用于传统 IOCTL）。                        |
| `cursor_y`                | `int`                                  | 光标的当前 y 坐标（用于传统 IOCTL）。                        |
| `enabled`                 | `bool`                                 | 表示 CRTC 是否启用（仅供传统驱动使用，原子驱动应参考 `drm_crtc_state.enable` 和 `drm_crtc_state.active`）。 |
| `mode`                    | `struct drm_display_mode`              | 当前模式时间（仅供传统驱动使用，原子驱动应参考 `drm_crtc_state.mode`）。 |
| `hwmode`                  | `struct drm_display_mode`              | 硬件中已调整的模式，用于高精度 vblank 时间戳计算。           |
| `x`                       | `int`                                  | 屏幕上的 x 坐标（仅供传统驱动使用，原子驱动应参考主平面的 `drm_plane_state.crtc_x`）。 |
| `y`                       | `int`                                  | 屏幕上的 y 坐标（仅供传统驱动使用，原子驱动应参考主平面的 `drm_plane_state.crtc_y`）。 |
| `funcs`                   | `const struct drm_crtc_funcs *`        | CRTC 控制函数集。                                            |
| `gamma_size`              | `uint32_t`                             | 伽马校正表的大小，通过 `drm_mode_crtc_set_gamma_size()` 设置。 |
| `gamma_store`             | `uint16_t *`                           | 用于存储伽马校正值。                                         |
| `helper_private`          | `const struct drm_crtc_helper_funcs *` | 中间层的私有数据。                                           |
| `properties`              | `struct drm_object_properties`         | 跟踪 CRTC 的属性。                                           |
| `state`                   | `struct drm_crtc_state *`              | 当前原子状态，由 `mutex` 保护。                              |
| `commit_list`             | `struct list_head`                     | 跟踪挂起的提交。                                             |
| `commit_lock`             | `spinlock_t`                           | 用于保护 `commit_list` 的自旋锁。                            |
| `debugfs_entry`           | `struct dentry *`                      | CRTC 的 DebugFS 目录（仅在 `CONFIG_DEBUG_FS` 配置启用时可用）。 |
| `crc`                     | `struct drm_crtc_crc`                  | CRC 捕获的配置设置。                                         |
| `fence_context`           | `unsigned int`                         | 用于 Fence 操作的时间线上下文。                              |
| `fence_lock`              | `spinlock_t`                           | 保护 `fence_context` 中 Fence 的自旋锁。                     |
| `fence_seqno`             | `unsigned long`                        | Fence 时间线的单调计数器。                                   |
| `timeline_name`           | `char[32]`                             | Fence 时间线的名称。                                         |
| `vop_dump_status`         | `enum vop_dump_status`                 | VOP 状态转储控制（仅在 `CONFIG_ROCKCHIP_DRM_DEBUG` 配置启用时可用）。 |
| `vop_dump_list_head`      | `struct list_head`                     | VOP 转储列表头。                                             |
| `vop_dump_list_init_flag` | `bool`                                 | VOP 转储列表初始化标志。                                     |
| `vop_dump_times`          | `int`                                  | VOP 转储次数。                                               |
| `frame_count`             | `int`                                  | 转储缓冲区的帧计数。                                         |

后面这几个成员是因为rk改了crtc的源码。主线中并没有这几行代码。

```
#if defined(CONFIG_ROCKCHIP_DRM_DEBUG)
	/**
	 * @vop_dump_status the status of vop dump control
	 * @vop_dump_list_head the list head of vop dump list
	 * @vop_dump_list_init_flag init once
	 * @vop_dump_times control the dump times
	 * @frme_count the frame of dump buf
	 */
	enum vop_dump_status vop_dump_status;
	struct list_head vop_dump_list_head;
	bool vop_dump_list_init_flag;
	int vop_dump_times;
	int frame_count;
#endif
```

```
https://elixir.bootlin.com/linux/v6.12.6/source/include/drm/drm_crtc.h#L926
```

这个对象部分成员用于原子操作，这个在plane中也见到过，以后研究。

存储了部分属性，比如gamma_xxx。还有ops，一些结构体基本的锁，父设备等。

最重要的就是plane，一个primary一个cursor。但是仅用于ioctl操作？

## ops

![CleanShot 2025-01-04 at 21.43.43](./assets/pics/CleanShot%202025-01-04%20at%2021.43.43.png)

这堆东西看得我头疼，东西太多，正好ctrc于plane已经看了一遍，那么可以结合应用层深入了解一下，包含上述不曾介绍过的参数。