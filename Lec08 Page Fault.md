*虚拟内存的优点：隔离性；提供了一层抽象*

*虚拟内存→物理内存的映射*

  *page table——静态（最开始设置好了）*

  *page fault——动态*


*触发page fault*

- 出错的虚拟地址：**trap机制**，将程序运行切换到内核，同时将出错地址存放在**STVAL寄存器**（supervisor trap value）
- 出错的原因：存放在**SCAUSE寄存器**（12-instruction page fault，13-load page fault，15-store page fault）
- 触发page fault的指令的地址：存放在**SEPC寄存器**（supervisor exception program counter），并同时保存在trapframe->epc中

---

#### lazy page allocation

- sbrk：使用户程序能扩大**heap**的上边界
  - 程序启动时，sbrk指向heap的底端（也是stack的顶端，p->sz）
  - 参数是申请的bytes数（负数表示缩小heap）
  - eager allocation：一旦调用sbrk，内核立刻分配内存


*应用程序倾向于申请多于自己所需要的内存*

- lazy allocation
  - sbrk只提升p->sz，内核**不会立刻**分配内存
  - 当应用程序使用到新申请的那部分内存，触发page fault

- page fault：虚拟地址＜p->sz，＞stack
  在page fault handler中，**分配一个内存page**，映射到user page table



---

#### zero fill on demand

*解释了riscv代码中的.text，.data是什么*

  *解释了riscv代码中的.text，.data是什么*


*BSS中有很多page，但所有内容都是0，过于浪费*

只分配一个物理page（全为0），将所有**虚拟地址空间中全0的page**都map到这一个物理page上

  PTE都是只读的，因为期望page的内容是全0，不允许修改


当应用程序尝试写BBS中的一个page时，触发page fault

- 申请一个新的物理page（全为0）
- 更新BSS中要被写的那个页：PTE设置为可读可写，指向新分配的物理page



类似lazy allocation，只有在需要时才申请内存，节省；减少了exec的工作

---

#### copy on write fork

*e.x. Shell通过fork创建子进程，fork将Shell进程的地址空间完整地拷贝给子进程；但子进程中exec会丢弃这个地址空间，取而代之该进程的地址空间。浪费*

fork时，直接共享parent的物理内存page，而非创建、分配并拷贝parent的内容到新的物理内存

注意：父进程和子进程的PTE标志位都应设成**只读**，确保进程间的隔离性

当父进程或子进程尝试执行store更新全局变量时，触发page fault（在向一个只读的PTE写数据）

- 拷贝相应的物理page。假设是子进程触发了page fault，那么将新分配的物理page映射到子进程。
- 此时新page只对子进程可见，旧page只对父进程可见，实现了隔离，因此PTE都可以改成**可读可写**
- 重新执行store指令

---

#### demand paging

*同理zero fill on demand，不过是对于text和data区域*

在虚拟地址空间中，为text和data分配地址段，但相应的PTE不对应任何物理内存page（valid=0）

触发page fault：应用程序从地址0开始运行，从0开始向上是text区域，但还没有真正地加载内存

处理page fault：

- 这些page是on-demand page，在某个地方记录这些page__对应的程序文件__
- 在page fault handler中，从程序文件里读取page数据，加载到内存，映射
- 重新执行指令

OOM：

- 撤回page（将部分内存page中的内容写回文件系统，释放page，得到一个新的空闲page）
- 策略：least recently used（LRU）；选择non-dirty page而非dirty page
  ![](image_1.c47ef07c.png)

  - access bit：定时地恢复为0，从而实现LRU
  - dirty bit：向一个page写入时，设置


---

#### memory mapped files

将完整或部分文件加载到内存中，从而通过load，store指令操控文件

**系统调用：mmap**

- 接收：va，len，protection，flags，打开文件的fd，offset （从文件描述符对应的文件的偏移量的位置开始，映射长度为lend 内容到虚拟地址va，同时加上一些保护，如只读或读写）
- 同样，实现方式是lazy而非eager
  - 先记录这个pte属于这个文件描述符
    相应信息在VMA（virtual memory area）结构体中保存，记录文件描述符，偏移量等，用于表示对应的va在哪

  - 当遇到一个 __位于VMA记录的地址范围内__ 的page fault时，内核从磁盘中读数据，并加载到内存中
  - unmap时，将dirty block写回文件


---

#### 总结

核心是**lazy**（目的：防止一下子申请过多内存；重复、浪费；实际不需要）

流程：只记录，先不分配内存——因为实际上未分配内存（访问了不存在的 / 无效的pte）触发page fault——分配内存





