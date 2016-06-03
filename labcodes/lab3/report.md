#### Lab3 report

###### 练习0:填写已有实验

###### 本实验依赖实验1/2。请把你做的实验1/2的代码填入本实验中代码中有“LAB1”,“LAB2”的注释相应部分。
***
使用**meld**工具对比lab2和lab3。使用之前担心学习工具需要花费大量时间，但担心是多余的。**meld**使用非常方便，图形化的形式展现出2个工程的不同，通过简单点击就可以进行merge。
***
###### 练习1:给未被映射的地址映射上物理页(需要编程)
######完成do_pgfault(mm/vmm.c)函数,给未被映射的地址映射上物理页。设置访问权限	的时候需要参考页面所在	VMA	的权限,同时需要注意映射物理页时需要操作内存控制	结构所指定的页表,而不是内核的页表。注意:在LAB2	EXERCISE	1处填写代码。执行 make qemu后,如果通过check_pgfault函数的测试后,会有“check_pgfault()	succeeded!”的输出,表示练习1基本正确。
***

根据程序说明完成还是比较容易，通过这部分的前后代码阅读还理解了进程拥有独立的页表来实现虚拟地址，本函数的参数**mm**即与进程相关。

在写为物理地址分配page的时候没有去看**pgdir_alloc_page**内容，又通过和lab2类似的方式将返回的page写入***ptep**：

	*ptep = page2pa(page) | PTE_P | PTE_U;
事实上，上述部分还有一个错误，可以看到**pgdir_alloc_page** -> **page_insert**中有，

	*ptep = page2pa(page) | PTE_P | perm;
    tlb_invalidate(pgdir, la);
完成了将整个写入pte的工作。

和答案对比，少了对于get_pte是NULL的情况以及在alloc page的时候可能是NULL的情况，对比后加入报错信息和**goto fail**。

	failed:
        return ret;
***
###### 练习2:补充完成基于FIFO的页面替换算法(需要编程)
###### 完成vmm.c中的do_pgfault函数,并且在实现FIFO算法的swap_fifo.c中完成map_swappable和swap_out_vistim函数。通过对swap的测试。注意:在LAB2	EXERCISE	2处填写代码。执行 make qemu 后,如果通过check_swap函数的测试后,会有“check_swap()	succeeded!”的输出,表示练习2基本正确。
***
这部分按照提示完成不难，do_pgfault根据给的信息（判断flag和虚拟地址合法性等），如果该虚拟地址在页表项中不存，那么通过创建新的项，并在物理内存上分配（以及不存在页表也要分配）；

如果该页表中已经存在，那么需要把这个page swap_in。观察调用情况：**swap_in** -> **alloc_page** -> **swap_out**，会处理整个swap过程。接着通过将相关信息补充完整，并加入swap队列。

在**map swappable**中把新的项加入队尾：

	list_add_before(head,entry);
然后在**_fifo_swap_out_victim**中设置换出策略，比较容易。

和答案对比，除了队列方向相反和一些少了一些assert和出错报告。
***
