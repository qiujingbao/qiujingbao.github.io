gef



- `legend`：颜色代码的文字解释
- `regs`：寄存器状态
- `stack``$sp`：寄存器指向的内存内容
- `code`：正在执行的代码
- `args`：如果在函数调用处停止，则打印调用参数
- `source`：如果使用源代码编译，这将显示相应的源代码行
- `threads`：所有线程
- `trace`：执行调用轨迹
- `extra`：如果检测到自动行为（易受攻击的格式字符串、堆漏洞等），它将显示在此窗格中
- `memory`：窥视任意内存位置

要隐藏某个部分，只需使用设置`context.layout`，并在部分名称前面加上`-`它或省略它。

```
gef config context.layout "-legend regs -stack code args source -threads -trace -extra -memory" 
```

```
gef config context.redirect "/dev/pts/8"
```

