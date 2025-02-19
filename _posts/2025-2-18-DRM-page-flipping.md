---
layout: post

title: DRM-page-flipping
categories: [DRM]
tags: [DRM,page-flipping]
typora-root-url: ..
---

# page-flipping

这个东西看着很熟悉，在vblank中。

```
enum drm_vblank_seq_type {
	_DRM_VBLANK_ABSOLUTE = 0x0,	/**< Wait for specific vblank sequence number */
	_DRM_VBLANK_RELATIVE = 0x1,	/**< Wait for given number of vblanks */
	/* bits 1-6 are reserved for high crtcs */
	_DRM_VBLANK_HIGH_CRTC_MASK = 0x0000003e,
	_DRM_VBLANK_EVENT = 0x4000000,   /**< Send event instead of blocking */
	_DRM_VBLANK_FLIP = 0x8000000,   /**< Scheduled buffer swap should flip */
	_DRM_VBLANK_NEXTONMISS = 0x10000000,	/**< If missed, wait for next vblank */
	_DRM_VBLANK_SECONDARY = 0x20000000,	/**< Secondary display controller */
	_DRM_VBLANK_SIGNAL = 0x40000000	/**< Send signal instead of blocking, unsupported */
};
```

如_DRM_VBLANK_FLIP所示，但是这个到底是啥？

Page Flipping 是一种高效的帧缓冲区切换机制，主要用于 **减少屏幕撕裂**。它的核心思想是：

- **双缓冲**（Double Buffering）：有 **前缓冲区（Front Buffer）** 和 **后缓冲区（Back Buffer）**，前缓冲用于显示，后缓冲用于绘制。
- **翻转（Flip）**：当绘制完成后，交换前后缓冲区，让新帧显示出来。

**关键问题**： 如果 **随时** 切换前后缓冲区，可能会在屏幕刷新到一半时切换，导致屏幕上 **一部分显示新帧，一部分仍然是旧帧**，这就是 **屏幕撕裂（Tearing）** 现象。

```
Page flipping is a synchronization mechanism that replaces the frame buffer being scanned out by the CRTC with a new frame buffer during vertical blanking, avoiding tearing (except when requested otherwise through the DRM_MODE_PAGE_FLIP_ASYNC flag). When an application requests a page flip the DRM core verifies that the new frame buffer is large enough to be scanned out by the CRTC in the currently configured mode and then calls this hook with a pointer to the new frame buffer.
```

翻页是一种同步机制，在vblank用新的帧缓冲区替换CRTC扫描出的帧缓冲，避免撕裂（除非通过DRM_MODE_Page_FLIP_ASYNC标志另有要求）。当应用程序请求翻页时，DRM核心会验证新的帧缓冲区是否足够大，以便CRTC在当前配置的模式下扫描出来，然后用指向新帧缓冲区的指针调用此函数。

上述也说了是在vblank期间。那么是如何跳到了vblank中呢？首先是ioctl。

```
	DRM_IOCTL_DEF(DRM_IOCTL_MODE_PAGE_FLIP, drm_mode_page_flip_ioctl, DRM_MASTER|DRM_UNLOCKED),
```

```

int drm_mode_page_flip_ioctl(struct drm_device *dev,
			     void *data, struct drm_file *file_priv)
{
	struct drm_mode_crtc_page_flip_target *page_flip = data;
	struct drm_crtc *crtc;
	struct drm_plane *plane;
	struct drm_framebuffer *fb = NULL, *old_fb;
	struct drm_pending_vblank_event *e = NULL;
	u32 target_vblank = page_flip->sequence;
	struct drm_modeset_acquire_ctx ctx;
	int ret = -EINVAL;

...
一些检查
...

	plane = crtc->primary;

...

	if (crtc->funcs->page_flip_target) {
		u32 current_vblank;
		int r;

		r = drm_crtc_vblank_get(crtc);
		if (r)
			return r;

		current_vblank = (u32)drm_crtc_vblank_count(crtc);

/*
#define DRM_MODE_PAGE_FLIP_EVENT 0x01
#define DRM_MODE_PAGE_FLIP_ASYNC 0x02
#define DRM_MODE_PAGE_FLIP_TARGET_ABSOLUTE 0x4
#define DRM_MODE_PAGE_FLIP_TARGET_RELATIVE 0x8
#define DRM_MODE_PAGE_FLIP_TARGET (DRM_MODE_PAGE_FLIP_TARGET_ABSOLUTE | \
				   DRM_MODE_PAGE_FLIP_TARGET_RELATIVE)
#define DRM_MODE_PAGE_FLIP_FLAGS (DRM_MODE_PAGE_FLIP_EVENT | \
				  DRM_MODE_PAGE_FLIP_ASYNC | \
				  DRM_MODE_PAGE_FLIP_TARGET)
struct drm_mode_crtc_page_flip_target {
	__u32 crtc_id;
	__u32 fb_id;
	__u32 flags;
	__u32 sequence;
	__u64 user_data;
};				  
可以看到这个结构体是与userspace关联的中间结构体
*/
/* 对比期待的vblank的seq与现在seq的关系 */
/* DRM_MODE_PAGE_FLIP_TARGET这个flag代表有target seq */
		switch (page_flip->flags & DRM_MODE_PAGE_FLIP_TARGET) {
		绝对帧目标 (Absolute Target VBlank)，指定 Page Flip 必须发生在 target_vblank 这一帧。
		约束条件：
		target_vblank 不能大于 current_vblank + 1，即最多提前 1 帧 进行 Flip。否则，返回 -EINVAL（无效参数）。
		case DRM_MODE_PAGE_FLIP_TARGET_ABSOLUTE:
			if ((int)(target_vblank - current_vblank) > 1) {
				DRM_DEBUG("Invalid absolute flip target %u, "
					  "must be <= %u\n", target_vblank,
					  current_vblank + 1);
				drm_crtc_vblank_put(crtc);
				return -EINVAL;
			}
			break;
			相对帧目标 (Relative Target VBlank)，表示 Page Flip 需要发生在 当前帧的后 target_vblank 帧。
			约束条件：
		target_vblank 只能是 0 或 1：
		0 代表 立即 Page Flip。
		1 代表 下一个 VBlank 才进行 Page Flip。
		超过 1 则返回 -EINVAL。
		case DRM_MODE_PAGE_FLIP_TARGET_RELATIVE:
			if (target_vblank != 0 && target_vblank != 1) {
				DRM_DEBUG("Invalid relative flip target %u, "
					  "must be 0 or 1\n", target_vblank);
				drm_crtc_vblank_put(crtc);
				return -EINVAL;
			}
			target_vblank += current_vblank;
			break;
		default:
		如果没有显式指定 TARGET_ABSOLUTE 或 TARGET_RELATIVE：
    如果 DRM_MODE_PAGE_FLIP_ASYNC 未设置：
    target_vblank = current_vblank + 1（等待下一个 VBlank）。
    如果 DRM_MODE_PAGE_FLIP_ASYNC 被设置：
    target_vblank = current_vblank（立即 Flip）。
			target_vblank = current_vblank +
				!(page_flip->flags & DRM_MODE_PAGE_FLIP_ASYNC);
			break;
		}
	} else if (crtc->funcs->page_flip == NULL ||
		   如果不存在这个回调函数那么就return
	}

	...
	关于plane与fb的一堆操作
	...
	
	/* 如果有event这个flag */
	/* event 与 ASYNC 没区别很明白现在 */
	if (page_flip->flags & DRM_MODE_PAGE_FLIP_EVENT) {
		e = kzalloc(sizeof *e, GFP_KERNEL);
		if (!e) {
			ret = -ENOMEM;
			goto out;
		}
		构造事件，base type是DRM_EVENT_FLIP_COMPLETE 
		因为这个字段是union，所以这两个event是一个
		e->event.base.type = DRM_EVENT_FLIP_COMPLETE;
		e->event.base.length = sizeof(e->event);
		e->event.vbl.user_data = page_flip->user_data;
		e->event.vbl.crtc_id = crtc->base.id;
/* 这个函数很关键，如下所示 */
		ret = drm_event_reserve_init(dev, file_priv, &e->base, &e->event.base);
		if (ret) {
			kfree(e);
			e = NULL;
			goto out;
		}
	}

调用驱动提供的回调	
	if (crtc->funcs->page_flip_target)
		ret = crtc->funcs->page_flip_target(crtc, fb, e,
						    page_flip->flags,
						    target_vblank,
						    &ctx);
	else
		ret = crtc->funcs->page_flip(crtc, fb, e, page_flip->flags,
					     &ctx);
...

out:
...
}
```

好吧，这个函数长的要命。先关注如下结构体和宏。

```
drm_pending_vblank_event
```

```
/**
 * struct drm_pending_vblank_event - pending vblank event tracking
 跟踪待处理的vblank事件，也就是说这个东西最后是交给vblank处理的。
 */
struct drm_pending_vblank_event {
	/**
	 * @base: Base structure for tracking pending DRM events.
	 */
	struct drm_pending_event base;
	/**
	 * @pipe: drm_crtc_index() of the &drm_crtc this event is for. event所在crtc index
	 */
	unsigned int pipe;
	/**
	 * @sequence: frame event should be triggered at 期待的frame的seq
	 */
	u64 sequence;
	/**
	 * @event: Actual event which will be sent to userspace.
	 将发送到用户空间的实际事件。
	 */
	union {
		/**
		 * @event.base: DRM event base class.
		 */
		struct drm_event base;

		/**
		 * @event.vbl:
		 *
		 * Event payload for vblank events, requested through
		 * either the MODE_PAGE_FLIP or MODE_ATOMIC IOCTL. Also
		 * generated by the legacy WAIT_VBLANK IOCTL, but new userspace
		 * should use MODE_QUEUE_SEQUENCE and &event.seq instead.
		 */
		struct drm_event_vblank vbl;

		/**
		 * @event.seq: Event payload for the MODE_QUEUEU_SEQUENCE IOCTL.
		 */
		struct drm_event_crtc_sequence seq;
	} event;
};

/**
 * Header for events written back to userspace on the drm fd.  The
 * type defines the type of event, the length specifies the total
 * length of the event (including the header), and user_data is
 * typically a 64 bit value passed with the ioctl that triggered the
 * event.  A read on the drm fd will always only return complete
 * events, that is, if for example the read buffer is 100 bytes, and
 * there are two 64 byte events pending, only one will be returned.
 *
 在drm-fd上写回用户空间的事件标题。类型定义了事件的类型，长度指定了事件的总长度（包括标头），user_data通常是触发事件的ioctl传递的64位值。drm-fd上的读取将始终只返回完整事件，也就是说，如果读取缓冲区为100字节，并且有两个64字节的待处理事件，则只返回一个。
 * Event types 0 - 0x7fffffff are generic drm events, 0x80000000 and
 * up are chipset specific.
 */
struct drm_event {
	__u32 type;
	__u32 length;
};
```

##### drm_event_reserve_init

```
/**
 * drm_event_reserve_init - init a DRM event and reserve space for it 初始化DRM事件并为其保留空间
 * @dev: DRM device
 * @file_priv: DRM file private data
 * @p: tracking structure for the pending event
 * @e: actual event data to deliver to userspace
 *
 * This function prepares the passed in event for eventual delivery. If the event
 * doesn't get delivered (because the IOCTL fails later on, before queuing up
 * anything) then the even must be cancelled and freed using
 * drm_event_cancel_free(). Successfully initialized events should be sent out
 * using drm_send_event() or drm_send_event_locked() to signal completion of the
 * asynchronous event to userspace.
 *此函数用于准备传入事件以供最终交付。如果事件未送达（因为稍后在排队之前，CTL失败），则必须使用drm_event_cancel_free（）取消并释放偶数。应使用drm_send_event（）或drm_send_exvent_locked（）向用户空间发送成功初始化的事件，以表示异步事件已完成。
 * If callers embedded @p into a larger structure it must be allocated with
 * kmalloc and @p must be the first member element.
 *
 * Callers which already hold &drm_device.event_lock should use
 * drm_event_reserve_init_locked() instead.
 *
 * RETURNS:
 *
 * 0 on success or a negative error code on failure.
 */
int drm_event_reserve_init(struct drm_device *dev,
			   struct drm_file *file_priv,
			   struct drm_pending_event *p,
			   struct drm_event *e)
{
	unsigned long flags;
	int ret;

	spin_lock_irqsave(&dev->event_lock, flags);
	ret = drm_event_reserve_init_locked(dev, file_priv, p, e);
	spin_unlock_irqrestore(&dev->event_lock, flags);

	return ret;
}
EXPORT_SYMBOL(drm_event_reserve_init);

int drm_event_reserve_init_locked(struct drm_device *dev,
				  struct drm_file *file_priv,
				  struct drm_pending_event *p,
				  struct drm_event *e)
{
	if (file_priv->event_space < e->length)
		return -ENOMEM;

	file_priv->event_space -= e->length;

	p->event = e;
	list_add(&p->pending_link, &file_priv->pending_event_list);
	p->file_priv = file_priv;

	return 0;
}
EXPORT_SYMBOL(drm_event_reserve_init_locked);
```

可以看到是将`drm_event`加入到如下地方。

```
/**
 * struct drm_file - DRM file private data
 *
 * This structure tracks DRM state per open file descriptor.
 */
struct drm_file {
...
	/**
	 * @pending_event_list:
	 *
	 * List of pending &struct drm_pending_event, used to clean up pending
	 * events in case this file gets closed before the event is signalled.
	 * Uses the &drm_pending_event.pending_link entry.
	 存储尚未完成的事件（pending events）。这些事件仍然在等待某些操作的完成，比如：等待一个异步操作（如 vsync 中断）完成。还没有达到可以交付给用户空间的时机。如果 drm_file 关闭了（比如进程终止），pending_event_list 里的所有事件都应该被清理，避免资源泄露。
	 *
	 * Protect by &drm_device.event_lock.
	 */
	struct list_head pending_event_list;
		/**
	 * @event_list:
	 *
	 * List of &struct drm_pending_event, ready for delivery to userspace
	 * through drm_read(). Uses the &drm_pending_event.link entry.
	 *
	 * Protect by &drm_device.event_lock.
	 存储已经准备好发送给用户空间的事件。这些事件通常是：某个异步操作已经完成，比如 vsync、帧渲染完成等。事件数据已经填充完毕，可以直接交付给用户空间。
	 */
	struct list_head event_list;
...
}
```

可以看到drm_event_reserve_init中，讲page-flip触发的event放到了pending_event_list。

##### crtc->funcs->page_flip

```
static const struct drm_crtc_funcs vop2_crtc_funcs = {
	.gamma_set = vop2_crtc_legacy_gamma_set,
	.set_config = drm_atomic_helper_set_config,
	.page_flip = drm_atomic_helper_page_flip,
...
};
```

可以看到rockchip使用的是原生的helper。

```
drm_atomic_helper_page_flip
```

```
int drm_atomic_helper_page_flip(struct drm_crtc *crtc,
				struct drm_framebuffer *fb,
				struct drm_pending_vblank_event *event,
				uint32_t flags,
				struct drm_modeset_acquire_ctx *ctx)
{
	struct drm_plane *plane = crtc->primary;
	struct drm_atomic_state *state;
	int ret = 0;

	state = drm_atomic_state_alloc(plane->dev);
	if (!state)
		return -ENOMEM;

	state->acquire_ctx = ctx;
	
	ret = page_flip_common(state, crtc, fb, event, flags);
	if (ret != 0)
		goto fail;

	ret = drm_atomic_nonblocking_commit(state);
fail:
	drm_atomic_state_put(state);
	return ret;
}
EXPORT_SYMBOL(drm_atomic_helper_page_flip);
```

```
static int page_flip_common(struct drm_atomic_state *state,
			    struct drm_crtc *crtc,
			    struct drm_framebuffer *fb,
			    struct drm_pending_vblank_event *event,
			    uint32_t flags)
{
	struct drm_plane *plane = crtc->primary;
	struct drm_plane_state *plane_state;
	struct drm_crtc_state *crtc_state;
	int ret = 0;

	crtc_state = drm_atomic_get_crtc_state(state, crtc);
	if (IS_ERR(crtc_state))
		return PTR_ERR(crtc_state);
	/* 在这里可以看到这个crtc_state的event与上述的drm_file的pending_event是一个指针 */
	crtc_state->event = event;
	crtc_state->pageflip_flags = flags;

	plane_state = drm_atomic_get_plane_state(state, plane);
	if (IS_ERR(plane_state))
		return PTR_ERR(plane_state);

	ret = drm_atomic_set_crtc_for_plane(plane_state, crtc);
	if (ret != 0)
		return ret;
	drm_atomic_set_fb_for_plane(plane_state, fb);

	/* Make sure we don't accidentally do a full modeset. */
	state->allow_modeset = false;
	if (!crtc_state->active) {
		DRM_DEBUG_ATOMIC("[CRTC:%d:%s] disabled, rejecting legacy flip\n",
				 crtc->base.id, crtc->name);
		return -EINVAL;
	}

	return ret;
}
```

```
/**
 * drm_atomic_nonblocking_commit - atomic nonblocking commit
 * @state: atomic configuration to check
 *
 * Note that this function can return -EDEADLK if the driver needed to acquire
 * more locks but encountered a deadlock. The caller must then do the usual w/w
 * backoff dance and restart. All other errors are fatal.
 *
 * This function will take its own reference on @state.
 * Callers should always release their reference with drm_atomic_state_put().
 *
 * Returns:
 * 0 on success, negative error code on failure.
 */
int drm_atomic_nonblocking_commit(struct drm_atomic_state *state)
{
	struct drm_mode_config *config = &state->dev->mode_config;
	int ret;

	ret = drm_atomic_check_only(state);
	if (ret)
		return ret;

	DRM_DEBUG_ATOMIC("committing %p nonblocking\n", state);
	/* wapper for atomic_commit */
	return config->funcs->atomic_commit(state->dev, state, true);
}
EXPORT_SYMBOL(drm_atomic_nonblocking_commit);
```

在vop2_crtc_atomic_flush中，vp->event = crtc->state->event，然后在vop2_handle_vblank上述vblank章节说在isr中被触发，然后触发drm_crtc_send_vblank_event。处理pending_event。

```
static void vop2_handle_vblank(struct vop2 *vop2, struct drm_crtc *crtc)
{
	struct drm_device *drm = vop2->drm_dev;
	struct vop2_video_port *vp = to_vop2_video_port(crtc);
	unsigned long flags;

	spin_lock_irqsave(&drm->event_lock, flags);
	if (vp->event) {
		drm_crtc_send_vblank_event(crtc, vp->event);
		drm_crtc_vblank_put(crtc);
		vp->event = NULL;
	}
	spin_unlock_irqrestore(&drm->event_lock, flags);

	if (test_and_clear_bit(VOP_PENDING_FB_UNREF, &vp->pending))
		drm_flip_work_commit(&vp->fb_unref_work, system_unbound_wq);
}
```

在这有个函数，是产生假的vblank事件，相当于软件的vblank跳动。

```
/**
 * drm_atomic_helper_fake_vblank - fake VBLANK events if needed
 * @old_state: atomic state object with old state structures
 *
 * This function walks all CRTCs and fake VBLANK events on those with
 * &drm_crtc_state.no_vblank set to true and &drm_crtc_state.event != NULL.
 * The primary use of this function is writeback connectors working in oneshot
 * mode and faking VBLANK events. In this case they only fake the VBLANK event
 * when a job is queued, and any change to the pipeline that does not touch the
 * connector is leading to timeouts when calling
 * drm_atomic_helper_wait_for_vblanks() or
 * drm_atomic_helper_wait_for_flip_done().
 *
 * This is part of the atomic helper support for nonblocking commits, see
 * drm_atomic_helper_setup_commit() for an overview.
 */
void drm_atomic_helper_fake_vblank(struct drm_atomic_state *old_state)
{
	struct drm_crtc_state *new_crtc_state;
	struct drm_crtc *crtc;
	int i;

	for_each_new_crtc_in_state(old_state, crtc, new_crtc_state, i) {
		unsigned long flags;

		if (!new_crtc_state->no_vblank)
			continue;

		spin_lock_irqsave(&old_state->dev->event_lock, flags);
		if (new_crtc_state->event) {
			drm_crtc_send_vblank_event(crtc,
						   new_crtc_state->event);
			new_crtc_state->event = NULL;
		}
		spin_unlock_irqrestore(&old_state->dev->event_lock, flags);
	}
}
EXPORT_SYMBOL(drm_atomic_helper_fake_vblank);
```

可以看到也是处理了drm_crtc_send_vblank_event事件。那么这个事件如何处理的？

##### drm_crtc_send_vblank_event

```
/**
 * drm_crtc_send_vblank_event - helper to send vblank event after pageflip
 * @crtc: the source CRTC of the vblank event
 * @e: the event to send
 *
 * Updates sequence # and timestamp on event for the most recently processed
 * vblank, and sends it to userspace.  Caller must hold event lock.
 *
 * See drm_crtc_arm_vblank_event() for a helper which can be used in certain
 * situation, especially to send out events for atomic commit operations.
 */
void drm_crtc_send_vblank_event(struct drm_crtc *crtc,
				struct drm_pending_vblank_event *e)
{
	struct drm_device *dev = crtc->dev;
	u64 seq;
	unsigned int pipe = drm_crtc_index(crtc);
	ktime_t now;

	if (dev->num_crtcs > 0) {
		seq = drm_vblank_count_and_time(dev, pipe, &now);
	} else {
		seq = 0;

		now = ktime_get();
	}
	e->pipe = pipe;
	send_vblank_event(dev, e, seq, now);
}
EXPORT_SYMBOL(drm_crtc_send_vblank_event);
```

```
static void send_vblank_event(struct drm_device *dev,
		struct drm_pending_vblank_event *e,
		u64 seq, ktime_t now)
{
	struct timespec64 tv;

	switch (e->event.base.type) {
	case DRM_EVENT_VBLANK:
	case DRM_EVENT_FLIP_COMPLETE:
		tv = ktime_to_timespec64(now);
		e->event.vbl.sequence = seq;
		/*
		 * e->event is a user space structure, with hardcoded unsigned
		 * 32-bit seconds/microseconds. This is safe as we always use
		 * monotonic timestamps since linux-4.15
		 */
		e->event.vbl.tv_sec = tv.tv_sec;
		e->event.vbl.tv_usec = tv.tv_nsec / 1000;
		break;
	case DRM_EVENT_CRTC_SEQUENCE:
		if (seq)
			e->event.seq.sequence = seq;
		e->event.seq.time_ns = ktime_to_ns(now);
		break;
	}
	trace_drm_vblank_event_delivered(e->base.file_priv, e->pipe, seq);
	drm_send_event_locked(dev, &e->base);
}
```

```

/**
 * drm_send_event_locked - send DRM event to file descriptor
 * @dev: DRM device
 * @e: DRM event to deliver
 *
 * This function sends the event @e, initialized with drm_event_reserve_init(),
 * to its associated userspace DRM file. Callers must already hold
 * &drm_device.event_lock, see drm_send_event() for the unlocked version.
 *
 * Note that the core will take care of unlinking and disarming events when the
 * corresponding DRM file is closed. Drivers need not worry about whether the
 * DRM file for this event still exists and can call this function upon
 * completion of the asynchronous work unconditionally.
 */
void drm_send_event_locked(struct drm_device *dev, struct drm_pending_event *e)
{
	assert_spin_locked(&dev->event_lock);

	if (e->completion) {
		complete_all(e->completion);
		e->completion_release(e->completion);
		e->completion = NULL;
	}

	if (e->fence) {
		dma_fence_signal(e->fence);
		dma_fence_put(e->fence);
	}

	if (!e->file_priv) {
		kfree(e);
		return;
	}

	list_del(&e->pending_link);
	list_add_tail(&e->link,
		      &e->file_priv->event_list);
	wake_up_interruptible(&e->file_priv->event_wait);
}
EXPORT_SYMBOL(drm_send_event_locked);

```

这些completion，fence后续会介绍。

可以看到page-flip的功能不止于此，后续在commit中会提及。

## 测试程序

```
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <time.h>
#include <unistd.h>
#include <signal.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

struct buffer_object {
	uint32_t width;
	uint32_t height;
	uint32_t pitch;
	uint32_t handle;
	uint32_t size;
	uint32_t *vaddr;
	uint32_t fb_id;
};

struct buffer_object buf[2];
static int terminate;

static int modeset_create_fb(int fd, struct buffer_object *bo, uint32_t color)
{
	struct drm_mode_create_dumb create = {};
 	struct drm_mode_map_dumb map = {};
	uint32_t i;

	create.width = bo->width;
	create.height = bo->height;
	create.bpp = 32;
	drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

	bo->pitch = create.pitch;
	bo->size = create.size;
	bo->handle = create.handle;
	drmModeAddFB(fd, bo->width, bo->height, 24, 32, bo->pitch,
			   bo->handle, &bo->fb_id);

	map.handle = create.handle;
	drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);

	bo->vaddr = mmap(0, create.size, PROT_READ | PROT_WRITE,
			MAP_SHARED, fd, map.offset);

	for (i = 0; i < (bo->size / 4); i++)
		bo->vaddr[i] = color;

	return 0;
}

static void modeset_destroy_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_destroy_dumb destroy = {};

	drmModeRmFB(fd, bo->fb_id);

	munmap(bo->vaddr, bo->size);

	destroy.handle = bo->handle;
	drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

static void modeset_page_flip_handler(int fd, uint32_t frame,
				    uint32_t sec, uint32_t usec,
				    void *data)
{
	static int i = 0;
	uint32_t crtc_id = *(uint32_t *)data;

	i ^= 1;

	drmModePageFlip(fd, crtc_id, buf[i].fb_id,
			DRM_MODE_PAGE_FLIP_EVENT, data);

	usleep(500000);
}

static void sigint_handler(int arg)
{
	terminate = 1;
}

int main(int argc, char **argv)
{
	int fd;
	drmEventContext ev = {};
	drmModeConnector *conn;
	drmModeRes *res;
	uint32_t conn_id;
	uint32_t crtc_id;

	/* register CTRL+C terminate interrupt */
	signal(SIGINT, sigint_handler);

	ev.version = DRM_EVENT_CONTEXT_VERSION;
	ev.page_flip_handler = modeset_page_flip_handler;

	fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);
/* 需要修改 */
	res = drmModeGetResources(fd);
	crtc_id = res->crtcs[1];
	conn_id = res->connectors[1];

	conn = drmModeGetConnector(fd, conn_id);
	buf[0].width = conn->modes[0].hdisplay;
	buf[0].height = conn->modes[0].vdisplay;
	buf[1].width = conn->modes[0].hdisplay;
	buf[1].height = conn->modes[0].vdisplay;

	modeset_create_fb(fd, &buf[0], 0xff0000);
	modeset_create_fb(fd, &buf[1], 0x0000ff);

	drmModeSetCrtc(fd, crtc_id, buf[0].fb_id,
			0, 0, &conn_id, 1, &conn->modes[0]);

	drmModePageFlip(fd, crtc_id, buf[0].fb_id,
			DRM_MODE_PAGE_FLIP_EVENT, &crtc_id);

	while (!terminate) {
		drmHandleEvent(fd, &ev);
	}

	modeset_destroy_fb(fd, &buf[1]);
	modeset_destroy_fb(fd, &buf[0]);

	drmModeFreeConnector(conn);
	drmModeFreeResources(res);

	close(fd);

	return 0;
}
```

