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
2. 制作根文件系统

　制作根文件系统费了一番波折，无论是使用dracut还是busybox都遇到各种错误，最后无奈之下只好用了github上的一个[menuos](https://github.com/mengning/menu)。
 
3. 执行如下命令
```
qemu-system-x86_64 -S -kernel arch/x86/boot/bzImage -initrd ~/rootfs.img  -append "nokaslr"
```
* -S参数是让虚拟CPU通电后立刻停止，便于调试。

* -append "nokaslr"是为了让内核解压到一个固定的内存地址，否则为了保护内核安全，内核会被解压到一个随机的地址，难以调试。

4. 下载并安装GDB，我是在学校的[镜像站](https://mirrors.ustc.edu.cn/gnu/gdb/)上下载的源码包，然后编译安装的，具体过程不再叙述。 


以上是环境的搭建和工具的配置，下面是具体的调试过程。

#### 调试过程

1. 在qemu中按下Ctrl+Alt+2，输入gdbserver tcp::1234，监听来自gdb的连接，然后再按Ctrl+Alt+1,切回启动界面。

2. 在终端启动GDB，执行如下命令，以一个比较友好的界面调试启动映像
```
gdb vmlinux -tui
```
　　再输入target remote:1234，与qemu建立连接。

3. 下面根据进入文件或函数执行的顺序进行说明(只列举了一部分函数)

- arch/x86/kernel/head_64.S

  - 在GDB调试界面输入list，可以发现，当前指令所在的文件是head_64.S，其中一个重要函数startup_64()。（仍然是汇编代码，难以看懂）
  - 查阅相关资料可知，startup_64()是内核解压后执行的第一个函数。那么为什么我们调试的时候没有解压内核那些步骤呢？我个人猜想，是因为用于调试的vmlinux是未压缩的内核，那么自然也就没有解压的操作。
  
  
- init/main.c  start_kernel()

  - start_kernel函数是linux内核中汇编语言和C语言的分界处，是内核初始化的入口函数。
  - 查阅[相关资料](https://blog.csdn.net/RichardYSteven/article/details/52629731)可知，启动程序是经过多次跳转才进入start_kernel的，简要概括如下
> bootloader->header.S->head_64.S->startup_64->start_kernel，
> 其中还包括从实模式到保护模式的跳转，从32位到64位的跳转等等。
  - start_kernel调用的第一个函数set_task_stack_end_magic(&init_task)实际上是创建了第一个进程init_task，被称为0号进程，也是唯一一个没有通过fork()或者kernel_thread()创建的进程。
  
  ```
  set_task_stack_end_magic(&init_task)
  ```
  ![before start_kernel](https://github.com/OSH-2018/OS_Li/blob/master/lab01/before%20start_kernel.png)
  
  从这张图也可以看出，在执行start_kernel之前，是有解压内核这些操作的。   
  
  -   一系列初始化函数，但是这些函数执行完后，屏幕上不会有任何信息（函数输出的信息全部写入缓冲区中），因为此时控制台还不可用，直到函数console_init()的执行，之前写入缓冲区的信息才会显示出来。 
   ```
   boot_cpu_init();
   page_address_init();
   setup_arch(&command_line);
   ...
   console_init();
   ```
   ![before console_init](https://github.com/OSH-2018/OS_Li/blob/master/lab01/console_init.png)
   
   -   在调用console_init()之后仍然是一系列初始化函数，最后一个函数调用rest_init()很重要。
   
   
- init/main.c  rest_init()
  - 调用kernel_thread函数创建1号进程kernel_init
  ```
  pid = kernel_thread(kernel_init, NULL, CLONE_FS);
  ```
  ![pid=1](https://github.com/OSH-2018/OS_Li/blob/master/lab01/%E5%86%85%E6%A0%B81%E5%8F%B7%E8%BF%9B%E7%A8%8B.png)
  
  -   调用kernel_thread函数创建2号进程kthreadd
  ```
  pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
  ```
  ![pid=2](https://github.com/OSH-2018/OS_Li/blob/master/lab01/kthread%E8%BF%9B%E7%A8%8B.png)
  
- init/main.c  kernel_init()
  - kernel_init()是init进程运行的函数，它调用各种函数完成设备驱动程序的初始化
  ![kernel_init](https://github.com/OSH-2018/OS_Li/blob/master/lab01/kernel_init's%20work.png)

### 总结

- linux内核在启动过程中最关键的事件应该是0号、１号和２号进程的创建。
  - 0号进程即IDLE进程，由操作系统直接创建，当内核加载完成后，运行队列中没有进程时，该进程会进入运行
  - 1号进程即init进程，由0号进程通过kernel_thread函数创建，并完成各种设备驱动程序的初始化，还是所有用户进程的祖先
  - 2号进程即kthreadd进程，同样由0号进程通过kernel_thread函数创建，负责管理内核线程  
- 不足之处
  - 没有跟踪内核启动到startup_64这段过程
- 遇到的问题
  - GDB在设置完断点，输入c进行调试时，会出现如下错误：
  
> Remote 'g' packet reply is too long
  
  在网络上查到两种解决办法，一种是修改源代码，但还需要重新编译，比较繁琐，另一种是在发生这个错误后输入如下命令：
  ```
  (gdb)disconnect
  (gdb)set arch i386:x86-64:intel
  (gdb)target remote:1234
  ```
