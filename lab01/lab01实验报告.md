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
2. 执行如下命令
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
  ```
  set_task_stack_end_magic(&init_task)
  ```
  ![before start_kernel](https://github.com/OSH-2018/OS_Li/blob/master/lab01/before%20start_kernel.png)
  - 一系列初始化函数，但是这些函数执行完后，屏幕上不会有任何信息（函数输出的信息全部写入缓冲区中），因为此时控制台还不可用，直到函数console_init()的执行，之前写入缓冲区的信息才会显示出来。 
   ```
   boot_cpu_init();
   page_address_init();
   setup_arch(&command_line);
   ...
   console_init();
   ```
   ![before console_init](https://github.com/OSH-2018/OS_Li/blob/master/lab01/console_init.png)
   
  - 在调用console_init()之后仍然是一系列初始化函数，最后一个函数调用rest_init()很重要。
- init/main.c  rest_init()
  - 调用kernel_thread函数创建1号进程kernel_init
  ```
  pid = kernel_thread(kernel_init, NULL, CLONE_FS);
  ```
  ![pid=1](https://github.com/OSH-2018/OS_Li/blob/master/lab01/%E5%86%85%E6%A0%B81%E5%8F%B7%E8%BF%9B%E7%A8%8B.png)
  - 调用kernel_thread函数创建2号进程kthreadd
  ```
  pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
  ```
  ![pid=2](https://github.com/OSH-2018/OS_Li/blob/master/lab01/kthread%E8%BF%9B%E7%A8%8B.png)

### 总结

- linux内核在启动过程中最关键的事件应该是0号、１号和２号进程的创建。
  - 0号进程即IDLE进程，由操作系统直接创建
  - 1号进程即init进程，由0号进程通过kernel_thread函数创建，是所有用户态进程的祖先
  - 2号进程即kthreadd进程，同样由0号进程通过kernel_thread函数创建，负责管理内核线程
  
- 不足之处
  - 没有跟踪内核启动到startup_64这段过程
