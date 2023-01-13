- **write ahead rule** 先记录在log——再写到文件系统的实际位置，保证了**原子性**
- **freeing rule** 直到log中所有写操作被更新到文件系统之前，都不能释放或重用log

*xv6的logging太慢：每个系统调用需要等待它包含的所有写磁盘操作结束才返回（**synchronized**）；每个block都被写了两次（第一次写入log，第二次写入实际位置）*

---

#### ext3 log format

**维护transaction信息**

- 序列号
- 一系列由该transaction修改的block编号
- 一系列系统调用

**log 结构**

- **super block** 包含log中第一个有效transaction的**起始位置**（log中的block编号）** & 序列号**
- 其他block储存transaction，每个包括
  - 一个**descriptor block** 包含实际block编号
  - **data block** 针对每一个block的更新数据
  - 当一个transaction完成并commit后，会有一个**commit block**


*恢复过程扫描log时区分这些block：descriptor block & commit block以一个32位的**魔法数字**（不太可能出现在data block中）*


*循环的数据结构，用到最后会从最开始位置重用*

---

#### ext3如何提升性能

- **异步的（asynchronous）**，系统调用只更新缓存在内存中的block，不等写磁盘操作
  - 应用程序很快从系统调用返回并继续运行，与此同时文件系统在后台**并行**地完成写磁盘操作（**I/O concurrency**）
  - 使大量的批量执行容易
  - 缺点：系统调用的返回**并不能表示数据已经存入磁盘**
    - **fsync** 接受一个fd作为参数，当所有数据确认写入磁盘后才返回

- **批量执行（batching）**，将多个系统调用打包成一个transaction
  - 任何时候只会有一个open transaction
  - 宣告开始一个新的transaction——接下来几秒内所有系统调用都属于这个transaction——commit所有
    - 在多个系统调用之间**分摊**了transaction的**固有损耗**（共用descriptor block, commit block；在硬盘中查找log的位置并等待磁碟旋转）
    - 容易触发**write absorption**（一堆系统调用反复更新一组相同的block，先快速在block cache中完成，再一次性写入磁盘）
    - **disk scheduling** 一次性大量写log——向磁盘的一系列[**连续位置**](onenote:..\计算机体系结构\I_O.one#I\O%20Devices%20--%20Disks&section-id={60716532-7738-4116-8B9F-B4F994722494}&page-id={C6344C2F-CF77-4C2C-BF26-D92D5FBB6D5A}&object-id={1F4C5CA3-33DB-4884-A294-646DB83FB002}&37&base-path=https://d.docs.live.net/1f65032c09a11ca3/Documents/yuebaitu%20的笔记本/4各学科/3CS/玥自学)写block

- **并发（concurrency）**
  - 允许**多个系统调用**同时执行
  - 允许**多个不同状态的transaction**同时存在
    - 只有一个open transaction——接收系统调用
    - 若干个正在commit的transaction
    - 若干个正在从cache中向文件系统写block的transaction
    - 若干个正在被释放的transaction


---

#### ext3文件系统调用格式

- 调用**start函数**，表示开始——类似xv6中的begin\_op，保证原子性
- 得到一个**handle**（它唯一识别当前的系统调用）
- 调用**get函数**，获取block在buffer中的缓存，同时告诉handle这个block需要被读写
- 调用**stop函数**表示结束（handle作为参数传入，让文件系统知道这是哪个transaction正在等待的系统调用；stop函数只是告诉logging系统当前transaction少了一个正在进行的系统调用，只有所有系统调用都执行stop后才能commit）

---

#### transaction commit步骤

- 阻止新的系统调用
- 等待包含在transaction中的所有系统调用结束

*此时可以开始一个新的transaction了*

- 更新[**descriptor block**](onenote:#Lec16%20File%20system%20performance%20%20fast%20crash%20recovery&section-id={7EB9EBCE-898C-462A-BBF8-2771E8E465C5}&page-id={E0AE5CF6-1CB1-4E87-A562-DE0A9CA8D86F}&object-id={92BADABD-11D1-0435-3CC6-8EA66F743832}&E9&base-path=https://d.docs.live.net/1f65032c09a11ca3/Documents/yuebaitu%20的笔记本/4各学科/3CS/玥自学/操作系统/lecture%20notes.one)
- 将被修改的block**从缓存写入磁盘的log**
- 写入[**commit block**](onenote:#Lec16%20File%20system%20performance%20%20fast%20crash%20recovery&section-id={7EB9EBCE-898C-462A-BBF8-2771E8E465C5}&page-id={E0AE5CF6-1CB1-4E87-A562-DE0A9CA8D86F}&object-id={C314ED60-9450-02DA-2781-75649ED50109}&38&base-path=https://d.docs.live.net/1f65032c09a11ca3/Documents/yuebaitu%20的笔记本/4各学科/3CS/玥自学/操作系统/lecture%20notes.one)

*到达**commit point***

- 将transaction包含的block**写入文件系统的实际位置**

*现在可以*[*重用*](onenote:#Lec16%20File%20system%20performance%20%20fast%20crash%20recovery&section-id={7EB9EBCE-898C-462A-BBF8-2771E8E465C5}&page-id={E0AE5CF6-1CB1-4E87-A562-DE0A9CA8D86F}&object-id={287DF73E-AFFC-0097-3867-DCDE8DA5176D}&F6&base-path=https://d.docs.live.net/1f65032c09a11ca3/Documents/yuebaitu%20的笔记本/4各学科/3CS/玥自学/操作系统/lecture%20notes.one)*这个transaction对应的log空间*

---

#### ext3恢复过程

- 释放某段log空间时，文件系统会更新[**super block**](onenote:#Lec16%20File%20system%20performance%20%20fast%20crash%20recovery&section-id={7EB9EBCE-898C-462A-BBF8-2771E8E465C5}&page-id={E0AE5CF6-1CB1-4E87-A562-DE0A9CA8D86F}&object-id={6D6F82EC-6958-0C89-17BF-197687E2D494}&45&base-path=https://d.docs.live.net/1f65032c09a11ca3/Documents/yuebaitu%20的笔记本/4各学科/3CS/玥自学/操作系统/lecture%20notes.one)的指针，让其指向**当前最早的transaction的起始位置**
- 恢复软件读取**super block**，找到log的**起始位置**，一直扫描找到**结束位置**
  - 需要<a href="onenote:#Lec16%20File%20system%20performance%20%20fast%20crash%20recovery&section-id={7EB9EBCE-898C-462A-BBF8-2771E8E465C5}&page-id={E0AE5CF6-1CB1-4E87-A562-DE0A9CA8D86F}&object-id={287DF73E-AFFC-0097-3867-DCDE8DA5176D}&31&base-path=https://d.docs.live.net/1f65032c09a11ca3/Documents/yuebaitu%20的笔记本/4各学科/3CS/玥自学/操作系统/lecture%20notes.one">区分</a>descriptor block和随机的data block
  - 防止data block也以魔法数字开头：写data block时用0替换前32bit的魔法数字，并在descriptor block中为该data block设置一个bit（表示替换），将block写回文件系统之前，会用魔法数字替换回来
  - 恢复软件从super block指向的开始位置一直扫描，直到
    - 某个commit block之后不是descriptor block
    - 或某个commit block之后是descriptor block，但根据descriptor block找到的并不是commit block

  停止扫描，并认为最后一个有效的commit block是结束位置


- 回到log的开始位置，将每个block**写入文件系统的实际位置**
- 启动剩下的操作系统

---

#### 为什么新的transaction需要等前一个transaction的系统调用全部执行完

e.g.，T1只包含一个create系统调用（创建文件x），在create结束之前，文件系统开始了一个新的transaction T2，它包含一个unlink系统调用（释放与文件y相关的inode）

- T2**将y的inode标记为空闲**
- T1的create**为x分配了y的inode**
- T1结束并commit了，但T2尚未结束，此时**crash**了
- 重启恢复时，由于T1已commit但T2没有，所以恢复软件**忽略T2**——y的inode没有被unlink，而x也在用这个inode，两个文件使用同一个inode！

---

#### Summary

- **log**保证多个步骤的写磁盘操作具有**原子性**；在crash之后，这些写操作**要么都发生**，**要么都不发生**
- **write ahead rule**——必须在任何实际修改前，将所有更新commit到log中
- **批量执行** & **并发**带来了可观的**性能**提升，但同时也增加了**复杂性**





