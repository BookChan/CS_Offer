在多道程序环境下，允许多个程序并发执行。为此引入了进程(Process)的概念，以便更好地描述和控制程序的并发执行，实现操作系统的并发性和共享性。简单来说，进程是对正在运行的程序的一个抽象。一个进程就是一个正在执行程序的实例，包括程序计数器、寄存器和变量的当前值。

# 进程

从不同的角度，进程可以有不同的定义，比较典型的定义有：

* 进程是程序的一次执行过程。
* 进程是一个程序及其数据在处理机上顺序执行时所发生的活动。
* 进程是具有独立功能的程序在一个数据集合上运行的过程，它是系统进行资源分配和调度的一个独立单位。

## 进程创建、终止

操作系统需要有一种方式来创建进程，主要有4种事件导致进程的创建：

* 系统初始化；
* 执行了进程创建系统调用；
* 用户请求创建一个新进程；
* 一个批处理作业的初始化

技术上来看，所有这些情形中，新进程都是由于一个已经存在的进程执行了一个用于创建进程的系统调用而创建的。UNIX系统中，只有一个系统调用可以用来创建新进程：fork。这个系统调用会创建一个与调用进程相同的副本。调用了 fork 之后，这两个进程拥有相同的存储映像、同样的环境字符串和同样的打开文件。

进程创建之后，子进程和父进程拥有不同的地址空间，如果其中某个进程在其地址空间修改了一个字，这个修改对其他进程是不可见的。

永恒是不存在的，进程也一样，一个进程迟早会结束，通常由下列条件引起：

* 正常退出（自愿的）
* 出错退出（自愿的）
* 严重错误（非自愿）
* 被其他进程杀死（非自愿）

## 进程状态转换

进程在其生命周期内，由于系统中各进程之间的相互制约关系及系统的运行环境的变化，使得进程的状态也在不断地发生变化（一个进程会经历若干种不同状态）。通常进程有三种状态。

* 运行状态：进程正在 CPU 上运行。在单处理机环境下，每一时刻最多只有一个进程处于运行状态。
* 就绪状态：进程已处于准备运行的状态，即进程获得了除 CPU 之外的一切所需资源，一旦得到 CPU 即可运行。
* 阻塞状态，又称等待状态：进程正在等待某一事件而暂停运行，如等待某资源为可用（不包括CPU）或等待输入/输出完成。即使CPU空闲，该进程也不能运行。

下图说明了进程间状态转换的过程：

![][1]

其中：

1. 进程因为等待资源或事件而阻塞；
2. 调度程序选择了另一个进程；
3. 调度程序选择这个进程
4. 进程获得资源或事件

## 进程实现

为了实现进程模型，操作系统维护着一张表格（一个结构数组），即进程表，每个进程占用一个进程表项（也叫进程控制块, PCB）。系统利用`进程控制块`（Process Control Block, PCB）来描述进程的基本情况和运行状态，进而控制和管理进程。PCB是进程存在的唯一标志！典型 PCB 中的一些字段如下：

![][2]

## 僵尸、孤儿进程

子进程先于父进程结束，而且父进程没有函数调用 wait() 或 waitpid() 等待子进程结束，也没有注册 SIGCHLD 信号处理函数，结果使得子进程的进程列表信息无法回收，这样子进程就变成了`僵尸进程（Zombie）`。

所以简单**来说一个已经终止，但是其父进程尚未对其进行善后处理**（终止子进程的有关信息）的进程被称为僵尸进程。

UNIX 提供了一种机制可以保证只要父进程想知道子进程结束时的状态信息，就可以得到。这种机制就是: 在每个进程(init除外)退出的时候，内核释放该进程所有的资源，包括打开的文件，占用的内存等。但是仍然为其保留一定的信息（包括进程号PID，退出状态，运行时间等)，直到父进程通过wait/waitpid 来取时才释放。

如果父进程不调用wait/waitpid的话，那么保留的那段信息就不会释放，其进程号就会一直被占用（这时用ps命令就能看到子进程的状态是“Z”），但是系统所能使用的进程号是有限的，如果大量的产生僵死进程，将因为没有可用的进程号而导致系统不能产生新的进程。此即为僵尸进程的危害，应当避免。

注意如果父进程先于子进程结束，这时的子进程应该称作`孤儿进程（Orphan）`。每当出现一个孤儿进程的时候，内核就把孤儿进程的父进程设置为init，而init进程会循环地调用wait()。这样，当一个孤儿进程结束其生命周期后，init进程就会进行善后工作。因此孤儿进程并不会有什么危害。

# 线程

线程最直接的理解就是“`轻量级进程`”，它是一个基本的CPU执行单元，也是程序执行流的最小单元，由线程ID、程序计数器、寄存器集合和堆栈组成。线程是进程中的一个实体，是被系统独立调度和分派的基本单位，线程自己不拥有系统资源，只拥有一点在运行中必不可少的资源，但它可与同属一个进程的其他线程共享进程所拥有的全部资源。

进程中使用线程的主要原因有以下几个：

* 通过将应用程序分解成可以准并行运行的多个顺序线程，程序设计模型会变得简单。因为有了多线程之后，并行实体可以共享同一个地址空间和所有可用数据，这是多进程模型（它们具有不同的地址空间）所无法表达的。
* 线程比进程更加轻量级，比进程更容易创建，也更容易撤销。
* 如果存在着大量的计算和大量的 I/O 处理，拥有多个线程允许这些活动彼此重叠进行，从而加快应用程序执行的速度。

进程用于把资源集中到一起，而线程则是在 CPU 上被调度执行的实体。在同一个进程中并行运行多个线程，是对在同一个计算机上并行运行多个进程的模拟。在前一种情形下，多个线程共享同一个地址空间和其他资源，而在后一种情况下，多个进程共享物理内存、磁盘、打印机和资源。

进程中的不同线程不像不同进程之间那样存在很大的独立性，所有的线程都有完全一样的地址空间，它们也共享同样的全局变量。由于每个线程都可以访问进程地址空间中的每一个内存地址，所以一个线程可以读、写甚至清除另一个线程的堆栈。线程之间没有保护，一是没必要，二是不可能。

和传统进程一样，线程可以处于若干个状态中的任何一个：运行、阻塞、就绪、终止，之间的互相转换和进程也是一样的。多线程情况下，进程通常会从当前的单个线程开始，这个线程有能力通过调用一个库函数（如 thread_create）创建新的线程，通常情况，线程之间是平等关系的。一个线程完成工作后，调用一个库过程（如 thread_exit）退出。

线程除了共享进程所拥有的资源之外，每个线程还独有一些内容，如下：

* 线程 ID
* `寄存器`
* 线程堆栈指针
* 程序计数器

线程间通信方式主要有：`事件、临界区、互斥量、信号量`。

## 用户级线程

有两种方式实现线程，在用户空间中和在内核中，各有利弊。

在`用户级线程`中，有关线程管理的所有工作都由应用程序完成，内核意识不到线程的存在。应用程序可以通过使用线程库设计成多线程程序。通常，应用程序从单线程起始，在该线程中开始运行，在其运行的任何时刻，可以通过调用线程库中的派生例程创建一个在相同进程中运行的新线程。

![][3]

优点主要有：

* 可以在不支持线程的操作系统上实现（只需要有线程函数库即可）；
* 线程切换比陷入内核要快一个数量级（调度程序也是本地程序，比内核调用效率高）；
* 允许每个进程有自己定制的调度算法。

当然，也有一些问题：

* `阻塞系统调用`问题。对应用程序来讲，同一进程中只能同时有一个线程在运行，一个线程的阻塞将导致整个进程中所有线程的阻塞。
* `页面故障`问题。如果有页面引起缺页中断，通常会阻塞整个进程直到磁盘I/O完成，尽管其它的线程是可以运行的。
* 如果一个线程开始运行，那么在该线程中的其它线程就不能运行，除非第一个线程自动放弃 CPU（thread_yelid）。

## 内核级线程

在内核级线程中，线程管理的所有工作由内核完成，应用程序没有进行线程管理的代码，只有一个到内核级线程的编程接口。内核为进程及其内部的每个线程维护上下文信息，调度也是在内核基于线程架构的基础上完成。

![][4]

内核级线程存在的一些问题：

* 所有能够阻塞线程的调用都以系统调用的形式实现，代价相当可观；
* 多线程进程创建新的进程时，新进程是否需要复制所有的线程；

## 混合实现

在一些系统中，使用组合方式的多线程实现。线程创建完全在用户空间中完成，线程的调度和同步也在应用程序中进行。一个应用程序中的多个用户级线程被映射到一些（小于或等于用户级线程的数目）内核级线程上。

![][5]

## 线程安全

`线程安全`就是多线程访问时，采用了加锁机制，当一个线程访问某个数据时，进行保护，其他线程不能进行访问直到该线程访问完毕，其他线程才可使用。不会出现数据不一致或者数据污染。线程不安全就是不提供数据访问保护，有可能出现多个线程先后更改数据造成所得到的数据是脏数据。

线程安全问题都是由`全局变量`或者`静态变量`引起的。若每个线程中对全局变量、静态变量只有读操作，而无写操作，一般来说是线程安全的；若有多个线程同时执行写操作，一般都需要考虑线程同步，否则的话就可能影响线程安全。

POSIX线程标准要求C标准库中的大多数函数具备线程安全性。

［[线程并发执行结果](http://www.nowcoder.com/questionTerminal/80c7cc46948c4263bc806e3f0e049bbc)］

## 进程与线程

下面主要从调度、并发性、系统开销、拥有资源等方面来对线程与进程进行比较。

1. `调度`：线程作为调度和分配的基本单位，进程作为拥有资源的基本单位。
2. `并发性`：在引入线程的操作系统中，不仅进程之间可以并发执行，而且在一个进程中的多个线程之间也可以并发执行，使得操作系统具有更好的并发性，从而能更加有效地使用系统资源和提高系统的吞吐量。  
3. `拥有资源`：进程是拥有资源的基本单位，一般地说，线程自己不拥有系统资源（也有一点必不可少的资源），但它可以访问其隶属进程的资源，即一个进程的代码段、数据段及所拥有的系统资源（如已打开的文件、I/O设备等），可供该进程中的所有线程所共享。
4. `系统开销`：在创建或撤销进程时，系统都要为之分配和回收进程控制块、内存空间、I/O设备等，因此操作系统所付出的开销显著地大于创建或撤销线程时的开销。
5. `上下文切换`：在进行进程切换时，涉及到当前进程CPU环境的保存及新被调度运行进程CPU环境的设置。而线程的切换只需要保存和设置少量寄存器的内容，不涉及存储器管理方面的操作。可见，进程切换的开销也远大于线程切换的开销。
6. `数据共享`：由于同一个进程中的多个线程具有相同的地址空间，在同步和通信的实现方面线程也比进程容易。在一些操作系统中，线程的切换、同步和通信都无须操作系统内核的干预。

# 调度

在多道程序系统中，进程的数量往往多于 CPU 的个数，进程争用 CPU 的情况就在所难免。调度是对 CPU 进行分配，就是从就绪队列中，按照一定的算法（公平、髙效）选择一个进程并将 CPU 分配给它运行，以实现进程并发地执行。

调度的时机：

1. 创建一个新的进程之后，必须决定运行父进程还是子进程；
2. 在一个进程退出时必须做出调度决策；
3. 当一个进程阻塞在 I／O 或者信号量上或由于其它原因阻塞时，必须选择另一个进程运行；
4. 在一个I／O 中断发生时，必须做出调度决策。

不同的环境需要不同的调度算法，主要分下面三种环境：

* 批处理：一定时间做好一定的事情
* 交互式：快速响应用户的请求
* 实时系统：或多或少必须满足截止时间

什么是好的调度算法？

* 批处理系统：通常检查三个指标：吞吐量（每小时最大作业数）、周转时间（从提交到终止间的最小时间）、CPU利用率（保持CPU始终忙碌）
* 交互式系统：最重要的指标是最小响应时间（满足快速响应请求），均衡性（满足用户的期望）
* 实时系统：最主要的要求是满足所有（或大多数）的截止时间要求

批处理系统中的调度算法：

* `先来先服务（FCFS）`。易于理解且便于在程序中运行，缺点是I/O密集型操作导致效率低下；
* `最短作业优先（SPF）`。非抢占式的批处理调度算法。
* `最短剩余时间优先`。抢占式的算法，调度程序总是选择剩余运行时间最短的那个进程运行。

交互式系统的调度算法：

* `轮转调度`。每个进程被分配一个时间段（时间片），允许该进程在该时间段运行。时间片太短，过多的进程切换降低效率，过长引起对短的交互请求的响应时间变长。
* `优先级调度`。每个进程被赋予一个优先级，允许优先级最高的可运行进程先运行。

［[作业调度设备利用率](http://www.nowcoder.com/questionTerminal/683d207653d9460ba9b60418695f2c8d)］  
［[响应比高者优先调度](http://www.nowcoder.com/questionTerminal/9a714e7cb8fe4d158aa230ec7277e6a1)］
  
## 并发、并行

并发和并行的区别就是一个处理器同时处理多个任务和多个处理器或者是多核的处理器同时处理多个不同的任务。前者是逻辑上的同时发生，而后者是物理上的同时发生．

* 并发(concurrent)：指能处理多个同时性活动的能力，并发事件之间不一定要同一时刻发生。
* 并行(parallel)：指同时发生的两个并发事件，具有并发的含义，而并发则不一定并行。

# 死锁

死锁的规范定义如下：**如果一个进程集合中的每个进程都在等待只能由该进程集合中其他进程才能引发的事件，那么该进程集合就是死锁的。**

产生死锁的原因主要是：

- 因为系统资源不足。
- 进程运行推进的顺序不合适。
- 资源分配不当等。

产生死锁的四个必要条件：

1. 互斥条件：每个资源要么已经分配给了一个进程，要么就是可用的。
2. 占有和等待条件：已经得到了某个资源的进程可以再请求新的资源。
3. 不可抢占条件：已经分配给一个进程的资源不能强制性地被抢占，只能被占有它的进程显式地释放；
4. 环路等待条件：死锁发生时，系统中一定有两个或者两个以上的进程组成的一条环路，该环路中的每个进程都在等待着下一个进程所占有的资源。

这四个条件是死锁的必要条件，只要系统发生死锁，这些条件必然成立，而只要上述条件之一不满足，就不会发生死锁。

四种处理死锁的策略：

1. 鸵鸟策略（忽略死锁）；
2. 检测死锁并恢复；
3. 仔细对资源进行分配，动态地避免死锁；
4. 通过破坏引起死锁的四个必要条件之一，防止死锁的产生。

避免死锁的主要算法是基于一个`安全状态`的概念。在任何时刻，如果没有死锁发生，并且即使所有进程忽然请求对资源的最大请求，也仍然存在某种调度次序能够使得每一个进程运行完毕，则称该状态是安全的。从安全状态出发，系统能够保证所有进程都能完成，而从不安全状态出发，就没有这样的保证。

`银行家算法`：判断对请求的满足是否会进入不安全状态，如果是，就拒绝请求，如果满足请求后系统仍然是安全的，就予以分配。不安全状态不一定引起死锁，因为客户不一定需要其最大贷款额度。

［[死锁产生必要条件](http://www.nowcoder.com/questionTerminal/28e91f200206451b9ee44dd1613f94ce)］  
［[资源一定，进程最多申请多少资源](http://www.nowcoder.com/questionTerminal/18b1f01c1901424382735d5d158a8f7f)］

# 更多阅读

《UNIX网络编程》  
《现代操作系统》  
《UNIX 环境高级编程》    
[进程与线程的一个简单解释](http://www.ruanyifeng.com/blog/2013/04/processes_and_threads.html)  
[Linux的IPC命令](http://www.cnblogs.com/cocowool/archive/2012/05/22/2513027.html)  
[操作系统(计算机)进程和线程管理](http://c.biancheng.net/cpp/u/xitong_2/)    
[内核线程与用户线程的一点小总结](http://www.jianshu.com/p/5a4fc2729c17)   
[孤儿进程与僵尸进程](http://www.cnblogs.com/Anker/p/3271773.html)    



[1]: http://7xrlu9.com1.z0.glb.clouddn.com/Linux_OS_ProcessThread_1.png
[2]: http://7xrlu9.com1.z0.glb.clouddn.com/Linux_OS_ProcessThread_2.png
[3]: http://7xrlu9.com1.z0.glb.clouddn.com/Linux_OS_ProcessThread_3.png
[4]: http://7xrlu9.com1.z0.glb.clouddn.com/Linux_OS_ProcessThread_4.png
[5]: http://7xrlu9.com1.z0.glb.clouddn.com/Linux_OS_ProcessThread_5.gif


