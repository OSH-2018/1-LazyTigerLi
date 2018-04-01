# lab01实验报告

## 调试操作系统的启动过程

### 实验目的

使用跟踪调试工具，追踪操作系统的启动过程，并找出其中至少两个关键事件。

### 实验环境和相关工具

操作系统环境：deepin 15.5桌面版

待调试内核版本：linux-4.15.14

调试工具：GDB 8.1，qemu 2.8

GCC版本：6.3.0

### 具体过程

#### 编译linux内核

1. 从[官网](https://www.kernel.org/)上下载linux内核

2. 将源码包提取出来，并在其根目录下执行命令

```
make defconfig
make menuconfig
```

* 第一步是根据机器的架构使用默认的配置(我的是x86_64)

* 在第二步中，需要进行如下设置：
**取消Kernel hacking->Compile-time compiler options下的Reduce debugging information并打开Provide GDB scripts for kernel debugging**(否则之后将无法用GDB进行调试)

* 最后执行如下命令生成内核启动映像
```
make bzImage
```

#### 环境的搭建和工具的配置

1. 下载并安装qemu，我直接在终端使用如下命令安装
```
sudo apt-get install qemu
```
2. 由于只需要跟踪linux的启动过程，并不用管内核启动完成后的事情，所以并不需要制作虚拟磁盘和文件系统，直接执行如下命令
```
qemu-system-x86_64 -S -kernel arch/x86/boot/bzImage -m 1024 -append "nokaslr"
```
* -S参数是让虚拟CPU通电后立刻停止，便于调试。

* -append "nokaslr"是为了让内核解压到一个固定的内存地址，否则为了保护内核安全，内核会被解压到一个随机的地址，难以调试。、

3. 下载并安装GDB，我是在学校的[镜像站](https://mirrors.ustc.edu.cn/gnu/gdb/)上下载的源码包，然后编译安装的，具体过程不再叙述。 


以上是环境的搭建和工具的配置，下面是具体的调试过程。

#### 调试过程

1. 在qemu中按下Ctrl+Alt+2，输入gdbserver tcp::1234，监听来自gdb的连接，然后再按Ctrl+Alt+1,切回启动界面。

2. 在终端启动gdb，执行如下命令，以一个比较友好的界面调试启动映像
```
gdb vmlinux -tui
```
　　再输入target remote localhost:1234，与qemu建立连接。

3. 下面根据进入文件或函数执行的顺序进行说明(只列举了一部分函数)

- arch/x86/kernel/head_64.S

- init/main.c  start_kernel()
  
  - start_kernel调用的第一个函数set_task_stack_end_magic(&init_task)实际上是创建了第一个进程init_task，被称为0号进程，也是唯一一个没有通过fork()或者kernel_thread()创建的进程。
  
  - boot_cpu_init(),page_address_init(),setup_arch()等一系列初始化函数，但是这些函数执行完后，屏幕上不会有任何信息（函数输出的信息全部写入缓冲区中），因为此时控制台还不可用，直到函数console_init()的执行，之前写入缓冲区的信息才会显示出来。、
  
  - 在调用console_init()之后仍然是一系列初始化函数，最后一个函数调用rest_init()很重要，

- init/main.c  rest_init()

  - 

　


### 总结
