因为找工作，很长一段时间没有进行操作系统实验了。所以，就先从这次的MultiThreading实验（难度中等）开始，慢慢做。  
**本次实验的目的是完成用户级别的线程切换，并熟悉多线程编程。  
课上学习的是内核的线程切换，用户级别的线程切换没有这么复杂，但仍然基于内核线程切换的思想（即切换pc、sp和相应的callee-saved寄存器）；
另一方面，学习到了pthread线程库的一些基础内容，比如互斥锁pthread_mutex_t和互斥锁相关的函数pthread_mutex_lock、pthread_mutex_unlock，再比如条件变量pthread_cond_t和条件变量相关的函数pthread_cond_wait、pthread_cond_broadcast等。**  
（实验链接：https://pdos.csail.mit.edu/6.S081/2020/labs/thread.html ）  
******  

实验一：完成用户级别的线程切换  
主要的要求是：  
（1）当线程第一次被切换(schedule)时，要能从传给该线程的“函数指针”处执行；并且当前线程应当运行在自己的栈上。代码如下：  
```
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->ctx.ra = (uint64)func;
  t->ctx.sp = (uint64)(t->stack)+STACK_SIZE;
}
```
**这里有个很重要的点就是：t->ctx.sp = (uint64)(t->stack)+STACK_SIZE;**  
因为sp（栈指针）是从高地址向低地址变化的，所以线程的初始sp应该在高地址。   
实际上，我最开始做的时候将sp放在了低地址，这就导致当“thread 1”运行时，因为“thread 1”运行在自己的“栈”上（这个“栈”实际上使我们通过“sturct thread”分配的空间，是一段连续的内存），而此时的“sp”随着地址的减小，很显然会干扰到前一个线程的“栈”空间保存的内容，此时读取“all_thread[0]->state”（也就是thread 0线程）的状态就错误了，会出现很离谱的数据。  
这里画出各个线程的内存空间（或者说运行空间），包括了栈、状态、切换所需的上下文信息。  
![](https://github.com/2351889401/MultiThreading/blob/main/images/sp.png)  
这个问题困扰了自己很久。实际上，最开始的时候很懒，不想使用“GDB”调试，觉得自己完全能做好这些，结果还是在找不出错误的时候依靠“GDB”调试，一步一步直到发现出现错误的指令，这才找到问题。**所以，不要想着走捷径！**  
这一部分其他的代码很简单，就不给出了。如果需要，请查看源码“user/uthread.c和user/uthread_switch.S”。  

对了，这一部分实际上还有一个问题没有明白，实验中描述如下：  
使用gdb帮助调试的时候，如果使用下面的命令：  
(gdb) file user/_uthread
Reading symbols from user/_uthread...
(gdb) b uthread.c:60

This sets a breakpoint at line 60 of uthread.c. The breakpoint may (or may not) be triggered before you even run uthread. How could that happen?  

实际上，也确实发现了这个问题，但是想不明白为什么会出现，下面是在gdb中调试的截图：  
![](https://github.com/2351889401/MultiThreading/blob/main/images/not_understand1.png)  
![](https://github.com/2351889401/MultiThreading/blob/main/images/not_understand2.png)  
可以从第1张图看到，当触发断点的时候，实际上操作系统的第一个用户进程“init”进程都还没有启动，因为如果启动的话会输出“init: starting sh”。  
第2张图是在gdb中查看线程的情况，发现是某个CPU上到了断点处，但是实际上并没有真正的执行，因为此时如果在gdb中使用“print”语句，查看“uthread.c”中全局变量的信息，实际上并没有。  
所以，自己仍然是不能理解这个原因，为什么程序在尚未运行的时候会被触发，甚至在操作系统的启动之前！  

实验二：基于pthread线程库解决“keys missing”的问题，并完成线程加速  
“keys missing”的问题好解决，主要就是通过加锁的方式，这里不去赘述；主要讨论的是线程加速的问题。  
两个线程对于哈希表（实际上是5个哈希桶，每个桶以链表形式存放key冲突的数据）的更新可能是：  
（1）同时insert：如果插入相同的哈希桶，需要加锁，否则会有“keys missing”；如果插入不同的哈希桶，不需要加锁  
（2）一个insert，一个update：无论操作的是不是相同的哈希桶，不需要互斥锁，因为本身就互不干扰  
（3）同时update：如果是不同的key值，不需要加锁；如果是相同的key值，对于本次实验不加锁也没关系，但实际上为了正确性应当加锁的。  
综上，只有同时“insert”插入相同哈希桶的情况，才需要加互斥锁，这种做法的效率也是比较高的，代码如下：  
```
//互斥锁的定义 每个哈希桶1个互斥锁
pthread_mutex_t myMutex[5];

//互斥锁的初始化
for(int i=0; i<NBUCKET; i++) pthread_mutex_init(&myMutex[i], NULL);

//在put函数中insert处使用互斥锁
pthread_mutex_lock(&myMutex[i]);
insert(key, value, &table[i], table[i]);
pthread_mutex_unlock(&myMutex[i]);
```  
下图的结果显示了双线程的插入速度，相比单线程，速度是 34915/16176=2.15 倍。  
![](https://github.com/2351889401/MultiThreading/blob/main/images/speed.png)  

