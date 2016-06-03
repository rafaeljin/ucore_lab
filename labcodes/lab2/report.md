#### Lab2 report

###### 练习0:填写已有实验

###### 本实验依赖实验1。请把你做的实验1的代码填入本实验中代码中有“LAB1”的注释相应部分。提示:可采用diff和patch工具进行半自动的合并(merge),也可用一些图形化的比较/merge工具来手动合并,比如meld,eclipse中的diff/merge工具,understand中的diff/merge工具等。
***
代码量比较小,直接通过搜索YOUR CODE和lab1确认,手动移植填写,以后的lab如果代码量加大了再尝试用工具半自动merge.
***

###### 列出本实验对应的OS原理的知识点,并说明本实验中的实现部分如何对应和体现了原理中的基本概念和关键知识点。
***
1.了解如何发现系统中的物理内存;

2.了解如何建立对物理内存的初步管理;

3.了解页表相关的操作;
***

###### 练习1:实现first-fit连续物理内存分配算法
###### 在实现first fit内存分配算法的回收函数时,要考虑地址连续的空闲块之间的合并操作。提示:在建立空闲页块链表时,需要按照空闲页块起始地址来排序,形成一个有序的链表。可能会修改default_pmm.c中的default_init,default_init_memmap,default_alloc_pages,default_free_pages等相关函数。请仔细查看和理解default_pmm.c中的注释。
***
本题中完成了2种实现，第一种为按照注释的语义一步一步编写（对比后与答案类似），第二种则是不将所有的page都放入free_list中，仅仅将每一个空闲的block的第1个page放入。下面为有关的定义，以及实现：

page的定义：

	struct Page {
        int ref;                        // page frame's reference counter
        uint32_t flags;                 // array of flags that describe the status of the page frame
        unsigned int property;          // the num of free block, used in first fit pm manager
        list_entry_t page_link;         // free list link
	};
以及

    /* Flags describing the status of a page frame */
	#define PG_reserved                 0       // if this bit=1: the Page is reserved for kernel, cannot be used in alloc/free_pages; otherwise, this bit=0 
	#define PG_property                 1       // if this bit=1: the Page is the head page of a free memory block(contains some continuous_addrress pages), and can be used in alloc_pages; if this bit=0: if the Page is the the head page of a free memory block, then this Page and the memory block is alloced. Or this Page isn't the head page.

在写第一种的时候总是弄混2个**property**，上述代码中可以知道**PG_property**(page property)和struct Page中的**property**的区别。个人感觉命名上感觉有些乱，注释的语言也比较难读懂。前者代表Page的一个Flag，意义根据注释为，空闲block第一页时为1，其他情况全部是0；后者代表block中页数。

**default_init_memmap**的实现：按照注释说明的实现会将**空闲block**中的每一页都放入free_list中，事实上，这和代码中关于**PG_property**的定义是矛盾的，也就是说，**答案的实现其实是不正确的**：答案中（也是我的第一种实现）通过将每一个空闲页（**包括空闲block中的其他页**）的**PG_property**都设置为1才能通过**assert(PageProperty(p))**。这里的实现除了常规的设置各个标志以外，仅仅将空闲block的**第一页**放入free_list。**这里设置PG_reserved说明了该过程是在内核态进行的。**

前一种实现中**default_alloc_pages**中的思路主要是遍历找到p->property >= n后，需要将所有需要的页都从列表中删除，并设置新的头。在这里改进的实现，仅仅**通过一次删除，一次插入**完成。

**default_free_pages**在第一次做的时候，设置flag上感觉和答案不太一样,觉得手册上有些地方好像矛盾。在仔细阅读后改进了代码。在搜索的过程中不是像答案中**while到末尾，过程中合并**，而是**首先确定列表中的位置，然后在前后考虑合并**。这里的**assert**开始时改变了原代码的**!PageReserved(p)**，发现没有通过后边的测试。结论是**这里free的来源是从用户**。
***

###### 你的first fit算法是否有进一步的改进空间
***
第二次写的代码和第一遍按照答案的来说有相当大的改进：1.不将所有空闲页放入free_list，而是选择block的第一页；2.删除合并的时候一步定位。
***

###### 练习2:实现寻找虚拟地址对应的页表项

###### 设计实现过程
***
第一步定位pdt后通过la中的PDX偏移找到对应的PDE

    pde_t *pdep = pgdir + PDX(la);  		 // (1) find page directory entry
第二步,如果Flag中的Present为0或者pdep为空指针,即对应page table不存在,需要进行相应的分配

    pte_t *ptep = NULL;
    if (!(*pdep & PTE_P)) {         		 // (2) check if entry is not present
第三步,如果传入create是FALSE,那么不需要create,此时直接返回NULL;然后通过alloc创建page,排除失败的情况返回NULL;

    if(!create)	return ptep;		 // (3) check if creating is needed, then alloc page for page table
    struct Page *page = alloc_page();	 // CAUTION: this page is used for page table, not for common data page
    if(!page) return ptep;
第四步,根据题设,需要设置ref为1表示页面状态(**表示该page有被进程引用**)

    set_page_ref(page,1);			 // (4) set page reference
第五步,调用page2pa得到page相应的**物理**地址，

    uintptr_t pa = page2pa(page);		 // (5) get linear address of page
第六步,将新分配的page清空内容

    memset(KADDR(pa),0,PGSIZE);              // (6) clear page content using memset
第七步,在pa得到的线性地址传给pde,并且设定权限

    *pdep = pa | PTE_U | PTE_W | PTE_P;  	 // (7) set page directory entry's permission
		}
第八步,通过映射得到的pte基址,然后加上偏移得到pte,进行返回

    ptep = KADDR(PDE_ADDR(*pdep)) + PTX(la);
    return ptep;					 // (8) return page table entry
过程中由于疏忽遇到了2个小问题,第七步的时候pdep忘了pdep是pde pointer,然后除错的时候报的是
	pa2page called with invalid pa
的错误,backtrace以后发现是insert的时候得到值不对,最后映射的地址不合法.调试时发现在*pdep的结果不同，注意到疏漏。
还有一个小问题出现在第八步里面的pte pointer的值：

	assertion failed: get_pte(boot_pgdir, PGSIZE, 0) == ptep
总结原因是是由于KAADR返回void指针,加上的偏移大小倍数不对.通过先赋值强制转换再加上偏移后结果就对了。
***
###### 如何ucore执行过程中访问内存,出现了页访问异常,请问硬件要做哪些事情?
***
首先是lab1中断相关,得到相应的中断,准备好中断栈,如果需要则设置好权限,

    #define T_PGFLT                 14  // page fault
进入中断执行段.如果再次缺失,则会发出double fault。否则，顺利时会将需要的页换进内存(调用相关算法后),调用page_remove,page_insert(会继续调用get_pte),然后返回新的页.
***
###### 如何ucore的缺页服务例程在执行过程中访问内存,出现了页访问异常,请问硬件要做哪些事情?
***
会出现
	#define T_DBLFLT                8   // double fault
的异常处理后退出系统。
***
###### 练习3:释放某虚地址所在的页并取消对应二级页表项的映射

###### 设计实现过程
***
(1)和练习2一样.

    if (*ptep & PTE_P) {                          //(1) check if this page table entry is present
(2)找到对应page.提示中的相关函数已经实现好.

    struct Page *page = pte2page(*ptep);  //(2) find corresponding page to pte
(3)减少ref.提示中的相关函数已经实现好.

    page_ref_dec(page);                   //(3) decrease page reference
(4)如果ref为0,freepage.提示中的相关函数已经实现好。（**经过这一步加深理解了映射机制：有些页可以被不同进程的不同页表，即不同的虚拟地址映射到**）

	if(!page_ref(page)) free_page(page);  //(4) and free this page when page reference reachs 0
(5)清空pte表项.

	*ptep = 0;                            //(5) clear second page table entry
(6)tlb中删除相关表项.提示中的相关函数已经实现好。（**虽然根据参数容易填好，但结果也提示了tlb根据页目录创建快表，为线性地址作映射**）

	tlb_invalidate(pgdir, la);            //(6) flush tlb
***

###### 数据结构Page的全局变量(其实是一个数组)的每一项与页表中的页目录项和页表项有无对应关系?如果有,其对应关系是啥?
***
指的应该是

    structPage *pages
有,在page_init函数中有对pages频繁操作,甚至还进行遍历(npage)的过程.而

	npage = maxpa / PGSIZE;
    pages = (struct Page *)ROUNDUP((void *)end, PGSIZE);
npage是与最大的物理内存相关,ends是bootloader结束标志,通过ROUNDUP找到第一张可以使用的页,

	for (i = 0; i < npage; i ++) {
        SetPageReserved(pages + i);
    }
对其进行标志Reserved,说明是给内核态使用的空间.而,

	uintptr_t freemem = PADDR((uintptr_t)pages + sizeof(struct Page) * npage);
之上的为空闲页.终上可以知道,pages指向的是可管理的物理内存空间的起始页,和pde,pte的关系就是,通过查询pde,pte之后再加上偏移得到是KADDR,即虚地址物理地址减去0xC0000000.
***
###### 如果希望虚拟地址与物理地址相等,则需要如何修改lab2,完成此事?	鼓励通过编程来具体完成这个问题
***
可以在pmm.h中发现两个转换的定义分别为

    #define PADDR(kva) ({                                                   \
        uintptr_t __m_kva = (uintptr_t)(kva);                       \
        if (__m_kva < KERNBASE) {                                   \
            panic("PADDR called with invalid kva %08lx", __m_kva);  \
        }                                                           \
        __m_kva - KERNBASE;                                         \
    })
    #define KADDR(pa) ({                                                    \
        uintptr_t __m_pa = (pa);                                    \
        size_t __m_ppn = PPN(__m_pa);                               \
        if (__m_ppn >= npage) {                                     \
            panic("KADDR called with invalid pa %08lx", __m_pa);    \
        }                                                           \
        (void *) (__m_pa + KERNBASE);                               \
    })
只需要分别改变最后返回的值去掉KERNBASE,不再进行+/-KERNBASE就可以得到相等的pa和kva.
***
