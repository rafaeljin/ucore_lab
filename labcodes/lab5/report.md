#### Lab5 report

###### 练习0:填写已有实验

###### 本实验依赖实验1/2/3/4。请把你做的实验1/2/3/4的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”的注释相应部分。注意:为了能够正确执行lab5的测试应用程序,可能需对已完成的实验1/2/3/4的代码进行进一步改进。
***
**meld**对比填写。
***
###### 练习0.5:	加载应用程序并执行(需要编码)
***
update lab1:在时钟等待时需要释放CPU控制权，以及**去掉**print_ticks，否则由于核对输出信息时在**spin**和**waitkill**由于多出来的信息出错；

    current->need_resched = 1;
update lab4:set_links中将进程间的link处理了，之前的

	list_add(&proc_list, &(proc->list_link));
    nr_process ++;
在set_links已经完成应该删除。

关于**T_SYSCALL**的用户级权限设置在lab1时已经妥当设置。
***
###### 练习1:	加载应用程序并执行(需要编码)
###### do_execv函数调用load_icode(位于kern/process/proc.c中)来加载并解析一个处于内存中的ELF执行文件格式的应用程序,建立相应的用户内存空间来放置应用程序的代码段、数据段等,且要设置好proc_struct结构中的成员变量trapframe中的内容,确保在执行此进程后,能够从应用程序设定的起始执行地址开始执行。需设置正确的trapframe内容。
***
load_icode将二进制代码填入已经清空了mm的proc_struct结构，并将栈、trapframe等设置好。这里的工作主要是完成trapframe的设置。

trapframe中cs设置为用户代码段，ds，es，ss都设置为用户数据段，esp设置为用户栈地址，eip设置为前面加载进来的elf可执行代码的入口。根据**mmu.h**将tf_eflags设置为中断标志，产生中断。

***
###### 练习2:父进程复制自己的内存空间给子进程(需要编码)
###### 创建子进程的函数do_fork在执行中将拷贝当前进程(即父进程)的用户内存地址空间中的合法内容到新进程中(子进程),完成内存资源的复制。具体是通过copy_range函数(位于kern/mm/pmm.c中)实现的,请补充copy_range的实现,确保能够正确执行。
***
根据要求，将page（被复制页）通过memcpy到npage（前面分配了新的页）后与(to,start)建立映射关系。

	void *src_kvaddr = page2kva(page);
    void *dst_kvaddr = page2kva(npage);
    memcpy(dst_kvaddr, src_kvaddr, PGSIZE);
    ret = page_insert(to, npage, start, perm);
代码量较少，根据提示也比较容易，根据assert(ret==0)以及前几次的经验也很容易考虑到出错情况，和参考答案基本一致。
***
###### 练习3:阅读分析源代码,理解进程执行fork/exec/wait/exit的实现,以及系统调用的实现(不需要编码)
###### 执行:make grade。如果所显示的应用程序检测都输出ok,则基本正确。(使用的是qemu-1.0.1)
***
* fork: 创建新的进程，即初始化proc_struct，使子进程为RUNNABLE；
* exec: 清空mm，调用load_icode将程序拷入，分配好新的栈和内存空间，转入用户态执行；
* wait: 若发现满足条件的子进程则回收相关资源，否则进入SLEEPING；
* exit: 若父进程为等待状态则唤醒父进程，将子进程的父进程设置为init_proc。

***