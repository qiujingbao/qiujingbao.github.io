---
layout: post

title: DRM-overview-obj
categories: [DRM]
tags: [DRM,SDK]
typora-root-url: ..
---

这部分是针对drm整体代码的介绍。

## helper架构

helper架构是我起的名，知道是指什么东西就好。DRM子系统的API比较难抽象，简单来说就是硬件各有各的不同，很多情况下，驱动可以使用一个共同的实现，而在其它情况下，驱动需要提供自己的实现。因此，DRM驱动核心的接口使用了helper架构，其基本思想是通过一组回调函数抽象特定组件的操作，比如`drm_connector_funcs`，同时又使用另外一组helper函数给出了原先那组回调函数的通用实现，让开发最者实现这组helper函数抽象出的回调函数即可。

这样双层的实现即能保证开发者有足够高的自由度（完全不用helper函数），也能简化开发者的开发（使用helper函数），同时提供给开发者hook特定helper函数的能力。下面以`drm_connector`为例说明helper架构的实现与使用方式。

正常情况下，创建`drm_connector`对象时需要提供`struct drm_connector_funcs`回调函数组，而使用helper函数时，可以直接用helper函数填充对应回调函数：

```
static const struct drm_connector_funcs vc4_hdmi_connector_funcs = {
        .detect = vc4_hdmi_connector_detect,
        .fill_modes = drm_helper_probe_single_connector_modes,
        .destroy = vc4_hdmi_connector_destroy,
        .reset = drm_atomic_helper_connector_reset,
        .atomic_duplicate_state = drm_atomic_helper_connector_duplicate_state,
        .atomic_destroy_state = drm_atomic_helper_connector_destroy_state,
};
```

事实上helper函数并不万能，只是抽象出了大多数驱动程序应该共享的行为，而特定于硬件的部分，则需要以回调函数的形式提供给helper函数，这个回调函数组由`struct drm_connector_helper_funcs`提供。在创建`drm_connector`时，需要通过`drm_connector_helper_add`函数注册。函数将对应的回调函数对象的地址保存在了`drm_connector`中的`helper_private`指针中，如下：

```
static inline void drm_connector_helper_add(struct drm_connector *connector,
                                            const struct drm_connector_helper_funcs *funcs)
{
        connector->helper_private = funcs;
}
```

这一套实现位于`include/drm/drm_modeset_helper_vtables.h`中，其他的DRM对象都有类似的实现，可以详细阅读`drm_connector_helper_funcs`的注释，理解其中对应的回调函数的用途。在实现DRM驱动时，helper架构会频繁用到，合理掌握helper函数可以极大简化开发，提升驱动程序的兼容性。

via：https://crab2313.github.io/post/drm-legacy-kms/

通过阅读gem的代码，在举几个例子。在rk中实现一个驱动需要实现如下回调。

```
static struct drm_driver rockchip_drm_driver = {
	.driver_features	= DRIVER_MODESET | DRIVER_GEM |
				  DRIVER_PRIME | DRIVER_ATOMIC |
				  DRIVER_RENDER,
	.postclose		= rockchip_drm_postclose,
	.lastclose		= rockchip_drm_lastclose,
	.open			= rockchip_drm_open,
	.gem_vm_ops		= &drm_gem_cma_vm_ops,
	.gem_free_object_unlocked = rockchip_gem_free_object,
	.dumb_create		= rockchip_gem_dumb_create,
	.dumb_map_offset	= rockchip_gem_dumb_map_offset,
	.dumb_destroy		= drm_gem_dumb_destroy,
...
};
```

最基本的dumb buffer的申请释放与映射。

```
/*
 * rockchip_gem_dumb_create - (struct drm_driver)->dumb_create callback
 * function
 *
 * This aligns the pitch and size arguments to the minimum required. wrap
 * this into your own function if you need bigger alignment.
 这将间距和大小参数与所需的最小值对齐。如果您需要更大的对齐，请将其打包到您自己的函数中。
 */
int rockchip_gem_dumb_create(struct drm_file *file_priv,
			     struct drm_device *dev,
			     struct drm_mode_create_dumb *args)
{
	struct rockchip_gem_object *rk_obj;
	int min_pitch = DIV_ROUND_UP(args->width * args->bpp, 8);

	/*
	 * align to 64 bytes since Mali requires it.
	 */
	args->pitch = ALIGN(min_pitch, 64);
	args->size = args->pitch * args->height;

	rk_obj = rockchip_gem_create_with_handle(file_priv, dev, args->size,
						 &args->handle, args->flags);

	return PTR_ERR_OR_ZERO(rk_obj);
}
```

```
/*
 * rockchip_gem_create_with_handle - allocate an object with the given
 * size and create a gem handle on it
 *
 * returns a struct rockchip_gem_object* on success or ERR_PTR values
 * on failure.
 */
static struct rockchip_gem_object *
rockchip_gem_create_with_handle(struct drm_file *file_priv,
				struct drm_device *drm, unsigned int size,
				unsigned int *handle, unsigned int flags)
{
	struct rockchip_gem_object *rk_obj;
	struct drm_gem_object *obj;
	int ret;
	bool alloc_kmap = flags & ROCKCHIP_BO_ALLOC_KMAP ? true : false;

	rk_obj = rockchip_gem_create_object(drm, size, alloc_kmap, flags);
	if (IS_ERR(rk_obj))
		return ERR_CAST(rk_obj);

	obj = &rk_obj->base;

	/*
	 * allocate a id of idr table where the obj is registered
	 * and handle has the id what user can see.
	 */
	ret = drm_gem_handle_create(file_priv, obj, handle);
	if (ret)
		goto err_handle_create;

	/* drop reference from allocate - handle holds it now. */
	drm_gem_object_put_unlocked(obj);

	return rk_obj;

err_handle_create:
	rockchip_gem_free_object(obj);

	return ERR_PTR(ret);
}
```

```
struct rockchip_gem_object *
rockchip_gem_create_object(struct drm_device *drm, unsigned int size,
			   bool alloc_kmap, unsigned int flags)
{
	struct rockchip_gem_object *rk_obj;
	int ret;

	rk_obj = rockchip_gem_alloc_object(drm, size);
	if (IS_ERR(rk_obj))
		return rk_obj;
	rk_obj->flags = flags;

	ret = rockchip_gem_alloc_buf(rk_obj, alloc_kmap);
	if (ret)
		goto err_free_rk_obj;

	return rk_obj;

err_free_rk_obj:
	rockchip_gem_release_object(rk_obj);
	return ERR_PTR(ret);
}
```

```
static struct rockchip_gem_object *
	rockchip_gem_alloc_object(struct drm_device *drm, unsigned int size)
{
	struct address_space *mapping;
	struct rockchip_gem_object *rk_obj;
	struct drm_gem_object *obj;

#ifdef CONFIG_ARM_LPAE
	gfp_t gfp_mask = GFP_HIGHUSER | __GFP_RECLAIMABLE | __GFP_DMA32;
#else
	gfp_t gfp_mask = GFP_HIGHUSER | __GFP_RECLAIMABLE;
#endif
	size = round_up(size, PAGE_SIZE);

	rk_obj = kzalloc(sizeof(*rk_obj), GFP_KERNEL);
	if (!rk_obj)
		return ERR_PTR(-ENOMEM);

	obj = &rk_obj->base;

	drm_gem_object_init(drm, obj, size);

	mapping = file_inode(obj->filp)->i_mapping;
	mapping_set_gfp_mask(mapping, gfp_mask);

	return rk_obj;
}
```

这些代码一路看下去，最终到`drm_gem_object_xxx`.这个create可能有些绕。来看mmap。

```
int rockchip_gem_dumb_map_offset(struct drm_file *file_priv,
				 struct drm_device *dev, uint32_t handle,
				 uint64_t *offset)
{
	struct drm_gem_object *obj;
	int ret = 0;

	obj = drm_gem_object_lookup(file_priv, handle);
	if (!obj) {
		DRM_ERROR("failed to lookup gem object.\n");
		return -EINVAL;
	}

	ret = drm_gem_create_mmap_offset(obj);
	if (ret)
		goto out;

	*offset = drm_vma_node_offset_addr(&obj->vma_node);
	DRM_DEBUG_KMS("offset = 0x%llx\n", *offset);

out:
	drm_gem_object_unreference_unlocked(obj);

	return ret;
}
```

可能有人说，这里面也没用到cma_helper呀，为什么说是helper架构，因为我们知道gem只是管理这个存储空间，至于你的存储空间是分配在哪内存还是dma都可以，用不用cma都可以，只是你想使用cma只需要加上cma_helper再加上自己想实现的功能，然后回调填上就可以，看mtk的dumb使用了dma完成的。

所以，结合上述代码会有更深的理解，对下面的话。

```
这样双层的实现即能保证开发者有足够高的自由度（完全不用helper函数），也能简化开发者的开发（使用helper函数），同时提供给开发者hook特定helper函数的能力（实现自己的需求）。
```

##  驱动入口

我们知道`drm_device`用于抽象一个完整的DRM设备，而其中与`Mode Setting`相关的部分则由`drm_mode_config`进行管理。为了让一个`drm_device`支持`KMS`相关的API，DRM框架要求驱动：

- 注册`drm_driver`时，`driver_features`标志位中需要存在`DRIVER_MODESET`
- 在probe函数中调用`drm_mode_config_init`函数初始化KMS框架，本质上是初始化`drm_device`中的`mode_config`结构体
- 填充mode_config中int min_width, min_height; int max_width, max_height的值，这些值是framebuffer的大小限制
- 设置mode_config->funcs指针，本质上是一组由驱动实现的回调函数，涵盖`KMS`中一些相当基本的操作
- 最后初始化`drm_device`中包含的`drm_connector`，`drm_crtc`等对象

我们知道注册一个支持`KMS`的DRM设备时，会在`/dev/drm/`下创建一个`card%d`文件，用户态可以通过打开该文件，并对文件描述符做相应的操作实现相应的功能。该文件描述符对应的文件操作回调函数（`filesystem_operations`）位于`drm_driver`中，并由驱动程序填充。典型如下：

```
static const struct file_operations vkms_driver_fops = {
        .owner          = THIS_MODULE,
        .open           = drm_open,
        .mmap           = drm_gem_mmap,
        .unlocked_ioctl = drm_ioctl,
        .compat_ioctl   = drm_compat_ioctl,
        .poll           = drm_poll,
        .read           = drm_read,
        .llseek         = no_llseek,
        .release        = drm_release,
};
```

基本都为DRM框架预先提供好的helper函数，可以根据驱动需要灵活改变。
