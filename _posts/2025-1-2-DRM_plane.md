---
layout: post

title: DRM-plane
categories: [DRM]
tags: [DRM,SDK]
typora-root-url: ..
---

代码位置：

```
kernel/drivers/gpu/drm/drm_plane.c
kernel/drivers/gpu/drm/drm_plane_helper.c
```

```
Plane 的中文总结
在 DRM 中，Plane（平面） 是一种表示图像源的抽象概念，可以在扫描输出过程中与 CRTC（显示控制器） 进行混合或叠加。

Plane 的作用
图像源：

Plane 的输入数据来自一个 drm_framebuffer 对象。
可以指定图像的裁剪、缩放，以及在显示管线（由 drm_crtc 表示）中可见区域的位置。
属性设置：

Plane 还可以通过额外的属性来设置像素的定位和混合方式，例如：
旋转（Rotation）。
Z 轴位置（Z-position，用于叠加次序）。
状态存储：

所有这些属性都存储在 drm_plane_state 中。
Plane 的创建
分配与初始化：

驱动程序需要分配并清零一个 drm_plane 结构体实例（可以是更大结构的一部分）。
然后通过调用 drm_universal_plane_init() 注册该 Plane。
类型关联：

每个 CRTC 至少需要一个 Primary Plane，以避免对用户空间造成意外。
特殊类型的 Plane（如光标或叠加）可以通过 drm_crtc_init_with_planes() 关联到对应的 CRTC。
Plane 的类型
Plane 的类型通过不可变的 "type" 枚举属性暴露给用户空间，可能的值包括：

Primary（主平面）：
每个 CRTC 至少有一个，用于显示主要内容（例如桌面背景或主窗口）。
Overlay（叠加平面）：
用于叠加显示额外的图像内容（例如视频或弹窗）。
Cursor（光标平面）：
专门用于显示鼠标光标，通常支持硬件加速。
```

相关概念文章：

https://blog.csdn.net/hexiaolong2009/article/details/84934294

### obj









```
int drm_plane_create_alpha_property(struct drm_plane *plane)
{
	struct drm_property *prop;

	prop = drm_property_create_range(plane->dev, 0, "alpha",
					 0, DRM_BLEND_ALPHA_OPAQUE);
	if (!prop)
		return -ENOMEM;

	drm_object_attach_property(&plane->base, prop, DRM_BLEND_ALPHA_OPAQUE);
	plane->alpha_property = prop;

	if (plane->state)
		plane->state->alpha = DRM_BLEND_ALPHA_OPAQUE;

	return 0;
}
EXPORT_SYMBOL(drm_plane_create_alpha_property);
```

可以看到，这个属性是直接附加到了该plane的base_obj上。

而且在init的时候。

```
int drm_universal_plane_init(struct drm_device *dev, struct drm_plane *plane,
			     uint32_t possible_crtcs,
			     const struct drm_plane_funcs *funcs,
			     const uint32_t *formats, unsigned int format_count,
			     const uint64_t *format_modifiers,
			     enum drm_plane_type type,
			     const char *name, ...)
{
...
	ret = drm_mode_object_add(dev, &plane->base, DRM_MODE_OBJECT_PLANE);
	if (ret)
		return ret;

	drm_modeset_lock_init(&plane->mutex);

	plane->base.properties = &plane->properties;
	plane->dev = dev;
```

可以看到base的properties指向的就是plane。至于propeties的关系在obj部分介绍过。

| **字段名**                | **类型**                                | **描述**                                                     | **备注**                                                     |
| ------------------------- | --------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `dev`                     | `struct drm_device *`                   | 所属的 DRM 设备。                                            | 关联的设备对象。                                             |
| `head`                    | `struct list_head`                      | 当前设备中所有平面的列表，链接自 `drm_mode_config.plane_list`。 | 生命周期与设备一致，无需锁保护。                             |
| `name`                    | `char *`                                | 平面的可读名称，可由驱动覆盖。                               | 便于识别的名称。                                             |
| `mutex`                   | `struct drm_modeset_lock`               | 保护模式设置平面状态，与绑定的 CRTC 的 `mutex` 一起保护。    | 对于原子驱动，主要保护 `state` 字段。                        |
| `base`                    | `struct drm_mode_object`                | 基础模式对象。                                               | 用于统一管理 DRM 对象。                                      |
| `possible_crtcs`          | `uint32_t`                              | 此平面可以绑定的 CRTC，由 `drm_crtc_mask()` 构造。           | 用于限制平面与 CRTC 的绑定关系。                             |
| `format_types`            | `uint32_t *`                            | 平面支持的格式数组。                                         | 包含支持的像素格式列表。kernel/include/uapi/drm/drm_fourcc.h |
| `format_count`            | `unsigned int`                          | `format_types` 数组的大小。                                  | 统计支持的格式数量。                                         |
| `format_default`          | `bool`                                  | 是否使用默认格式。                                           | 仅供 `drm_plane_init` 兼容性包装器使用。                     |
| `modifiers`               | `uint64_t *`                            | 平面支持的修饰符数组。                                       | 包含格式修饰符列表。                                         |
| `modifier_count`          | `unsigned int`                          | `modifiers` 数组的大小。                                     | 统计支持的修饰符数量。                                       |
| `crtc`                    | `struct drm_crtc *`                     | 当前绑定的 CRTC，仅对非原子驱动有意义。                      | 对于原子驱动，强制为 NULL，需通过 `drm_plane_state.crtc` 检查。 |
| `fb`                      | `struct drm_framebuffer *`              | 当前绑定的帧缓冲区，仅对非原子驱动有意义。                   | 对于原子驱动，强制为 NULL，需通过 `drm_plane_state.fb` 检查。 |
| `old_fb`                  | `struct drm_framebuffer *`              | 模式设置期间临时跟踪旧的帧缓冲区，仅对非原子驱动使用。       | 对于原子驱动，强制为 NULL。                                  |
| `funcs`                   | `const struct drm_plane_funcs *`        | 平面控制函数。                                               | 包含平面相关的操作函数。                                     |
| `properties`              | `struct drm_object_properties`          | 平面的属性跟踪。                                             | 用于管理平面属性。                                           |
| `type`                    | `enum drm_plane_type`                   | 平面的类型。                                                 | 参见 `enum drm_plane_type` 以获取详细信息。                  |
| `index`                   | `unsigned`                              | 在 `mode_config.list` 中的位置，可用作数组索引。             | 在平面生命周期内保持不变。                                   |
| `helper_private`          | `const struct drm_plane_helper_funcs *` | 中间层的私有数据。                                           | 包含辅助操作函数。                                           |
| `state`                   | `struct drm_plane_state *`              | 平面的当前原子状态。                                         | 受 `mutex` 保护，非阻塞原子提交可通过特定的遍历宏访问。      |
| `alpha_property`          | `struct drm_property *`                 | 平面的可选透明度属性。                                       | 可通过 `drm_plane_create_alpha_property()` 创建。            |
| `zpos_property`           | `struct drm_property *`                 | 平面的可选 Z 位置属性。                                      | 可通过 `drm_plane_create_zpos_property()` 创建。             |
| `rotation_property`       | `struct drm_property *`                 | 平面的可选旋转属性。                                         | 可通过 `drm_plane_create_rotation_property()` 创建。         |
| `blend_mode_property`     | `struct drm_property *`                 | 平面的可选像素混合模式属性。                                 | 描述当前平面像素与背景的合成方式。                           |
| `color_encoding_property` | `struct drm_property *`                 | 平面的可选颜色编码属性。                                     | 用于非 RGB 格式，参见 `drm_plane_create_color_properties()`。 |
| `color_range_property`    | `struct drm_property *`                 | 平面的可选颜色范围属性。                                     | 用于非 RGB 格式，参见 `drm_plane_create_color_properties()`。 |

### enmu

| **枚举值**               | **描述**                                                     | **备注**                                                     |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `DRM_PLANE_TYPE_OVERLAY` | 表示所有非主平面和非光标平面的覆盖平面。一些驱动程序内部将这些类型的平面称为“sprites”。 | 默认情况下，这些平面仅对旧版用户空间可见，除非用户空间启用了 `DRM_CLIENT_CAP_UNIVERSAL_PLANES` 客户端功能。 |
| `DRM_PLANE_TYPE_PRIMARY` | 表示一个 CRTC 的“主”平面。主平面是由 CRTC 模式设置和翻转操作（如 `drm_crtc_funcs.page_flip` 和 `drm_crtc_funcs.set_config` 钩子）操作的平面。 | 主平面通常用于显示主要内容。                                 |
| `DRM_PLANE_TYPE_CURSOR`  | 表示一个 CRTC 的“光标”平面。光标平面是由 `DRM_IOCTL_MODE_CURSOR` 和 `DRM_IOCTL_MODE_CURSOR2` IOCTL 操作的平面。 | 光标平面通常用于显示鼠标指针或其他小型图形。                 |

##### **补充说明**

1. **历史原因**：不同类型的平面有不同的用户空间 API 语义。虽然对于支持通用平面的用户空间，所有平面类型的功能都是一致的，但硬件和驱动可能限制其支持能力。
2. **兼容性**：为兼容旧版用户空间，默认情况下只会向用户空间提供覆盖平面（`DRM_PLANE_TYPE_OVERLAY`）。启用 `DRM_CLIENT_CAP_UNIVERSAL_PLANES` 后，可以获取包含所有平面类型的通用平面列表。
3. **警告**：此枚举的值属于用户空间 ABI (UABI)，因为它们会暴露在“type”属性中，因此需要保证其值的稳定性。

来回顾一点历史：内核向用户态导出的接口实际上不包含`Primary Plane`，对应plane的接口只能操作`Cursor Plane`和`Overlay Plane`，后期提供了一个`Universial Plane`特性，使得用户态API可以直接操作`Primary Plane`。在明白这个历史遗留问题后，对`drm_plane`的实现就好理解了。

```

在 Linux 内核的 DRM（Direct Rendering Manager）子系统中，Universal Plane（通用平面） 是一种抽象的显示平面管理模型，旨在简化用户空间对显示硬件的访问，同时为现代显示硬件提供更强大的支持。

什么是 Universal Plane？
Universal Plane 是对显示硬件中所有平面的统一抽象。显示硬件中的平面通常分为以下三种类型：

Primary Plane（主平面）

主平面用于显示主要内容（如桌面或全屏应用）。
它与 CRTC（显示控制器）直接关联，通常是屏幕的默认显示内容。
Cursor Plane（光标平面）

专用于显示鼠标指针或其他小型、可移动的图形。
支持硬件加速的光标操作。
Overlay Plane（覆盖平面）

用于显示附加内容（如视频播放窗口）。
允许硬件进行叠加和混合操作，支持硬件加速的多平面显示。
在传统的 DRM 模型中，这些平面是分别管理的，用户空间必须通过不同的接口访问和操作它们。

Universal Plane 的引入
Universal Plane 的概念统一了上述三种平面类型，允许用户空间通过一个通用接口访问所有类型的平面。它的主要特点包括：

统一的接口

用户空间可以通过 DRM_IOCTL_MODE_GETPLANERESOURCES 获取所有平面的列表，无需区分类型。
平面类型通过 type 属性区分（如 Primary、Cursor、Overlay）。
增强的灵活性

用户空间可以根据硬件能力和需求动态选择合适的平面类型。
支持更复杂的显示场景（如多窗口叠加、透明效果等）。
兼容性

为了兼容旧版用户空间，默认情况下，用户空间只能看到 Overlay 平面。
如果用户空间启用了 DRM_CLIENT_CAP_UNIVERSAL_PLANES 功能，则可以访问所有类型的平面。
实现细节
Universal Plane 是通过 DRM 的 Atomic Mode Setting 机制实现的，它将平面、CRTC 和连接器的状态整合到一个统一的模型中。
在驱动层面，每个平面对应一个 struct drm_plane 对象，包含平面类型、支持的格式、可能的绑定 CRTC 等信息。
平面的类型通过 enum drm_plane_type 表示，包括：
DRM_PLANE_TYPE_PRIMARY
DRM_PLANE_TYPE_CURSOR
DRM_PLANE_TYPE_OVERLAY
优点
简化用户空间开发
用户空间不需要分别处理不同类型的平面，减少了接口复杂性。

支持现代硬件
现代显示硬件通常具有多个 Overlay 平面，Universal Plane 模型可以充分利用这些能力。

更强的灵活性
用户空间可以根据需求自由配置显示硬件，实现多窗口、多内容的高效显示。

使用场景
视频播放器：使用 Overlay 平面加速视频显示。
图形桌面环境：通过 Primary 和 Cursor 平面实现高效的桌面显示和鼠标指针渲染。
游戏应用：利用多个平面叠加实现复杂的图形效果。
总结
Universal Plane 是 DRM 子系统的重要特性，统一了平面的管理和操作接口，提升了用户空间开发的灵活性和硬件利用率。它是现代显示硬件和复杂显示场景支持的基础。
```

具体实现往后再看。

### ops

下面是对应ops的介绍，chatgpt总结。

| **回调函数**                 | **功能描述**                                                 | **适用场景**                                                 | **返回值**                              |
| ---------------------------- | ------------------------------------------------------------ | :----------------------------------------------------------- | --------------------------------------- |
| **`update_plane`**           | 配置并启用 Plane，与指定的 CRTC 和 Framebuffer 绑定，设置源矩形和目标矩形。 | 传统模式设置下启用 Plane；原子驱动推荐使用 `drm_atomic_helper_update_plane()` 实现此功能。 | 成功返回 0，失败返回负错误码。          |
| **`disable_plane`**          | 禁用 Plane，确保其不再被 CRTC 处理。                         | 传统模式设置下禁用 Plane；原子驱动推荐使用 `drm_atomic_helper_disable_plane()` 实现此功能。 | 成功返回 0，失败返回负错误码。          |
| **`destroy`**                | 清理 Plane 资源。                                            | 在驱动卸载时调用，用于释放 Plane 相关资源。                  | 无返回值。                              |
| **`reset`**                  | 将 Plane 的硬件和软件状态重置为关闭状态。                    | 通过 `drm_mode_config_reset()` 调用；原子驱动可使用 `drm_atomic_helper_plane_reset()` 实现。 | 无返回值。                              |
| **`set_property`**           | 更新 Plane 上的属性（仅适用于传统驱动）。                    | 如果驱动不支持任何传统属性，可以省略此回调；原子驱动不使用此回调。 | 成功返回 0，失败返回负错误码。          |
| **`atomic_duplicate_state`** | 复制 Plane 的当前原子状态。                                  | 原子驱动中用于状态管理；必须实现此回调。                     | 返回复制的状态指针，分配失败返回 NULL。 |
| **`atomic_destroy_state`**   | 销毁通过 `atomic_duplicate_state` 复制的状态，释放相关资源。 | 原子驱动中用于状态管理；必须实现此回调。                     | 无返回值。                              |
| **`atomic_set_property`**    | 解码驱动私有属性值，并将其存储到状态结构中。                 | 原子驱动中处理驱动私有属性；推荐将硬件/厂商特定状态封装为私有属性。 | 成功返回 0，属性未实现返回 -EINVAL。    |
| **`atomic_get_property`**    | 读取解码后的驱动私有属性值，用于实现 `GETPLANE` IOCTL。      | 原子驱动中处理驱动私有属性。                                 | 成功返回 0，属性未实现返回 -EINVAL。    |
| **`late_register`**          | 注册附加的用户空间接口（如 debugfs 接口）。                  | 在驱动加载的后期调用；与 `early_unregister` 配对使用。       | 成功返回 0，失败返回负错误码。          |
| **`early_unregister`**       | 注销通过 `late_register` 注册的用户空间接口。                | 在驱动卸载的早期调用；与 `late_register` 配对使用。          | 无返回值。                              |
| **`atomic_print_state`**     | 打印扩展的驱动私有状态信息。                                 | 如果驱动子类化了 `struct drm_plane_state`，可以实现此回调以输出额外的状态信息。 | 无返回值。                              |
| **`format_mod_supported`**   | 检查 Plane 是否支持指定的格式和修饰符组合。                  | DRM 核心用于生成正确的格式位掩码，或在原子检查阶段验证修饰符。 | 支持返回 `true`，不支持返回 `false`。   |

**传统驱动**：使用 `update_plane`、`disable_plane` 和 `set_property` 等回调处理模式设置和属性更新。

**原子驱动**：推荐实现 `atomic_duplicate_state`、`atomic_destroy_state` 和其他与原子状态相关的回调，以支持现代模式设置和属性管理。

**扩展性**：`late_register` 和 `early_unregister` 用于扩展用户空间接口，`atomic_print_state` 用于调试扩展状态。

### plane核心

![CleanShot 2025-01-04 at 13.58.16](./assets/pics/CleanShot%202025-01-04%20at%2013.58.16.png)

```
/**
 * drm_plane_init - Initialize a legacy plane
 * @dev: DRM device
 * @plane: plane object to init
 * @possible_crtcs: bitmask of possible CRTCs
 * @funcs: callbacks for the new plane
 * @formats: array of supported formats (DRM_FORMAT\_\*)
 * @format_count: number of elements in @formats
 * @is_primary: plane type (primary vs overlay)
 *
 * Legacy API to initialize a DRM plane.
 *
 * New drivers should call drm_universal_plane_init() instead.
 *
 * Returns:
 * Zero on success, error code on failure.
 */
int drm_plane_init(struct drm_device *dev, struct drm_plane *plane,
		   uint32_t possible_crtcs,
		   const struct drm_plane_funcs *funcs,
		   const uint32_t *formats, unsigned int format_count,
		   bool is_primary)
{
	enum drm_plane_type type;

	type = is_primary ? DRM_PLANE_TYPE_PRIMARY : DRM_PLANE_TYPE_OVERLAY;
	return drm_universal_plane_init(dev, plane, possible_crtcs, funcs,
					formats, format_count,
					NULL, type, NULL);
}
EXPORT_SYMBOL(drm_plane_init);
```

```
/**
 * drm_universal_plane_init - Initialize a new universal plane object
 * @dev: DRM device
 * @plane: plane object to init
 * @possible_crtcs: bitmask of possible CRTCs
 * @funcs: callbacks for the new plane 盲猜是helper的函数，或者自己写或者helper套一层
 * @formats: array of supported formats (DRM_FORMAT\_\*)
 * @format_count: number of elements in @formats
 * @format_modifiers: array of struct drm_format modifiers terminated by
 *                    DRM_FORMAT_MOD_INVALID
 * @type: type of plane (overlay, primary, cursor)
 * @name: printf style format string for the plane name, or NULL for default name
 *
 * Initializes a plane object of type @type.
 *
 * Returns:
 * Zero on success, error code on failure.
 */
int drm_universal_plane_init(struct drm_device *dev, struct drm_plane *plane,
			     uint32_t possible_crtcs,
			     const struct drm_plane_funcs *funcs,
			     const uint32_t *formats, unsigned int format_count,
			     const uint64_t *format_modifiers,
			     enum drm_plane_type type,
			     const char *name, ...)
{
	struct drm_mode_config *config = &dev->mode_config;
	unsigned int format_modifier_count = 0;
	int ret;


	/* 初始化obj为DRM_MODE_OBJECT_PLANE类型 */
	ret = drm_mode_object_add(dev, &plane->base, DRM_MODE_OBJECT_PLANE);
	if (ret)
		return ret;

	drm_modeset_lock_init(&plane->mutex);
	/* 可以看到base的properties与plane的properties一样 */
	plane->base.properties = &plane->properties;
	plane->dev = dev;
	plane->funcs = funcs;
	/* 分配存储类型用的空间 */
	plane->format_types = kmalloc_array(format_count, sizeof(uint32_t),
					    GFP_KERNEL);
	if (!plane->format_types) {
		DRM_DEBUG_KMS("out of memory when allocating plane\n");
		drm_mode_object_unregister(dev, &plane->base);
		return -ENOMEM;
	}

...
	/* 在这里拷贝进去 */
	memcpy(plane->format_types, formats, format_count * sizeof(uint32_t));
	plane->format_count = format_count;
	/* 还是不太理解这个修饰符啥意思 */
	memcpy(plane->modifiers, format_modifiers,
	       format_modifier_count * sizeof(format_modifiers[0]));
	plane->possible_crtcs = possible_crtcs;
	plane->type = type;

	list_add_tail(&plane->head, &config->plane_list);
	plane->index = config->num_total_plane++;

	drm_object_attach_property(&plane->base,
				   config->plane_type_property,
				   plane->type);
	/* check feature */
	if (drm_core_check_feature(dev, DRIVER_ATOMIC)) {
		drm_object_attach_property(&plane->base, config->prop_fb_id, 0);
		drm_object_attach_property(&plane->base, config->prop_in_fence_fd, -1);
		drm_object_attach_property(&plane->base, config->prop_crtc_id, 0);
		drm_object_attach_property(&plane->base, config->prop_crtc_x, 0);
		drm_object_attach_property(&plane->base, config->prop_crtc_y, 0);
		drm_object_attach_property(&plane->base, config->prop_crtc_w, 0);
		drm_object_attach_property(&plane->base, config->prop_crtc_h, 0);
		drm_object_attach_property(&plane->base, config->prop_src_x, 0);
		drm_object_attach_property(&plane->base, config->prop_src_y, 0);
		drm_object_attach_property(&plane->base, config->prop_src_w, 0);
		drm_object_attach_property(&plane->base, config->prop_src_h, 0);
	}
/* 应该也是modifiers的代码 */
	if (config->allow_fb_modifiers)
		create_in_format_blob(dev, plane);

	return 0;
}
EXPORT_SYMBOL(drm_universal_plane_init);

```

在mtk的代码中

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
```

```
static const struct drm_plane_funcs mtk_plane_funcs = {
	.update_plane = drm_atomic_helper_update_plane,
	.disable_plane = drm_atomic_helper_disable_plane,
	.destroy = drm_plane_cleanup,
	.reset = mtk_plane_reset,
	.atomic_duplicate_state = mtk_plane_duplicate_state,
	.atomic_destroy_state = mtk_drm_plane_destroy_state,
};
```

可以看到一部分是helper的函数

这里提供的是一些承上启下的函数，真正也逻辑代码在helper中，也就是func的函数。

### func的包装函数

```
int drm_plane_register_all(struct drm_device *dev)
{
	struct drm_plane *plane;
	int ret = 0;

	drm_for_each_plane(plane, dev) {
		if (plane->funcs->late_register)
			ret = plane->funcs->late_register(plane);
		if (ret)
			return ret;
	}

	return 0;
}

void drm_plane_unregister_all(struct drm_device *dev)
{
	struct drm_plane *plane;

	drm_for_each_plane(plane, dev) {
		if (plane->funcs->early_unregister)
			plane->funcs->early_unregister(plane);
	}
}
```

```
/**
 * drm_mode_plane_set_obj_prop - set the value of a property
 * @plane: drm plane object to set property value for
 * @property: property to set
 * @value: value the property should be set to
 *
此函数用于在给定的plane对象上设置给定的属性。此函数调用驱动程序的->set_property回调，如果回调成功，则更改属性的软件状态。
 * Returns:
 * Zero on success, error code on failure.
 */
int drm_mode_plane_set_obj_prop(struct drm_plane *plane,
				struct drm_property *property,
				uint64_t value)
{
	int ret = -EINVAL;
	struct drm_mode_object *obj = &plane->base;

	if (plane->funcs->set_property)
		ret = plane->funcs->set_property(plane, property, value);
	if (!ret)
		drm_object_property_set_value(obj, property, value);

	return ret;
}
EXPORT_SYMBOL(drm_mode_plane_set_obj_prop);
```

### helper函数

上述ops函数的简单实现。





上述只是简单的介绍了各个函数的功能，但是plane于crtc如何联系？整个图像的显示流程是什么？光标如何移动，画面如何更新都没有涉及。

