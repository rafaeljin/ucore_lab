#### Lab7 report

###### 练习0:填写已有实验
###### 本实验依赖实验1/2/3/4/5/6。请把你做的实验1/2/3/4/5/6的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”/“LAB5”/“LAB6”的注释相应部分。并确保编译通过。注意:为了能够正确执行lab7的测试应用程序,可能需对已完成的实验1/2/3/4/5/6的代码进行进一步改进。
***
**meld**对比填写。
***
###### 练习1:	理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题(不需要编码)
###### 完成练习0后,建议大家比较一下(可用kdiff3等文件比较软件)个人完成的lab6和练习0完成后的刚修改的lab7之间的区别,分析了解lab7采用信号量的执行过程。执行make	grade,大部分测试用例应该通过。
***
首先看到信号量的定义**sem.h**：

    typedef struct {
    	int value;
    	wait_queue_t wait_queue;
	} semaphore_t;
其中value是信号量的值，wait_queue是信号量的等待队列，和课件的完全一致。以及4个函数，

	void sem_init(semaphore_t *sem, int value);
    void up(semaphore_t *sem);
    void down(semaphore_t *sem);
    bool try_down(semaphore_t *sem);
从上到下分别对应于课件中的信号量初始化，V和P操作。最后一个是一种不阻塞的P操作，下面具体分析**sem.c**中的实现。每个具体实现的部分都有：

	local_intr_save(intr_flag);
    ...
    local_intr_restore(intr_flag);
用于设置临界区。在ucore中由于是单处理机，可以通过关中断实现**临界区**代码保护。其中**down()**和**up()**实现思路和可见中无异，不再赘述。**try_down**在之前没有提到，是一种不阻塞的**down()**，

	bool intr_flag, ret = 0;
	if (sem->value > 0) {
        sem->value --, ret = 1;
    }
尝试获得锁，如果失败，不进入等待队列，而是直接返回。

哲学家就餐问题：**check_sync.c**中**philosopher_using_semaphore()**为用信号量实现的哲学家就餐问题的主循环。这里的实现和理论课的实现不同，锁不设置在叉子上，而是设置在哲学家上。由于取放动作相比吃饭和思考的时间较短，这里的实现是：**将取放叉子动作作为临界区**，即每个时刻只有一个哲学家可以做出动作。**phi_take_forks_sema()**:在临界区尝试拿叉子，如果失败，则将进入等待队列（只有1个元素的等待队列）；**phi_put_forks_sema**：进入test左右邻居，如果满足就餐条件就将其唤醒；**phi_test_sema**：test中意义比较复杂，在自身拿起测试的时候意义是进入吃饭状态，在别人放下测试的时候在于把目标设置为吃饭状态并唤醒。

从这个例子也可以看到，信号量的配对问题。如果情况更加复杂，出现更多的信号量，分散于更多的函数，那么在写这部分代码很容易出现配对错误，并且难于调试。

***
###### 练习2：	完成内核级条件变量和基于内核级条件变量的哲学家就餐问题(需要编码)
###### 首先掌握管程机制,然后基于信号量实现完成条件变量实现,然后用管程机制实现哲学家就餐问题的解决方案(基于条件变量)。
###### 执行:make grade。如果所显示的应用程序检测都输出ok,则基本正确。如果只是某程序过不去,比如matrix.c,则可执行 make run-matrix命令来单独调试它。大致执行结果可看附录。(使用的是qemu-1.0.1)。
***
**monitor.h**中条件变量的定义：

    typedef struct condvar{
        semaphore_t sem;
        int count;              // the number of waiters on condvar
        monitor_t * owner;      // the owner(monitor) of this condvar
    } condvar_t;
结构类似信号灯，但是这里的int的意义是等待该条件的个数，而不是实际意义的资源；这里借用信号灯实现wait()和signal()。根据伪代码，很容易可以得到**monitor.c**中wait()和signal()的实现：

	// signal()
	if(cvp -> count > 0){		// if exist proc in condition's wait queue
		monitor_t *mtp = cvp -> owner;
		mtp -> next_count++;
        up(&cvp -> sem);		// wake up a proc waiting for condition
        down(&mtp -> next);	  	// put itself into monitor's next list
		mtp -> next_count--;
	}
可以看到signal()的实现思路清晰，如果有等待该条件的，则将其唤醒，自身进入monitor的wait list；

	// wait()
	cvp -> count++;
    if (mtp -> next_count > 0)		// if monitor's next list has proc
    	up(&mtp -> next);			// wake it up
    else
        up(&mtp -> mutex);			// else release mutex
    down(&cvp -> sem);				// wait for condition
    cvp -> count--;
wait()中检查管程的next队列，如果有，则唤醒它，否则直接放弃临界锁；然后进入等待条件变量。
这里的实现可以看到是**Hoare管程**，即signal以后优先等待条件变量的进程。

哲学家就餐问题：**check_sync.c**中**philosopher_using_condvar()**为用信号量实现的哲学家就餐问题的主循环，实现思想和之前信号量中一样，不再赘述。这里让我们填写的代码非常简单，管程入口和出口的代码比较关键：

    down(&(mtp->mutex));
    //--------into routine in monitor--------------
    ...
    //--------leave routine in monitor--------------
    if(mtp->next_count>0)
	    up(&(mtp->next));
    else
		up(&(mtp->mutex));
这里体现了管程的思想：管程代码任何时刻只有一个线程在运行。出口中还需考虑如果管程next队列中有由于**signal()**进入睡眠的线程应当先唤醒，否则可能出现该线程永远睡眠的情况。这里的down不仅和出口的up配对，还和wait()中放弃锁的up配对，体现了管程和信号量的区别。

在phi_take_forks_condvar()和phi_put_forks_condvar()的具体的实现上和信号量很相似（**甚至可以说上面的信号量实现更像特意在使用信号量实现管程**），这里只说以下和参考答案的区别：

	if (state_condvar[i] != EATING)
在之前也提到了这是Hoare的实现，用if就可以了，我使用的是if。但答案中的while显然也对。
***
