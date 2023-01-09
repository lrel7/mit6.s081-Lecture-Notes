##### trap：用户空间和内核空间的切换

- 寄存器
  - 32个**用户寄存器**（用户程序可以使用）
  - 硬件中的寄存器：**PC（Program Counter）**，**MODE**（表明当前mode是supervisor还是user），**SATP**（包含了指向页表的物理内存地址），**STVEC**（指向内核中处理trap指令的起始地址），**SEPC**（在trap的过程中保存PC的值），**SSRATCH**


trap的开始，CPU的所用状态都设置成*运行用户代码*而非内核代码；trap实际就是要更改一些这里的状态，这样我们才能运行内核中的一些程序

需要做的操作：

- 保存32个用户寄存器：需要恢复用户应用程序的执行，因此这32个用户寄存器不能被内核弄乱；但同时内核代码又要使用这些寄存器
- 保存程序计数器：需要在用户程序运行中断的**位置**继续执行 *——ecall*
- 将mode改成**supervisor mode** *——ecall*
- 将SATP指向kernel page table：当前它正指向用户程序的页表，并没有包含整个内核数据的内存映射
- 将**堆栈寄存器（stack pointer）**指向内核中的一个地址：需要一个堆栈来调用内核的函数
- 跳入内核的C代码



trap的执行流程











用户代码调用系统调用

- 用户代码调用系统调用，实际上是调用一个由**汇编语言**写的函数（在usys.S中）
  - 这个函数用**li**将该系统调用的序号（在syscall.h中定义）载入a7寄存器
  - 然后执行**ecall**——代码执行跳转到内核
  - 内核完成工作后，代码执行返回用户空间，继续执行ecall之后的指令，也就是**ret**，返回用户代码


*当前的页表仍然是用户程序的页表，而非整个内核的；PC指向用户程序的地址（较小）*

- **执行ecall**
  - ecall会将mode从user改到supervisor
  - 将PC的值保存在**SEPC寄存器**（下一步就要跳转了）
  - 跳转到**STVEC寄存器**指向的地址——trampoline page，内核在这里执行trap开始的一些指令


*页表没变（仍然是用户程序的页表，因此每个用户程序的页表都含有trampoline page），但PC指向用户内存中的trampoline page（较大）*



##### TRAMPOLINE——uservec函数

- 保存用户寄存器
  - 执行**csrrw指令**，交换**a0寄存器**（存放该系统调用的第一个参数）和**SSCRATCH寄存器**（存放trapframe page的地址）的内容
    保存了a0寄存器的值，同时获得了指向trapframe的指针（a0现在的内容是trapframe的地址）

  - 执行**sd指令**，将每个寄存器保存在trapframe的不同偏移位置（对应的位置已经在trapframe结构体中定义好了）

- 将**stack pointer（sp寄存器）**指向这个进程的kernel stack的最顶端
  - 执行**ld指令**，将**a0**（此时指向trapframe）中储存的kernel stack顶端的地址（**kernel\_sp**）载入**sp**

- 跳入内核的C代码（只是载入地址，在uservec函数结束时才跳入）
  - 将要执行的第一个C函数是**usertrap()**
  - 执行**ld指令**，将**a0**中储存的usertrap()的地址（**kernel\_trap**）载入**t0**

- 将**SATP寄存器**指向**kernel page table**
  - 将**a0**中储存的kernel page table的地址（**kernel\_satp**）载入**t1**
  - 执行**csrw指令**，交换**SATP寄存器**和**t1寄存器**的值


*至此，当前程序从user page table切换到kernel page table*

**注意：我们还在trampoline.S的代码中；trampoline page在user page table和kernel page table中的映射完全一样，所以切换页表后，寻址的结果不会改变，我们可以继续在trampoline代码中执行程序而不崩溃（重申：程序中使用的是虚拟地址，需要寻址到物理地址）**

trampoline意思是**蹦床**，我们在它上面**弹跳**了一下，从用户空间走到了内核空间

- 最后执行**jr t0指令**，从trampoline跳入内核的C代码中（t0储存着usertrap()函数的地址）



##### usertrap函数

- 更改**STVEC寄存器**（此时它指向用户空间中trap处理代码的位置），让其指向内核空间中trap处理代码的位置（**kernelvec**）
- 调用**myproc函数**获得当前进程
  *myproc根据CPU核的编号查找当前进程，而CPU核的编号已经在uservec函数中被放入tp寄存器了*


- 保存**用户程序计数器**
  *PC仍然保存在SEPC寄存器中（ecall过程），但程序还在内核中执行时，可能切换到另一个进程，并进入它的用户空间，该进程可能再调用一个系统调用，进而导致SEPC寄存器的内容被覆盖*

  *需要保存当前的SEPC寄存器到一个****与该进程关联****的内存中（trapframe），防止覆盖*


- 找出触发trap的原因
  *根据触发trap的原因，**SCAUSE寄存器**会有不同数字，数字8表示”是因为系统调用“*


- 调用**syscall函数**，它根据系统调用编号**调用函数**
  *系统调用编号被保存在****a7寄存器****中（ecall过程），而用户寄存器现在都保存在****trapframe****，所以syscall会到trapframe里的a7寄存器取值，syscall.c中定义了一个函数指针数组，syscall根据这个编号索引数组，调用相应的函数*

  *syscall会将函数返回值存放在a0寄存器（同样也是由trapframe访问）*


- 调用**usertrapret函数**



##### usertrapret函数（usertrap-ret）

- 设置**STVEC寄存器**指向trampoline代码，那里会执行sret指令返回用户空间
- 设置trapframe内容，方便**下一次切换**时使用
  - kernel page table
  - 当前用户进程的kernel stack
  - usertrap函数的指针（方便trampoline跳转到这里）
  - 当前的CPU核编号

- 设置**SSTATUS寄存器**，它的**SPP bit**控制sret指令的行为，为0时表示下次执行sret时**返回user mode**

将PC设成**SEPC寄存器**储存的值（要继续系统调用前的下一条指令）

- ~~设置~~**SATP寄存器**~~，让它指向user page table~~（之后在trampoline中完成页表的切换）
- 跳到**trampoline**的**userret函数**



*程序执行回到trampoline代码（汇编语言）*

userret函数

- 用**csrw指令**将user page table（在usertrapret中作为第二个参数传递给userret，在a1寄存器）存储在**SATP寄存器**

*至此，程序回到user page table*

- 恢复已保存的所有**用户寄存器**
  *通过当前的****a0寄存器****（内核使用的寄存器，存放着usertrapret传递过来的第一个参数，即trapframe的地址），找到trapframe中保存的****a0寄存器****（这是在uservec函数中保存的****用户寄存器****，btw它现在储存着系统调用的返回值，见syscall函数），这个****a0****加上各种索引就能找到同样保存在trapframe中的其他****用户寄存器***

  *最后恢复**a0寄存器**（交换**SSCRATCH寄存器**和**a0**寄存器的值，即将uservec函数所做的交换恢复）*


- **sret**指令
  - 程序切换回**user mode**
  - **SEPC寄存器**的值被拷贝回**pc寄存器**
  - 重新打开**中断**




*回到了用户空间*



总结：系统调用形似函数，但背后的**user/kernel**转换比函数调用复杂得多，其目的就是保持user/kernel之间的**隔离性**







  

    













