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
**这里有个很重要的点就是：t->ctx.sp = (uint64)(t->stack)+STACK_SIZE; **  
因为sp（栈指针）是从高地址向低地址变化的，所以线程的初始sp应该在高地址。   
实际上，我最开始做的时候将sp放在了低地址，这就导致当“thread 1”运行时，因为“thread 1”运行在自己的“栈”上（这个“栈”实际上使我们通过“sturct thread”分配的空间，是一段连续的内存），而此时的“sp”随着地址的减小，很显然会干扰到前一个线程的“栈”空间保存的内容，此时读取“all_thread[0]->state”（也就是thread 0线程）的状态就错误了，会出现很离谱的数据。  
这里画出各个线程的内存空间（或者说运行空间），包括了栈、状态、切换所需的上下文信息。  

这个问题困扰了自己很久。实际上，最开始的时候很懒，不想使用“GDB”调试，觉得自己完全能做好这些，结果还是在找不出错误的时候依靠“GDB”调试，一步一步直到发现出现错误的指令，这才找到问题。**所以，不要想着走捷径！**  
这一部分其他的代码很简单，就不给出了。如果需要，请查看源码“user/uthread.c和user/uthread_switch.S”。  

