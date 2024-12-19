---
layout: post

title: Linux SMP 争用
categories: [kernel]
tags: [C,八股]
typora-root-url: ..
---


## Linux SMP

**CPU：** 本文中的CPU都是指逻辑CPU。
**UP：** 单处理器(单CPU)。
**SMP：** 对称多处理器(多CPU)。
**线程、执行流：** 线程的本质是一个执行流，但执行流不仅仅有线程，还有ISR、softirq、tasklet，都是执行流。本文中说到线程一般是泛指各种执行流，除非是在需要区分不同执行流时，线程才特指狭义的线程。
**并发、并行：** 并发是指线程在宏观上表现为同时执行，微观上可能是同时执行也可能是交错执行，并行是指线程在宏观上是同时执行，微观上也是同时执行。
**伪并发、真并发：** 伪并发就是微观上是交错执行的并发，真并发就是并行。UP上只有伪并发，SMP上既有伪并发也有真并发。
**临界区：** 访问相同数据的代码段，如果可能会在多个线程中并发执行，就叫做临界区，临界区可以是一个代码段被多个线程并发执行，也可以是多个不同的代码段被多个线程并发执行。
**同步：** 首先线程同步的同步和同步异步的同步，不是一个意思。线程同步的同步，本文按照字面意思进行解释，同步就是统一步调、同时执行的意思。
**线程同步现象：** 线程并发过程中如果存在临界区并发执行的情况，就叫做线程同步现象。
**线程防同步机制：** 如果发生线程同步现象，由于临界区会访问共同的数据，程序可能就会出错，因此我们要防止发生线程同步现象，也就是防止临界区并发执行的情况，为此我们采取的防范措施叫做线程防同步机制。



### 概念

​	SMP (Symmetric Multi Processing),对称多处理系统内有许多紧耦合多处理器，在这样的系统中，所有的CPU共享全部资源，如总线，内存和I/O系统等，操作系统或管理数据库的复本只有一个，这种系统有一个最大的特点就是共享所有资源。多个CPU之间没有区别，平等地访问内存、外设、一个操作系统。操作系统管理着一个队列，每个处理器依次处理队列中的进程。如果两个处理器同时请求访问一个资源（例如同一段内存地址），由硬件、软件的锁机制去解决资源争用问题。

​	对于 SMP 服务器而言，每一个共享的环节都可能造成 SMP 服务器扩展时的瓶颈，而最受限制的则是内存。由于每个 CPU 必须通过相同的内存总线访问相同的内存资源，因此随着 CPU 数量的增加，内存访问冲突将迅速增加，最终会造成 CPU 资源的浪费，使 CPU 性能的有效性大大降低。实验证明， SMP 服务器 CPU 利用率最好的情况是 2 至 4 个 CPU 。

![img](./assets/pics/688602f2036eee704e4471a30c6e523f1e46f3.jpg)

​	由于 SMP 在扩展能力上的限制，人们开始探究如何进行有效地扩展从而构建大型系统的技术， NUMA 就是这种努力下的结果之一。利用 NUMA 技术，可以把几十个 CPU( 甚至上百个 CPU) 组合在一个服务器内。其 CPU 模块结构如图 2 所示：

![img](./assets/pics/b87d6594183c84b5b16374ab52554a0c3f909a.jpg)

​	NUMA 服务器的基本特征是具有多个 CPU 模块，每个 CPU 模块由多个 CPU( 如 4 个 ) 组成，并且具有独立的本地内存、 I/O 槽口等。由于其节点之间可以通过互联模块 ( 如称为 Crossbar Switch) 进行连接和信息交互，因此每个 CPU 可以访问整个系统的内存 ( 这是 NUMA 系统与 MPP 系统的重要差别 ) 。显然，访问本地内存的速度将远远高于访问远地内存 ( 系统内其它节点的内存 ) 的速度，这也是非一致存储访问 NUMA 的由来。由于这个特点，为了更好地发挥系统性能，开发应用程序时需要尽量减少不同 CPU 模块之间的信息交互。

### 多进程与多线程

略

### 单CPU与多CPU

​	如果整个计算机系统中只有一个CPU，那么照样可以拥有多个进程执行。但是每一刻只能有一个进程占用处理器。

并行：多个CPU实例或多台机器同时执行一段处理逻辑，是真正的同时。

并发：通过CPU调度算法，让用户看上去同时执行，实际上CPU操作层面不是真正的同时。

​	所谓进程执行，指的是CPU逐条执行某个进程的指令。那么当单个CPU执行的时候，多个进程可能访问共享资源，避免不了资源竞争而导致数据错乱的问题。当拥有多个CPU时，那么多个进程并行的时候也会出现相同的问题。那么多个进程争用资源的问题为什么需要提及CPU？因为下面是那kernel举例。高级语言中提供了一些具体抽象，只需要解决不同进程之间争夺资源的问题即可，所以在高级语言使用者的严重，并没有多个CPU的问题，他们只需要考虑多个进程之间资源的争用情况即可。但是在上述SMP与NUMA的架构中不仅需要解决单CPU多进程之间的问题，还需要考虑多进程并行，其实也就是多CPU资源竞争的问题。让我们回归进程的本质--一组指令序列考虑问题。

​	我们的程序逻辑经常遇到这样的操作序列：

1. 读一个位于memory中的变量的值到寄存器中
2. 修改该变量的值（也就是修改寄存器中的值）
3. 将寄存器中的数值写回memory中的变量值

​	如果这个操作序列是串行化的操作（在一个thread中串行执行），那么一切OK，然而，世界总是不能如你所愿。在多CPU体系结构中，运行在两个CPU上的两个内核控制路径同时并行执行上面操作序列，有可能发生下面的场景：

| CPU1上的操作 | CPU2上的操作 |
| ------------ | ------------ |
| 读操作       |              |
|              | 读操作       |
| 修改         | 修改         |
| 写操作       |              |
|              | 写操作       |

​	多个CPUs和memory chip是通过总线互联的，在任意时刻，只能有一个总线master设备（例如CPU、DMA controller）访问该Slave设备（在这个场景中，slave设备是RAM chip）。因此，来自两个CPU上的读memory操作被串行化执行，分别获得了同样的旧值。完成修改后，两个CPU都想进行写操作，把修改的值写回到memory。但是，硬件arbiter的限制使得CPU的写回必须是串行化的，因此CPU1首先获得了访问权，进行写回动作，随后，CPU2完成写回动作。在这种情况下，CPU1的对memory的修改被CPU2的操作覆盖了，因此执行结果是错误的。

​	不仅是多CPU，在单CPU上也会由于有多个内核控制路径的交错而导致上面描述的错误。同时单CPU多个进程也是这样的问题，多个进程访问一个共享资源。

​	一个具体的例子如下：

| 系统调用的控制路径 | 中断handler控制路径 |
| ------------------ | ------------------- |
| 读操作             |                     |
|                    | 读操作              |
|                    | 修改                |
|                    | 写操作              |
| 修改               |                     |
| 写操作             |                     |

​	系统调用的控制路径上，完成读操作后，硬件触发中断，开始执行中断handler。这种场景下，中断handler控制路径的写回的操作被系统调用控制路径上的写回覆盖了，结果也是错误的。

​	为了防止上述的情况，解决方案就是增加原子操作。所谓原子操作，就是一组操作的序列，那么kernel中面对多个CPU争用的情况如何设计的原子操作呢？



### 指令重排与内存屏障

#### 指令重排问题

![module.jpg](./assets/pics/module.jpg)

##### 1.1 指令重排

每个 CPU 运行一个程序，程序的执行产生内存访问操作。在这个抽象 CPU 中，内存操作的顺序是松散的，CPU 假定进程间不依靠内存直接通信，在不改变程序执行 结果的推测下由自己方便的顺序执行内存访问操作。

例如，考虑下面的执行过程：

{ A = 1 b = 2}

| CPU 1 | CPU 2 |
| ----- | ----- |
| A=3;  | x=B;  |
| B=4;  | y=A;  |

这有 24 中内存访问操作的组合，每种组合都有可能出现：

```
STORE A=3,      STORE B=4,      y=LOAD A->3,    x=LOAD B->4
STORE A=3,      STORE B=4,      x=LOAD B->4,    y=LOAD A->3
STORE A=3,      y=LOAD A->3,    STORE B=4,      x=LOAD B->4
STORE A=3,      y=LOAD A->3,    x=LOAD B->2,    STORE B=4
STORE A=3,      x=LOAD B->2,    STORE B=4,      y=LOAD A->3
STORE A=3,      x=LOAD B->2,    y=LOAD A->3,    STORE B=4
STORE B=4,      STORE A=3,      y=LOAD A->3,    x=LOAD B->4
STORE B=4, ...
...
```

从而产生 4 种结果：

```
x == 2, y == 1
x == 2, y == 3
x == 4, y == 1
x == 4, y == 3
```

更残酷的是，一个 CPU 已经提交的 store 操作，另一个 CPU 可能不会感知到，从而 load 操作取到旧的值。

比如：

{A = 1, B = 2, C = 3, P = &A, Q = &C}

| CPU 1 | CPU 2 |
| ----- | ----- |
| B=4;  | Q=p;  |
| P=&B; | D=*Q; |

可以产生 4 种结果：

```
(Q == &A) and (D == 1)
(Q == &B) and (D == 2)
(Q == &B) and (D == 4)
```

##### 1.2 设备操作

一些设备将自己的控制接口映射成一个内存地址，访问这些地址的指令顺序是极重要的。比如一个拥有一系列内部寄存器的网卡，可以通过一个地址寄存器 (A) 和一个数据寄存器 (D) 访问它们。如果要访问内部寄存器 5 ，则使用下面的代码：

```
*A = 5;
x = *D;
```

但这个代码可能生成以下两种执行顺序：

```
STORE *A = 5, x = LOAD *D
x = LOAD *D, STORE *A = 5
```

##### 1.3 合并内存访问

CPU 还可能将内存操作合并。比如

```
X = *A; Y = *(A + 4);
```

可能会生成下面任何一种执行顺序：

```
X = LOAD *A; Y = LOAD *(A + 4);
Y = LOAD *(A + 4); X = LOAD *A;
{X, Y} = LOAD {*A, *(A + 4) };
```

而

```
*A = X; *(A + 4) = Y;
```

则可能生成下面任何一种执行：

```
STORE *A = X; STORE *(A + 4) = Y;
STORE *(A + 4) = Y; STORE *A = X;
STORE {*A, *(A + 4) } = {X, Y};
```

##### 1.4 最小保证

可以期望 CPU 提供了一些最小保证，不满足最小保证的 CPU 都是假的 CPU。

1. 有依赖关系的内存访问操作是有顺序的。也就是说：

```
Q = READ_ONCE(P); smp_read_barrier_depends(); D = READ_ONCE(*Q);
```

总是在 CPU 中以这样的顺序执行：

```
Q = LOAD P, D = LOAD *Q
```

smp_read_barrier_depends() 只在 DEC Alpha 中有用，READ_ONCE 的作用在 [这里](https://quant67.com/post/linux/access_once.html) 提到。

1. 在一个 CPU 中的覆盖 load-store 操作是有顺序的。比如

```
a = READ_ONCE(*X); WRITE_ONCE(*X, b);
```

总是以这样的顺序执行：

```
a = LOAD *X, STORE *X = b
```

而

```
WRITE_ONCE(*X, c); d = READ_ONCE(*X);
```

总是以下面的顺序执行：

```
STORE *X = c, d = LOAD *X
```

最小保证不适用于位图。假设我们有一个长度为 8 的位图，CPU 1 要将 1 位置 1， CPU 2 要将 2 位 置 1：

{ A = 0 }

| CPU 1             | CPU 2             |
| ----------------- | ----------------- |
| A = A OR (1 << 1) | A = A OR (1 << 2) |

可能有三种个结果：

```
A = 2
A = 4
A = 6
```

##### 1.5 示例

```
int main(){
	int a;
	a=0;
	a=1;
	a=2;
	printf("%d\n",a);
}
```

```
riscv64-unknown-elf-gcc week.c -S -O2 week
```

```
main:
	lui	a0,%hi(.LC0)
	addi	sp,sp,-16
	li	a1,2
	addi	a0,a0,%lo(.LC0)
	sd	ra,8(sp)
	call	printf
	ld	ra,8(sp)
	li	a0,0
	addi	sp,sp,16
	jr	ra
	.size	main, .-main
	.section	.rodata.str1.8,"aMS",@progbits,1
	.align	3
```

​	可以看到，在O2级别的优化下，变量a直接被赋值了2。

​	在编译器对代码的编译过程中会造成不可想象的指令重排。解决指令重排的方法就是使用barrier。

#### barrier

Linux 内核拥有三种级别的屏障：

- 编译器屏障
- CPU 内存屏障
- MMIO 写屏障

##### READ_ONCE与WRITE_ONCE

```
// linux-5.4.18/include/linux/compiler.h
#define READ_ONCE(x) __READ_ONCE(x, 1)

#define __READ_ONCE(x, check)						\
({									\
//	定义一个联合体，其大小就是x的大小。
// __c代表首地址
	union { typeof(x) __val; char __c[1]; } __u;			\
	if (check)							\
	//将x的地址传入并且传入第一个字节
		__read_once_size(&(x), __u.__c, sizeof(x));		\
	else								\
		__read_once_size_nocheck(&(x), __u.__c, sizeof(x));	\
	smp_read_barrier_depends(); /* Enforce dependency ordering from x */ \
	__u.__val;							\
})

static __always_inline
void __read_once_size(const volatile void *p, void *res, int size)
{
	__READ_ONCE_SIZE;
}

#define __READ_ONCE_SIZE						\
({									\
	switch (size) {							\
	case 1: *(__u8 *)res = *(volatile __u8 *)p; break;		\
	case 2: *(__u16 *)res = *(volatile __u16 *)p; break;		\
	case 4: *(__u32 *)res = *(volatile __u32 *)p; break;		\
	case 8: *(__u64 *)res = *(volatile __u64 *)p; break;		\
	default:							\
		barrier();						\
		__builtin_memcpy((void *)res, (const void *)p, size);	\
		barrier();						\
	}								\
})
```

​	为什么要使用联合体绕一圈？参考如下讨论链接。

https://stackoverflow.com/questions/54177247/why-this-union-has-char-array-at-the-end/54178113#54178113

https://github.com/torvalds/linux/commit/dd36929720f40f17685e841ae0d4c581c165ea60

​	意思是，如果type中含有const的时候，会转换出错，所以使用union技巧转换，当然可以强制转换。但是不容易移植。

​	READ_ONCE与WRITE_ONCE采用volatile关键字，如何是结构体等复杂结构，采用了barrier函数来保证访问顺序的一执行。

​	**READ_ONCE() 和 WRITE_ONCE() 函数为多个CPU对单个变量的访问提供了缓存一致性，防止了编译器和CPU对变量访问的重新排序，确保了访问的正确性和一致性。**

​	为什么？上述我们知道了这两个函数是通过volatile关键字，阻止编译器对指令进行重排，进而确保了访问的正确性和一致性。

​	volatile是多CPU之间修改变量的及时性，内存屏障是多CPU之间对变量读写的顺序性。两者不是一回事。volatile会在函数的开头把一个变量加载到寄存器，然后一直操作这个寄存器，函数末尾把寄存器的值写会内存(实际上是cache，但是不影响逻辑)，如果这个函数中间执行很长或者中间被切走了，那么变量修改的值就不能及时传递到其他CPU，这样其他CPU就不会看到此次变量的修改，所以加个volatile，让它每次都读写内存，这样其他CPU就能及时看到变量的修改。

​	volatile会写到内存，有cache就写到cache，逻辑是一样的，因为CPU之间寄存器不是共享的，每个CPU都有自己的寄存器，默认是先改寄存器，后面某个时刻再写到内存，这样会影响变量修改传播的及时性。没有cache的时候，它的语义是写到内存，因为所有CPU内存是共享的，其他CPU就能及时看到修改，有了cache就是写到cache，因为有cache一致性协议，所以语义是不变的。

​	内存屏障对变量修改传播的及时性没有作用，它是让多CPU之间看到的执行顺序有一致性。

​	在没有cache的年代，volatile是让读写变量都直接从内存读写，不用寄存器缓存，这样做的目的是为了及时在多CPU之间传播更改，因为内存是多个CPU共享的，而寄存器是各个CPU私有的。后来有了cache，如果把cache看成透明的话，不影响理解，如果考虑cache的参与，cache在多CPU之间有一致性协议，可以及时传播更新，所以符合volatile的语义。有些人不知道volatile背后的语义，机械的理解volatile，以为volatile写到内存是为了绕过cache。

​	可以这样理解，我们并不能感知到cache的存在，他对于我们是透明的，存在一个MESI协议来协调cache对内存的一致性。https://xiaolincoding.com/os/1_hardware/cpu_mesi.html

​	**记住volatile的本质是及时在多CPU之间传播更新，就不会有误解，内存是多CPU共享的，cache有一致性协议，都可以实现及时传播更改的目的。**

​	volatile还有一个常用的用途就是驱动中修饰硬件外设寄存器，硬件设备寄存器这个是全局的，所有的CPU都是共享的，volatile让编译器不用寄存器缓存，这是属于CPU和设备之间的及时传播更新。


原文链接：https://blog.csdn.net/weixin_45030965/article/details/132852641

https://stackoverflow.com/questions/50589499/write-once-and-read-once-in-linux-kernel

https://maple-leaf-0219.github.io/2020/linux%E5%86%85%E6%A0%B8%E4%B8%AD%E7%9A%84%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C-%E8%AF%91/

https://www.kernel.org/doc/Documentation/memory-barriers.txt

#### 3.1 编译器屏障

Linux 内核提供了编译器屏障函数，它防止编译器将屏障以便的内存操作移动到另一边：

```
barrier();
```

这是个一般屏障，READ_ONCE() 和 WRITE_ONCE() 可以认为是 barier() 的弱化版本。

barrier() 有一下两个功能：

​	(1) 防止编译器将 barrier() 之后的内存访问重排到 barrier() 之前。 

​	(2) 在循环中，迫使编译器在循环里面每次都取条件判断中需要的值，而不是只取一次。



​	通过使用 barrier() 函数，可以在编写并发代码时控制内存访问的顺序和正确性。它是一种同步原语，用于确保数据的一致性和可见性，特别是在多线程或中断处理的上下文中。

​	需要注意的是，barrier() 函数只提供编译器层面的屏障效果，而不提供硬件层面的原子性或同步保证。在需要更强的同步保证时，可能需要使用其他同步原语，如原子操作或锁定。


#### 3.2 CPU 内存屏障

Linux 内核拥有 8 个基本的内存屏障：

| TYPE            | MANDATORY              | SMP CONDITIONAL            |
| --------------- | ---------------------- | -------------------------- |
| GENERAL         | mb()                   | smp_mb()                   |
| WRITE           | wmb()                  | smp_wmb()                  |
| READ            | rmb()                  | smp_rmb()                  |
| DATA DEPENDENCY | read_barrier_depends() | smp_read_barrier_depends() |

除了数据依赖屏障，其它每个内存屏障都隐含这编译器屏障。数据依赖屏障不影响编译器的生成的指令顺序。

SMP 内存屏障在单处理器编译系统上的作用跟编译器屏障一样，因为我们假设单 CPU 是不会把事情搞砸的。

在 SMP 系统中，必须使用 SMP 内存屏障来规范共享内存的访问次序，即使使用锁已经足够了。

#### 3.3 MMIO 写屏障

Linux 内核有一个专门用于 MMIO 写的屏障：

```
mmiowb()
```

https://zhuanlan.zhihu.com/p/102753962

https://aijishu.com/a/1060000000350623

https://quant67.com/post/linux/memory-barriers/memory-barriers.html

http://www.wowotech.net/kernel_synchronization/453.html

### atomic_t

```
typedef struct {
	int counter;
} atomic_t;
```

​	不可能将所有的数据都作为原子操作的数据，如果这样做，系统就没有并行性可言，CPU只可能同时有一个CPU在工作。这不符合SMP的设计。

​	所以如上讲某些数据设为是原子操作可用的数据，其余数据都可以自由访问。只有这个原子数据是访问受限的。那么将这个数据作为flag，然后控制指令访问其余数据。

​	为了针对read-modify-write的变量的原子操作，所以上述包装了一个int作为atomic变量。然后针对这个变量进行原子操作，其余变量无所谓。

​	下述是atomic的接口。每个arch都有其实现。

| 函数                                                         | 含义                                                         | RMW     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------- |
| static inline void atomic_add(int i, atomic_t *v)            | 给一个原子变量v增加i                                         | RMW     |
| static inline int atomic_add_return(int i, atomic_t *v)      | 同上，只不过将变量v的最新值返回                              | RMW     |
| static inline void atomic_sub(int i, atomic_t *v)            | 给一个原子变量v减去i                                         | RMW     |
| static inline int atomic_sub_return(int i, atomic_t *v)      | 同上，只不过将变量v的最新值返回                              | RMW     |
| static inline int atomic_cmpxchg(atomic_t *ptr, int old, int new) | 比较old和原子变量ptr中的值，如果相等，那么就把new值赋给原子变量。 返回旧的原子变量ptr中的值 | RMW     |
| atomic_read                                                  | 获取原子变量的值                                             | Non-RMW |
| atomic_set                                                   | 设定原子变量的值                                             | Non-RMW |
| atomic_inc(v)                                                | 原子变量的值加一                                             | RMW     |
| atomic_inc_return(v)                                         | 同上，只不过将变量v的最新值返回                              | RMW     |
| atomic_dec(v)                                                | 原子变量的值减去一                                           | RMW     |
| atomic_dec_return(v)                                         | 同上，只不过将变量v的最新值返回                              | RMW     |
| atomic_sub_and_test(i, v)                                    | 给一个原子变量v减去i，并判断变量v的最新值是否等于0           | RMW     |
| atomic_add_negative(i,v)                                     | 给一个原子变量v增加i，并判断变量v的最新值是否是负数          | RMW     |
| static inline int atomic_add_unless(atomic_t *v, int a, int u) | 只要原子变量v不等于u，那么就执行原子变量v加a的操作。 如果v不等于u，返回非0值，否则返回0值 | RMW     |

**Non-RMW ops**

```
atomic_read()
atomic_set()
atomic_read_acquire()
atomic_set_release()
```

**RMW atomic operations**
Arithmetic 算术：

```
atomic_{add,sub,inc,dec}()
atomic_{add,sub,inc,dec}_return{,_relaxed,_acquire,_release}()
atomic_fetch_{add,sub,inc,dec}{,_relaxed,_acquire,_release}()
```


Bitwise 位运算：

```
atomic_{and,or,xor,andnot}()
atomic_fetch_{and,or,xor,andnot}{,_relaxed,_acquire,_release}()
```

Swap 交换：

```
atomic_xchg{,_relaxed,_acquire,_release}()
atomic_cmpxchg{,_relaxed,_acquire,_release}()
atomic_try_cmpxchg{,_relaxed,_acquire,_release}()
```

对于Non-RMW操作来说

​	这个用于`atomic-fallback.h`中自动生成的函数。对于读写这种`non-RMW`操作，如你所见就是这么简单：

```
static __always_inline int atomic_read(const atomic_t *v)
{
	return READ_ONCE(v->counter);
}
static __always_inline void atomic_set(atomic_t *v, int i)
{
	WRITE_ONCE(v->counter, i);
}
```

​	还是那句话，硬件保证原子性，内核只要保证生成的指令不走样就行了，这也是使用`READ_ONCE`和`WRITE_ONCE`的原因。接下来就是重头戏：对于`RMW`操作实现。RISC-V中定义了原子操作指令，即被称为`A`的扩展，Linux内核默认要求其已被实现。内核中通过內联汇编模板的方式实现这些操作，也算是比较简洁的了。

对于`RMW`类的原子操作，我们主要关注其：

- 功能正确性
- 内存序正确性

对于RISCV架构

https://crab2313.github.io/post/riscv-atomic-barrier-bitops/

对于ARM架构

http://www.wowotech.net/kernel_synchronization/atomic.html

​	由于对这部分指令不是很了解，先略过。







https://tinylab.org/riscv-atomics/

https://blog.mygraphql.com/zh/notes/low-tec/kernel/5-sync/synchronizeation-primitives/

https://blog.csdn.net/u012849539/article/details/106876434

https://crab2313.github.io/post/riscv-atomic-barrier-bitops/