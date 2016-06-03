#### Lab6 report

###### 练习0:填写已有实验

###### 本实验依赖实验1/2/3/4/5。请把你做的实验2/3/4/5的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”“LAB5”的注释相应部分。并确保编译通过。注意:为了能够正确执行lab6的测试应用程序,可能需对已完成的实验1/2/3/4/5的代码进行进一步改进。
***
**meld**对比填写。
***
###### 练习1:	使用 Round Robin 调度算法(不需要编码)
###### 完成练习0后,建议大家比较一下(可用kdiff3等文件比较软件)个人完成的lab5和练习0完成后的刚修改的lab6之间的区别,分析了解lab6采用RR调度算法后的执行过程。执行make	grade,大部分测试用例应该通过。但执行priority.c应该过不去。
***
在ucore利用**struct+函数指针**实现类似面向对象中的类。下面是Round Robin调度算法定义的支撑框架：

    struct sched_class default_sched_class = {
        .name = "RR_scheduler",
        .init = RR_init,
        .enqueue = RR_enqueue,
        .dequeue = RR_dequeue,
        .pick_next = RR_pick_next,
        .proc_tick = RR_proc_tick,
    };
其中name对应于算法名字，之后定义了5个调度函数：
* init在程序开始执行，初始化了runnable队列以及进程数；


    list_init(&(rq->run_list));
    rq->proc_num = 0;
* enqueue为进队操作，加入到就绪队列尾并设置相关参数：


    list_add_before(&(rq->run_list), &(proc->run_link));
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;
    }
    proc->rq = rq;
    rq->proc_num ++;
* dequeue在进程被移出就绪队列时被调用：


	list_del_init(&(proc->run_link));
    rq->proc_num --;
* pick_next在进程切换时被调用，选择就绪队列中最后一个进程（NULL的话idle process）：


	list_entry_t *le = list_next(&(rq->run_list));
    if (le != &(rq->run_list)) {
        return le2proc(le, run_link);
    }
    return NULL;
* proc_tick在产生时钟中断时被调用，根据进程 时间片剩余与否选择减少时间还是换出：


	if (proc->time_slice > 0) {
        proc->time_slice --;
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;
    }
通过完成**proc.c**对新加入的元素初始化以及**trap.c**添加run_timer_list处理时钟后，make grade除了priority都通过。
***
###### 练习2:实现 Stride Scheduling 调度算法(需要编码)
###### 首先需要换掉RR调度器的实现,即用default_sched_stride_sched.c覆盖default_sched.c。然后根据此文件和后续文档对Stride度器的相关描述,完成Stride调度算法的实现。后面的实验文档部分给出了Stride调度算法的大体描述。这里给出Stride调度算法的一些相关的资料(目前网上中文的资料比较欠缺)。
* http://wwwagss.informatik.uni-kl.de/Projekte/Squirrel/stride/node3.html
* http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.138.3502&rank=1
* 也可GOOGLE “Stride Scheduling” 来查找相关资料

###### 执行:make	grade。如果所显示的应用程序检测都输出ok,则基本正确。如果只是priority.c过不去,可执行	make run-priority 命令来单独调试它。大致执行结果可看附录。(	使用的是	qemu-1.0.1)。
***
在default_sched.c中注释default_sched_class的定义，引入default_sched_stride_class.c文件实现更换调度算法。根据
	#define USE_SKEW_HEAP 1
的值选取skew heap还是list，两种方法测试通过。**stride_init**中分别调用函数初始化2种数据结构和设置初始进程数；**stride_enqueue**入队，并设置好进程时间片；**stride_dequeue**野比较简单，两种数据结构已经定义好出队函数，分别调用即可；**stride_pick_next**中斜堆通过取顶点（即最小值，但这里数据结构已经提供现成接口），使用列表则通过遍历找到最小值；最后是设置正确的stride（使用大值除以特权级）；**stride_proc_tick**在被触发时减少时间片，如果为0则将need_resched设置为1；这里**BIG_STRIDE**没有设置为INT的最大值，而是稍微小一些，因为没有处理溢出。

***
