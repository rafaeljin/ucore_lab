#### Lab4 report

###### 练习0:填写已有实验

###### 本实验依赖实验1/2。请把你做的实验1/2的代码填入本实验中代码中有“LAB1”,“LAB2”的注释相应部分。
***
**meld**对比填写。
***
###### 练习1:分配并初始化一个进程控制块(需要编码)
######alloc_proc函数(位于kern/process/proc.c中)负责分配并返回一个新的struct	proc_struct结构,用于存储新建立的内核线程的管理信息。ucore需要对这个结构进行最基本的初始化,你需要完成这个初始化过程。【提示】在alloc_proc函数的实现中,需要初始化的proc_struct结构中的成员变量至少包括: state/ pid/ runs/ kstack/ need_resched /parent /mm /context / tf/ cr3 / flags/ name。
***
alloc_proc先利用kmalloc在内存上初始化分配一个proc_struct结构，并且完成相关域填写，可以看到在proc_init中先调用alloc_proc分配一个proc_struct，再完成对应的初始化工作。

这里几个需要注意的域：pid设为-1，说明未初始化，state设置为PROC_UNINIT表示还没初始化，cr3为boot_cr3，表示使用内核页表，其他的通过memset初始化为0。

对比答案，这里没有逐个初始化为0。

***
###### 练习2:为新创建的内核线程分配资源(需要编码)
###### 创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用do_fork函数完成具体内核线程的创建工作。do_kernel函数会调用alloc_proc函数来分配并初始化一个进程控制块,但alloc_proc只是找到了一小块内存用以记录进程的必要信息,并没有实际分配这些资源。ucore一般通过do_fork实际创建新的内核线程。do_fork的作用是,创建当前内核线程的一个副本,它们的执行上下文、代码、数据都一样,但是存储位置不同。在这个过程中,需要给新内核线程分配资源,并且复制原进程的状态。你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。它的大致执行步骤包括:
###### 1，调用alloc_proc,首先获得一块用户信息块。
###### 2，为进程分配一个内核栈。
###### 3，复制原进程的内存管理信息到新进程(但内核线程不必做此事)
###### 4，复制原进程上下文到新进程
###### 5，将新进程添加到进程列表
###### 6，唤醒新进程
###### 7，返回新进程号
***
根据步骤概述提示写比较简单，每一步都提供了相关的函数，只要将相关参数填入即可，注意出错的可能（根据返回结果判断），分别跳转到fork_out（第一步分配失败即alloc_proc返回NULL），bad_fork_cleanup_kstack（需要清除和释放内核栈和proc_struct结构即在kstack和proc都分配后出现问题）和bad_fork_cleanup_proc（仅释放proc_struct即在kstack分配时出错）。

需要注意的是在第5步需要保存intr_flag，防止多线程不同步。还有需要注意的是大致步骤里面没有提示完成相关域的初始化。大部分的域如state，kstack，tf等都已经在相关函数中完成，但需要将pid和parent手动填入。

对比答案开始时想得太简单，有些细节没有考虑，也出现过由于**==**和**=**优先级的低级错误。
***
###### 练习3:阅读代码,理解	proc_run和它调用的函数如何完成进程切换的。(无编码工作)
###### 完成代码编写后,编译并运行代码:make	qemu，如果可以得到如附录A所示的显示内容(仅供参考,不是标准答案输出),则基本正确。
***
proc_run运行proc。在关键代码：

	current = proc;
	load_esp0(next->kstack + KSTACKSIZE);
    lcr3(next->cr3);
    switch_to(&(prev->context), &(next->context));
从第二行到第四行分别做的是，切换栈，载入页表，切换上下文。switch_to中分别做了保存prev的上下文以及载入到next的上下文。
***