##### 符号 and Symbol Table


​	An object file's symbol table holds information needed to locate and relocate a program's symbolic definitions and references. A symbol table index is a subscript into this array. Index 0 both designates the first entry in the table and serves as the undefined symbol index. The contents of the initial entry are specified later in this section.

​	ELF文件中的“符号表（symbol table）”包含的是程序中的符号信息 -- 这些符号代表的或许是**定义**（例如**定义全局变量时使用的变量名**，或者**定义函数时使用的函数名**），或许代表的是**引用**（**例如使用关键字extern声明的变量或函数时使用的符号名称**）。当代表的是定义时，在链接阶段链接器需要为它们重定位；当代表的是引用时，在链接阶段链接器需要在其他编译模块定位到该符号的定义。 

存在三种符号：

- 由模块 m 定义并能被其他模块引用的**全局符号**。全局链接器符号对应于非静态的 C 函数和全局变量。 **重点：非静态的C函数与全局变量称作全局符号**
- 由其他模块定义并被模块 m 引用的**全局符号**。这些符号称为外部符号，对应于在其他模块中定义的非静态 C 函数和全局变量。**重点：相较于该模块之外的全局符号成为外部符号，或者说存在extern引用的符号**
- 只被模块 m 定义和引用的**局部符号**。它们对应于带 static 属性的 C 函数和全局变量。这些符号在模块 m 中任何位置都可见，但是不能被其他模块引用。 **static修饰的全局变量和C函数称为局部符号。**

**局部变量是不是一个符号？**

**static修饰的局部变量是不是一个符号？**

```
static int add(int a,int b){
    int no_static=10;
    static int static_var=11;
    return a+b;
}
int sub(int a,int b){
    int no_static=10;
    static int static_var=11;
    return a-b;
}
static int res=0;
int t_res=0;
```

```
0000000000000000 T _sub 函数
0000000000000028 d _sub.static_var 定义在函数内部的静态变量
0000000000000050 S _t_res 全局变量
0000000000000000 t ltmp0
0000000000000028 d ltmp1
0000000000000050 s ltmp2
0000000000000030 s ltmp3
```

```
static int add(int a,int b){
    int no_static=10;
    static int static_var=11;
    return a+b;
}
int sub(int a,int b){
    int no_static=10;
    static int static_var=11;
    add(no_static, static_var);
    return a-b;
}
static int res=0;
int t_res=0;
使用readelf可以看到符号的bind情况
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS add.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000    46 FUNC    LOCAL  DEFAULT    1 add
     6: 0000000000000004     4 OBJECT  LOCAL  DEFAULT    6 static_var.1559
     7: 0000000000000000     4 OBJECT  LOCAL  DEFAULT    4 res
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     9: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
    10: 0000000000000000     4 OBJECT  LOCAL  DEFAULT    6 static_var.1553
    11: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
    12: 0000000000000000     0 SECTION LOCAL  DEFAULT    8 
    13: 000000000000002e    76 FUNC    GLOBAL DEFAULT    1 sub
    14: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    5 t_res
```

​	如上实验中，如果静态函数在本地没有引用，则不会加入符号表，static修饰的局部变量一样。

​	是不是符号表中应该从如下角度考虑，是否被其他地方的代码引用，如果是被外部程序引用则是全局符号，例如非静态全局变量，非静态函数。static修饰的只有本地可以使用，且本地已经使用的，如static修饰的函数和局部变量，因为static修饰的局部变量的运算指向的是一块地址也就是一个标签。当然这里面符号导出的时候会注明是本地的。

​	所以，.symtab 中的符号表不包含非静态的局部变量。这些符号在运行时在栈中被管理，链接器对此类符号不感兴趣。定义为带有 C static 属性的本地过程变量是不在栈中管理的。相反，编译器在 .data 或 .bss 中为每个定义分配空间，并在符号表中创建一个有唯一名字的本地链接器符号。

https://hansimov.gitbook.io/csapp/part2/ch07-linking/7.5-symbols-and-symbol-tables

https://blog.csdn.net/helowken2/article/details/113782851

### **链接进阶**

```
riscv64-unknown-elf-ld.exe -o xxx.o xxx.o
```

- **第一步 地址与空间分配**
  扫描所有的输入目标文件，获得它们的各个节的长度、属性、位置，并将输入目标文件中的符号表中所有的符号定义和符号引用收集起来，统一放到一个全局的符号表。这一步，链接器能够获得所有输入目标文件的节的长度，并将它们合并，计算出输出文件中各个节合并后的长度与位置，并建立映射关系。
- **第二步 符号解析与重定位**
  使用前一步中收集到的所有信息，读取输入文件中节的输数据、重定位信息，并且进行符号解析与重定位、调整代码、调整代码中的地址等。事实上，第二步是链接过程的核心，尤其是重定位。

**静态链接与动态链接**

静态链接：在程序运行前所有的库都进行了链接和加载

动态链接：外部的函数在第一次被调用时才会加载和链接，每次程序开始运行，它都会按照需要链接最新版本的库函数。另外，如果 多个程序使用了同一个动态链接库，库代码在内存中只会加载一次。

**静态链接优缺点：**

**缺点：**

如果这样的库很大，链接一个库到多个程序中会十分占用内存。

链接时库是绑定的，即使它们后来的更新修复了 bug，强制的静态链接的代码仍然会使用旧的、有 bug 的 版本。

**优点：**

依赖性较低

**动态链接优缺点：**

1. 减少了包体积

1. 减少了内存占用

1. 减少了静态链接耗时

但带来了以下缺点：

1. 更慢的启动耗时（将链接过程从编译时改为运行时）

1. 更多的脏内存（每个动态库都有自己的 DATA ）

#### 动态链接的基本实现

动态链接涉及运行时的链接以及多个文件的装载，必需要有操作系统的支持。因为动态链接的情况下，进程的虚拟地址空间的分布会比静态链接情况下更为复杂，还有一些存储管理、内存共享、进程线程等机制在动态链接下也会有一些微妙的变化。

目前，主流操作系统都支持动态链接。在Linux中，ELF动态链接文件被称为 **动态共享对象（DSO，Dynamic Shared Objects）**，一般以`.so`为后缀；在Windows中，动态链接文件被称为 **动态链接库（Dynamic Linking Library）**，一般以`.dll`为后缀。

在Linux中，常用的C语言库的运行库glibc，其动态链接形式的版本保留在 `/lib`目录下，文件名为 `libc.so`。整个系统只保留一份C语言动态链接文件`libc.so`，所有的C语言编写的、动态链接的程序都可以在运行时使用它。当程序被装载时，系统的**动态链接器**会将程序所需要的所有动态链接库装载到进程的地址空间，并将程序中所有未解析的符号绑定到相应的动态链接库中，并进行重定位。

#### 动态链接程序运行时地址空间分布

对于静态链接的可执行文件来说，整个进程只有一个文件要被映射，即可执行文件。而对于动态链接，除了可执行文件，还有它所依赖的共享目标文件。

关于共享目标文件在内存中的地址分配，主要有两种解决方案，分别是：

- **静态共享库（Static Shared Library）**（地址固定）
- **动态共享库（Dynamic Shared Libary）**（地址不固定）

**静态共享库**

静态共享库的做法是将程序的各个模块统一交给操作系统进行管理，操作系统在**某个特定的地址**划分出一些地址块，为那些已知的模块预留足够的空间。因为这个地址对于不同的应用程序来说，都是固定的，所以称之为**静态**。

但是静态共享库的目标地址会导致地址冲突、升级等问题。

**动态共享库**

采用动态共享库的方式，也称为**装载时重定位（Load Time Relocation）**。其基本思路是：**在链接时，对所有绝对地址的引用都不作重定位，而把这一步推迟到装载时再完成。一旦模块装载地址确定，即目标地址确定，那么系统就对程序中所有的绝对地址引用进行重定位。**

但是这种方式也存在一些问题。比如，动态链接模块被装载映射至虚拟空间后，指令部分是在多个进程间共享的，由于装载时重定位的方法需要修改指令，所以没有办法做到同一份指令被多个进程共享，因为指令被重定位后对于每个进程来说都是不同的。

#### linux动态共享库

**搜索路径：**

```
1. 首先在环境变量LD_LIBRARY_PATH所记录的路径中查找。
2. 然后从缓存文件/etc/ld.so.cache中查找。这个缓存文件由ldconfig命令读取配置文 件/etc/ld.so.conf之后生成，稍后详细解释。
3. 如果上述步骤都找不到，则到默认的系统路径中查找，先是/usr/lib然后是/lib。
```

ldconfig是一个动态链接库管理命令，其目的为了让动态链接库为系统所共享。

ldconfig的主要用途：默认搜寻/lilb和/usr/lib，以及配置文件/etc/ld.so.conf内所列的目录下的库文件。

**Linux动态库搜索工具**

> ldd不是一个可执行程序，只是一个shell脚本，如果程序执行时，依赖的某个库找不到，通过这个命令可以迅速定位问题所在。

```
bingsanlang@ubuntu:~$ ldd ./main
	linux-vdso.so.1 =>  (0x00007fffb5f17000) 显示需要的类库
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fec28aeb000) 显示实际提供的类库 
	/lib64/ld-linux-x86-64.so.2 (0x00007fec28eb5000) 显示提供类库的开始地址空间
```

**GCC静态链接方式**

```
gcc -c xxx.c xxx.c xxx.c xxx.c #首先进行预处理 编译 汇编 但不链接用-c
```

```
ar rs libname.a xxx.o xxx.o xxx.o xxx.o  #打包成一个静态库 参考下面ar工具使用
```

```
gcc main.c -L. -lxxx -Ixxx -o main
```

-L选项告诉编译器去哪里找需要的库文件，-L.表示在当前目录找。

-lxxx告诉编译器要链 接libxxx库。

-I选项告诉编译器去哪里找头文件。注意，即使库文件就在当前目录，编译器默认也不会去找的，所以-L.选项不能少。

**GCC动态链接方式**

```
$ gcc -c -fPIC xxx.c xxx.c xxx.c xxx.c
```

```
gcc main.c -g -L. -lstack -Istack -o main 
```

​	此时运行报错，因为找不到动态库，可以参照上述搜索路径添加。

https://www.cnblogs.com/thechosenone95/p/10605172.html

## 