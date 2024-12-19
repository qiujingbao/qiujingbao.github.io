---
layout: post

title: Linux Kernel Lock
categories: [Linux,Kernel]
tags: [Linux,Kernel,OS,Lock]
typora-root-url: ..
---

## Linux-锁的实现

概念：事实上，同步原语是一种软件机制，提供了两个或者多个[并行](https://en.wikipedia.org/wiki/Parallel_computing)进程或者线程在不同时刻执行一段相同的代码段的能力。

### 自旋锁

​	自旋锁是为了保护共享资源，多个CPU使用的时候，仅仅同时允许一个CPU访问。

​	自旋锁的定义：当一个线程尝试去获取某一把锁的时候，如果这个锁此时已经被别人获取(占用)，那么此线程就无法获取到这把锁，该线程将会等待，间隔一段时间后会再次尝试获取。这种采用循环加锁 -> 等待的机制被称为`自旋锁(spinlock)`。

##### **原始自旋锁**

```
struct self_spinlock {
	__u32 lock;
};

static inline void self_spinlock(struct self_spinlock *lock)
{
	while (__atomic_test_and_set(&lock->lock, __ATOMIC_ACQUIRE))
		while (__atomic_load_n(&lock->lock, __ATOMIC_RELAXED))
			;
}

static inline void self_unspinlock(struct self_spinlock *lock)
{
	__atomic_clear(&lock->lock, __ATOMIC_RELEASE);
}

```

​	对于每个CPU来说，总会不断的原子执行test_and_set指令。这会导致两个问题。

1. 不公平，先来的不一定先服务。
2. 因为最终内容在内存上，但是每个cpu都会有cache，所以对导致多核之间cache的影响。

```
+------+    +------+    +------+    +------+
| CPU0 |    | CPU1 |    | CPU2 |    | CPU3 |
+------+    +------+    +------+    +------+
   v           v           v           v
+------+    +------+    +------+    +------+
|cache0|    |cache1|    |cache2|    |cache3|
+------+    +------+    +------+    +------+
         \      \          /     /
          \  +----------------+ /
           \ | Flag in memory |/
             +----------------+
```

​	对于第二个问题，我们展开看下，在有cache的系统里，系统大概的样子如上，如果CPU0占有锁，cach0为1(cache0/1/2/3是Flag在各个core上的cache)，CPU1/2/3的cache也会在各个core读Flag时被设置为1，CPU0释放锁的时候，cache0被写成0，同时CPU1/2/3上对应的cach被无效化，随后哪个core抢先把Flag写成1，对应的cache就是1。后面重复之前的逻辑。可以看出，本来在unlock core和lock core之间的通行行为被扩展到了所有参与竞争锁的core，不但锁的请求顺序和实际获得锁的顺序不一致，而且做了很多无用功。

​	也就是当其中某个持有锁的cpu放弃cpu的时候，会修改cache，对应的其他CPU的cache需要重新写入来保持一致。

​	我们希望，只有释放锁的cpu和获得（排队）锁的CPU进行操作，其他cpu等待即可。

##### **票据自旋锁**

​	tick spinlock自旋锁的结构体如下

```
struct __raw_tickets {
	u16 next;
	u16 owner;
}
```

**获取锁**

​	获取锁的行为就是原子的增加next值，然后作为自己的ticket，拿着自己的ticket一直和owner做对比，看看是不是轮到自己拿锁。也就是共享这个变量，另外保存一个ticket值。

**释放锁**

​	释放锁的行为就是增加owner的值。

```
                        +------+    +------+    +------+    +------+
                        | CPU0 |    | CPU1 |    | CPU2 |    | CPU3 |
                        +------+    +------+    +------+    +------+

        lock               ----------------------------------------> 
  local next:              0           1       +---2---+   +---3---+
                                               |       |   |       |
                                     owner++   ^       v   ^       v
      unlock                         ----->    |       |   |       |
owner in __raw_tickets:    0           1       +-owner-+   +-owner-+

```

​	如上是一个ticket spinlock的示意图，CPU2/3现在在等待锁，CPU1在释放锁。试图获取锁
的CPU原子的对next加1并在本地保存一个加1后的本地next，作为自己的ticket，释放锁的
CPU把锁里的owner值加1，试图获取锁的CPU，一直在拿自己的ticket和owner做比较，如果
ticket和owner相等自己就拿到了锁。

​	可以看出，ticket spinlock解决了上面的问题1，但是没有解决问题2，因为竞争锁的core
必须时刻拿着自己的ticket和owner做对比，实际上谁得到锁这个信息只要依次传递就好。

##### **MCS自旋锁**

​	MCS锁想要解决cache不一致的问题，所以说对于每个cpu都保存自己的副本即可。

​	其实从上面ticket spinlock的图上我们已经可以大概看出来要怎么做，就是把每次lock搞一个锁的副本出来，然后把这些副本用链表链接起来，加锁时还是spin在自己的副本上，解锁时顺着链表依次释放锁。

```
struct mcs_spinlock {
	struct mcs_spinlock *next;
	int locked;
	int count;
};
```

```
                                      ------------+
                                      | lock tail |
                                    / +-----------+
+------+    +------+    +------+   /+------+
| CPU0 |    | CPU1 |    | CPU2 |  / | CPU3 |
+------+    +------+    +------+ /  +------+
                              1 /        +-----+
                               /         |     | 3 spin on owner
+------+    +------+    +------+    +------+   |
| owner|    | owner|    | owner| 2  | owner|<--+
| next |--->| next |--->| next |--->| next |
+------+    +------+    +------+    +------+
```

**加锁**

​	如上所示，加锁就是找见当前锁链表结尾(步骤1)，把要加的锁节点挂在上面(步骤2)，然后
就spin在自己的owner标记上等待(步骤3)。

**解锁**

​	解锁就是把当前锁节点的下一个节点的owner配置成1，这样，spin的core检测到owner为1，
就知道现在自己拥有锁了。

​	MCS锁可以解决上面的两个问题，但是占用的内存比较多了。我们考虑MCS的实际实现问题，
一个MCS锁需要一个lock tail以及每个core上的mcs node结构，占用内存比较大，实际上在现在的Linux内核中只有MCS node的定义，并没有MCS锁的实现，只所以MCS的定义还在，是因为qspinlock里要复用MCS node的定义。

##### **队列自旋锁**

​	定义如下

```
typedef struct qspinlock {
	union {
		atomic_t val;
		struct {
			u8	locked;
			u8	pending;
		};
		struct {
			u16	locked_pending;
			u16	tail;
		};
	};
} arch_spinlock_t;
```

内存分布如下

```
(tail,          pending, locked)
<----16bit----><----16bit-----> 
```

自旋锁结构的大小只占用了4个字节，这主要归功于将指向MCS锁内的指针变量用一个16位的tail域来取代。tail如何取代mcs的？

```
struct qnode {
	struct mcs_spinlock mcs;
#ifdef CONFIG_PARAVIRT_SPINLOCKS
	long reserved[2];
#endif
};

static DEFINE_PER_CPU_ALIGNED(struct qnode, qnodes[MAX_NODES]);
```

对于每个CPU都有一个qnode数组，此数组大小为4个。qnode就是mcs锁的结构体。使用CPU编号和节点编号就可以确定唯一一个MCS结构体。所以如图所示。

![img](./assets/pics/202406072314393.png)

**上锁**

使用tail、pending和locked字段作为一个三元向量，分析这个三元向量。



第一个初始化并上锁，locked=1，pending=0。

```
+-------+------------+----------+
| tail  |  pending 0 | locked 1 |
+-------+------------+----------+
```

第二个上锁，此时锁在CPU1上，locked=0，pending=1。

```
                             +------+
                             |      |
+-------+------------+----------+   |  spin check locked
| tail  |  pending 1 | locked 0 |<--+
+-------+------------+----------+
```

1. 如果qspinlock整体val为0，说明锁空闲，则当前CPU设置qspinlock的locked位为1后直接持锁；
2. 第1个等锁的CPU设置pending位后，自旋等待locked位变成0； 
3. 第2个等锁的CPU将tail设置为指向本CPU变量的mcs_spinlock节点，然后自旋等待locked域和pending位都变成0； 
4. 第N个等锁的CPU也将tail设置为指向本CPU变量的mcs_spinlock节点，并将之前队尾节点的next指向自己，然后自旋转等待本CPU变量的mcs_spinlock节点中的locked域变成1。 

代码复杂的原因是多CPU竞争必须小心处理，然后尽可能的优化。

**解锁**

```
static __always_inline void queued_spin_unlock(struct qspinlock *lock)
{
	/* 将自旋锁的locked域设置成0 */
	smp_store_release(&lock->locked, 0);
}
```

可以看到，只是简单的将自旋锁的locked域设置成0就行了。因为其他CPU一直在while check，所以一旦解锁其他CPU就可以去获得锁了。



### 信号量

```
struct semaphore {
    raw_spinlock_t        lock;
    unsigned int        count;
    struct list_head    wait_list;
};
```

在内核中， `信号量` 结构体由三部分组成：

- `lock` - 保护 `信号量` 的 `自旋锁`;
- `count` - 现有资源的数量;
- `wait_list` - 等待获取此锁的进程序列.

自旋获得lock，然后修改count，如果count不够放入wait list休眠等待。如果获得不了lock则休眠等待

**自旋锁适用于多CPU系统，并不是适用于单CPU系统**

​	自旋锁不适合单处理器的原因是由于当一个进程处于自旋锁状态时说明其正在等待某一个资源，而在单处理器系统中，一旦发生了这种自旋锁，那么拥有资源的那个进程也会由于无法得到CPU而一直等待，这样之后，占用CPU的进程会等待另一个进程释放资源，占用资源的进程会等待另一个进程释放CPU，从而导致程序进入一个死锁的状态，因此单处理器不适合自旋锁
​	在多处理器中，即使一个进程处于自旋锁状态而占用一个CPU，拥有资源的进程也可以在另外空闲的处理器上执行，直到资源释放，处于忙等的进程这时便可以正常进行，防止了死锁的发生

#### 乐观锁

- 乐观锁：乐观锁在操作数据时非常乐观，认为别人不会同时修改数据。因此乐观锁不会上锁，只是在执行更新的时候判断一下在此期间别人是否修改了数据：如果别人修改了数据则放弃操作，否则执行操作。
- 悲观锁：悲观锁在操作数据时比较悲观，认为别人会同时修改数据。因此操作数据时直接把数据锁住，直到操作完成后才会释放锁；上锁期间其他人不能修改数据。

#### osq流程分析

`optimistic spinning`，乐观自旋，到底有多乐观呢？当发现锁被持有时，`optimistic spinning`相信持有者很快就能把锁释放，因此它选择自旋等待，而不是睡眠等待，这样也就能减少进程切换带来的开销了。

看一下数据结构吧：

![img](./assets/pics/202406072314058.png)

![img](./assets/pics/202406072314630.png)

- osq加锁有几种情况：
  1. 无人持有锁，那是最理想的状态，直接返回；
  2. 有人持有锁，将当前的Node加入到OSQ队列中，在没有高优先级任务抢占时，自旋等待前驱节点释放锁；
  3. 自旋等待过程中，如果遇到高优先级任务抢占，那么需要做的事情就是将之前加入到OSQ队列中的当前节点，从OSQ队列中移除，移除的过程又分为三个步骤，分别是处理prev前驱节点的next指针指向、当前节点Node的next指针指向、以及将prev节点与next后继节点连接；
- 加锁过程中使用了原子操作，来确保正确性；

### 互斥锁

- `Mutex`互斥锁是Linux内核中用于互斥操作的一种同步原语；
- 互斥锁是一种休眠锁，锁争用时可能存在进程的睡眠与唤醒，context的切换带来的代价较高，适用于加锁时间较长的场景；
- 互斥锁每次只允许一个进程进入临界区，有点类似于二值信号量；
- 互斥锁在锁争用时，在锁被持有时，选择自选等待，而不立即进行休眠，可以极大的提高性能，这种机制（`optimistic spinning`）也应用到了读写信号量上；
- 互斥锁的缺点是互斥锁对象的结构较大，会占用更多的CPU缓存和内存空间；
- 与信号量相比，互斥锁的性能与扩展性都更好，因此，在内核中总是会优先考虑互斥锁；
- 互斥锁按为了提高性能，提供了三条路径处理：快速路径，中速路径，慢速路径；

使用注意事项

- 一次只能有一个进程能持有互斥锁；
- 只有锁的持有者能进行解锁操作；
- 禁止多次解锁操作；
- 禁止递归加锁操作；
- mutex结构只能通过API进行初始化；
- mutex结构禁止通过`memset`或者拷贝来进行初始化；
- 已经被持有的mutex锁禁止被再次初始化；
- mutex不允许在硬件或软件上下文（`tasklets, timer`）中使用；

从`mutex_lock`加锁来看一下大概的流程：

![img](./assets/pics/1771657-20200504154828438-1431928421.png)

- `mutex_lock`为了提高性能，分为三种路径处理，优先使用快速和中速路径来处理，如果条件不满足则会跳转到慢速路径来处理，慢速路径中会进行睡眠和调度，因此开销也是最大的。

##### 3.2.1 fast-path

- 快速路径是在`__mutex_trylock_fast`中实现的，该函数的实现也很简单，直接调用`atomic_long_cmpxchg_release(&lock->owner, 0UL, curr)`函数来进行判断，如果`lock->owner == 0`表明锁未被持有，将`curr`赋值给`lock->owner`标识`curr`进程持有该锁，并直接返回；
- `lock->owner`不等于0，表明锁被持有，需要进入下一个路径来处理了；

##### 3.2.2 mid-path

- 中速路径和慢速路径的处理都是在`__mutex_lock_common`中实现的；
- `__mutex_lock_common`的传入参数为(`lock, TASK_INTERRUPTIBLE, 0, NULL, _RET_IP_, false`)，该函数中很多路径覆盖不到，接下来的分析也会剔除掉无效代码；

中速路径的核心代码如下：

![img](./assets/pics/202406072314271.png)

- 当发现mutex锁的持有者正在运行（另一个CPU）时，可以不进行睡眠调度，而可以选择自选等待，当锁持有者正在运行时，它很有可能很快会释放锁，这个就是乐观自旋的原因；
- 自旋等待的条件是持有锁者正在临界区运行，自旋等待才有价值；
- `__mutex_trylock_or_owner`函数用于尝试获取锁，如果获取失败则返回锁的持有者。互斥锁的结构体中`owner`字段，分为两个部分：1）锁持有者进程的task_struct（由于L1_CACHE_BYTES对齐，低位比特没有使用）；2）`MUTEX_FLAGS`部分，也就是对应低三位，如下：
  1. `MUTEX_FLAG_WAITERS`：比特0，标识存在非空等待者链表，在解锁的时候需要执行唤醒操作；
  2. `MUTEX_FLAG_HANDOFF`：比特1，表明解锁的时候需要将锁传递给顶部的等待者；
  3. `MUTEX_FLAG_PICKUP`：比特2，表明锁的交接准备已经做完了，可以等待被取走了；
- `mutex_optimistic_spin`用于执行乐观自旋，理想的情况下锁持有者执行完释放，当前进程就能很快的获取到锁。实际需要考虑，如果锁的持有者如果在临界区被调度出去了，`task_struct->on_cpu == 0`，那么需要结束自旋等待了，否则岂不是傻傻等待了。
  1. `mutex_can_spin_on_owner`：进入自旋前检查一下，如果当前进程需要调度，或者锁的持有者已经被调度出去了，那么直接就返回了，不需要做接下来的`osq_lock/oqs_unlock`工作了，节省一些额外的overhead；
  2. `osq_lock`用于确保只有一个等待者参与进来自旋，防止大量的等待者蜂拥而至来获取互斥锁；
  3. `for(;;)`自旋过程中调用`__mutex_trylock_or_owner`来尝试获取锁，获取到后皆大欢喜，直接返回即可；
  4. `mutex_spin_on_owner`，判断不满足自旋等待的条件，那么返回，让我们进入慢速路径吧，毕竟不能强求；

##### 3.2.3 slow-path

慢速路径的主要代码流程如下：

![img](./assets/pics/1771657-20200504154912494-504587706.png)

- 从`for(;;)`部分的流程可以看到，当没有获取到锁时，会调用`schedule_preempt_disabled`将本身的任务进行切换出去，睡眠等待，这也是它慢的原因了；

##### 3.3 释放锁流程分析

![img](./assets/pics/202406072314467.png)

- 释放锁的流程相对来说比较简单，也分为快速路径与慢速路径，快速路径只有在调试的时候打开；
- 慢速路径释放锁，针对三种不同的`MUTEX_FLAG`来进行判断处理，并最终唤醒等待在该锁上的任务；

### rwlock 读写锁

![img](./assets/pics/202406072314518.png)