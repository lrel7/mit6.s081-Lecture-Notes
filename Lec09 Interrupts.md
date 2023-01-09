*硬件想得到操作系统的关注*

  *e.g. 用户通过键盘按下一个按键——键盘产生一个终端*


操作系统需要：**保存**当前的工作，**处理中断**，处理完成后**恢复**之前的工作（仍然是**trap机制**，类似系统调用，page fault）



*中断 vs 系统调用*

1. __asynchronous__ 硬件产生中断时，interrupt handler与当前运行的进程在CPU上__没有任何关联__；但系统调用发生在运行进程的context下
2. __concurrency__ CPU和生成中断的设备 __并行运行__
3. __program device__ 设备的编程

---

外设中断来自主板上的设备（主板可以连接以太网卡，MicroUSB，MicroSD等），它们与CPU连接在一起

处理器上的**Platform Level Interrupt Control (PLIC)**将中断**路由**到某个CPU核；如果所有CPU核都在处理中断，PLIC会保留中断直到有一个CPU核可以用来处理中断，具体流程：

- PLIC通知当前有一个待处理的中断
- 其中一个CPU核会**claim**接受中断，这样PLIC就不会把中断发给其他的CPU处理
- CPU核处理完中断以后，会通知PLIC
- PLIC不再保存中断的信息

---

**驱动** 管理设备的代码，在内核中

- bottom部分：Interrupt handler，CPU接受中断时会调用它；它并不运行在任何特定进程的context中
- top部分：用户进程或内核其他部分调用的接口
- 队列（buffer）：top部分和bottom部分的代码都可以从队列读写数据，它可以将并行运行的设备和

CPU解耦开


**memory mapped I/O**

设备地址出现在物理地址的特定区间（由主板制造商决定），操作系统通过普通的load / store指令对这些地址进行编程

---

*Shell程序的$是如何输出的？*

  设备将字符传输给UART的寄存器——UART发送完字符后产生一个中断——线路另一端有另一个UART芯片连接到console，它进一步将$显示在console上


*用户输入的字符如何显示在console上？*

  键盘连接到UART的输入线路；键盘上按下一个键——UART芯片将按键字符通过串口线发送到另一端的UART芯片——它将数据bit合并成一个byte，再产生一个中断，告诉处理器这里有一个来自于键盘的字符——Interrupt handler处理字符


---

*RISC-V中与中断相关的寄存器*

- **SIE (Supervisor Interrupt Enable)**
  bit(E)针对外部设备的中断，bit(S)针对软件中断（可能由一个CPU核触发给另一个CPU核）,bit(T)针对定时器中断


- **SSTATUS (Supervisor Status)**
  一个bit打开 / 关闭中断（控制所有的中断）


- **SIP (Supervisor Interrupt Pending)**
  处理器通过查看它得知当前 __中断类型__


- **SCAUSE**
  表明当前状态的原因是中断


- **STVEC**
  保存当trap，page fault，中断发生时，当前用户进程的程序计数器，便于稍后恢复程序



---

#### 打开中断

**start函数** 将所有中断都设置在Supervisor mode，然后设置SIE寄存器来接收三种中断（E, S, T），并初始化定时器

**consoleinit函数** 初始化锁，调用uartinit配置好UART芯片

**uartinit函数** 先关闭中断，设置波特率（串口线的传输速率），设置字符长度为8 bits，重置FIFO，最后重新打开中断

**plicinit函数** 使能了UART的中断

**plicinithart函数** 每个CPU核通过调用它表明自己对来自于UART和VIRTIO的中断感兴趣

**scheduler函数** 执行intr\_on (打开SSTATUS寄存器的中断标志位)使CPU能接收中断，然后运行进程

*至此，中断完全打开*

---

*"$ "字符的输出*

UART设备的top部分（用户进程或内核其他部分调用的接口）

**init.c中的main函数** 系统启动后运行的第一个进程，mknod操作创建Console（文件描述符为0，1，2）

**sh.c** 打开文件描述符0，1，2，然后向文件描述符2（stdout）打印提示符“$"

*Unix系统中，设备由文件表示，Shell程序并不知道文件描述符2对应的是什么*

fprintf函数——vprintf函数——putc函数——write系统调用——sys_write函数——filewrite函数

**filewrite函数** 判断fd的类型（mknod生成的文件描述符属于FD_DEVICE），对于设备类型，为特定的设备这姓设备相应的write函数（我们现在的设备是console，所以调用console.c中的consolewrite函数）

**consolewrite函数** 将字符拷入，调用uartputc函数（将字符写入UART设备）；是一个UART驱动的top部分（用户进程或内核其他部分调用的接口）

**uartputc函数** UART内部有一个buffer（用于发送数据），还有一个为consumer提供的 __读指针__和为producer（在我们的例子中，Shell是producer）提供的 __写指针__，构建一个环形的buffer

- 判断环形buffer是否已满：读写指针相同——buffer是空的；写指针加1等于读指针——buffer已满，将CPU让出给其他进程
- buffer不满，则将字符送到buffer，更新写指针，调用uartstart函数

**uartstart函数** 通知设备执行操作

- 检查设备是否空闲
- 空闲，则从buffer中读出数据，将数据写入**THR (Transmission Holding Register)寄存器**（相当于告诉UART设备，这里有一个byte需要你来发送）；当数据送入UART设备后，这个write系统调用会返回，应用程序Shell继续执行（这里从内核返回用户空间，机制与trap机制一样）
- 与此同时，UART设备向console发送这个数据；在某个时间点，**PLIC**会收到UART设备的**中断**



*至此，\$被打印到console；但实际上\$后面还有一个空格*



---

#### UART驱动的bottom部分（Interrupt handler）

**PLIC**将中断**路由**给一个特定的CPU核（它设置了**SIE**的E bit，表明针对外部中断），这个CPU核会：

- 清除SIE寄存器相应的bit，阻止CPU核被其他中断打扰，处理完成以后再恢复bit
- 保存当前的程序计数器：设置**SEPC寄存器**为当前的程序计数器
- 保存当前的mode（在我们的例子中，当前运行的是Shell程序，所以会记录user mode）
- 将mode设置为Supervisor mode
- 最后将程序计数器的值设置成**STVEC的值**（它用于保存trap处理程序的地址，在xv6中，它保存uservec或kernelvec的地址，具体取决于发生中断时程序是运行在用户空间还是内核空间，在我们的例子中，它保存的是uservec的值，uservec函数会调用usertrap函数，进入到trap过程）

**usertrap函数** 调用devintr函数

**devintr函数**

- 通过SCAUCE寄存器的值判断中断类型，如果是来自外设，则调用plic\_claim函数（返回中断号，UART的中断号是10）
- 如果是UART中断，则调用uartintr函数（当前UART的接收寄存器为空， 因为我们是向UART发送数据，还没有通过键盘输入任何数据；所以代码直接运行到uartstart函数）

**uartstart函数** 发现buffer中还有一个空格字符，将这个空格字符也送出到console



*top部分和bottom部分最终都会走向uartstart函数，它们被解耦开了*

---

#### 并发

- 设备与CPU并行运行
  e.g. UART向console发送字符时，CPU返回执行Shell，而Shell可能会再执行一次系统调用，向bufferz中写入另一个字符，这称为**producer-consumer并行**


- 中断会停止当前运行的程序
  代码运行在kernel mode也会被中断，说明内核代码并非直接串行运行；对于一些不能再执行期间被中断的代码来

  说，内核需要临时关闭中断


- 驱动的top和bottom并行地在不同的CPU核上运行
  e.g. Shell在传输完“$”后再传输空格字符，代码会走到UART驱动的top部分（uartputc），将空格写入buffer中；但同时另一个CPU核可能会收到来自UART的中断，进而执行UART驱动的bottom部分，查看相同的buffer





##### producer / consumer并发

驱动中有一个buffer（32个字符），它有两个指针：读指针和写指针

  **producer** (e.g. Shell调用uartputc函数时)将字符写入到**写指针**的位置，并将写指针加1

    写指针追上读指针（写指针 + 1 等于读指针）时，buffer满，代码会sleep，暂时搁置Shell并运行其他进程


  读指针追上写指针（两个指针相等）时，buffer空，此时不做任何操作

    读指针追上写指针（两个指针相等）时，buffer空，此时不做任何操作



buffer对所有CPU核共享（它存在于内存，只有一份），因此需要lock

*以上就是“$ ”字符输出的全部过程*

---

*如何在Shell中用键盘输入？*

UART读取键盘输入

Shell调用read（系统调用）——fileread函数——如果读取的文件类型是设备，则调用相应设备的read函数

在这个例子里，这个read函数就是consoleread函数

这里与UART类似，也有一个buffer (128字符），但Shell变成了consumer（Shell要从buffer中读取数据），而键盘是producer（将数据写入buffer）

- buffer为空时，进程会sleep
- 在某个时间点，用户通过键盘输入一个字符——字符被发送到UART芯片——产生中断——被PLIC路由到某个CPU核——触发devintr函数，它发现这是一个UART中断——通过uartgetc函数获取字符——将字符传给consoleintr函数——consputc函数将字符输出到console上
- 之后，字符被存放在buffer中，当遇到换行符时，唤醒之前sleep的Shell进程，它再从buffer中将数据读出，执行相应的指令

通过buffer将consumer和producer解耦，让他们并行运行

  *如果其中一个运行过快，那么buffer要么是满的，要么是空的，它会sleep，等待另一个追上来*


---

#### Interrupt的演进

Unix刚开发时，Interrupt处理很快；现在相对变慢了（需要很多步骤）

*如果一个设备高速地产生中断，处理器很难跟上*

**polling** 除了依赖Interrupt，CPU可以一直读取外设的控制寄存器，来检查是否有数据

  但是浪费了CPU cycles（在使用CPU不停地检查寄存器的内容，而没有运行任何程序）

  因此polling适用于快设备，不适用于慢设备




