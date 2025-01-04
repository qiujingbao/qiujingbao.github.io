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

然后一个plane可以