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