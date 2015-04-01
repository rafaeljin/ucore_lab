# Lab1 erport

## 练习0:填写已有实验

### 本实验依赖实验1。请把你做的实验1的代码填入本实验中代码中有“LAB1”的注释相应部分。提示:可采用diff和patch工具进行半自动的合并(merge),也可用一些图形化的比较/merge工具来手动合并,比如meld,eclipse中的diff/merge工具,understand中的diff/merge工具等。
```
	代码量比较小,直接通过搜索YOUR CODE和lab1确认,手动移植填写,以后的lab如果代码量加大了再尝试用工具半自动merge.
```

### 列出本实验对应的OS原理的知识点,并说明本实验中的实现部分如何对应和体现了原理中的基本概念和关键知识点。
```
	了解如何发现系统中的物理内存;
	了解如何建立对物理内存的初步管理;
	了解页表相关的操作;
```

## 练习1:实现first-fit连续物理内存分配算法
## 在实现first fit内存分配算法的回收函数时,要考虑地址连续的空闲块之间的合并操作。提示:在建立空闲页块链表时,需要按照空闲页块起始地址来排序,形成一个有序的链表。可能会修改default_pmm.c中的default_init,default_init_memmap,default_alloc_pages,default_free_pages等相关函数。请仔细查看和理解default_pmm.c中的注释。

### 相关实现说明:
```
	default_init_memmap中flag中的第二位property和Page的property刚开始看的时候弄混了好多次,觉得这里的名字起得不太好,至少应该在相关函数以及变量中说明,不然给我们造成很大的不便;对该函数的更改是在文件中注释所说,将该空间的所有page(除了第一页)property设为0,第一页设为block中的页数;检查页是否是被保留;将flag置0后,标注flags中的property表明该页空闲;最后还有将所有页通过list_add_before(通过一直插入到free_list前面将页连接).

	default_alloc_pages中的思路主要是遍历找到p->property >= n,之后将这段里面的page按照要求设定2个flag,并逐一从链表中删除.按照题意一步一步做,并不难,但由于更改的时候忘记删除一条list_add语句,导致出bug调试很久才找到结果(通过和result对比,中间cprintf一些中间结果).
	
	default_free_pages在设置flag上感觉和答案不太一样,觉得手册上有些地方好像矛盾,flag中property究竟是不是kernel权限reserve(在alloc中提示将其设为1)?更新:老师的解释是"lab2目前还没有涉及到用户空间，都在内核空间。"和之前的猜想一致.整个过程比较顺利,指示碰到一次低级bug.
	
```
### 你的first fit算法是否有进一步的改进空间
```
	在找到对应的block分配出去时不需要遍历该list进行delete,但需要通过原子操作,相对而言使用定义好的几个list操作更加安全,同时,因为遍历该list改变page属性的步骤不可避免,所以相对开销并不大.
	
```

## 练习2:实现寻找虚拟地址对应的页表项

### 设计实现过程
```
	第一步定位pdt后通过la中的PDX偏移找到对应的PDE	
		pde_t *pdep = pgdir + PDX(la);  		 // (1) find page directory entry
	第二步,如果Flag中的Present为0或者pdep为空指针,即对应page table不存在,需要进行相应的分配
		pte_t *ptep = NULL;
		if (!(*pdep & PTE_P)) {         		 // (2) check if entry is not present
	第三步,如果传入create是FALSE,那么不需要create,此时直接返回NULL;然后通过alloc创建page,排除失败的情况返回NULL;	
			if(!create)	return ptep;		 // (3) check if creating is needed, then alloc page for page table
			struct Page *page = alloc_page();	 // CAUTION: this page is used for page table, not for common data page	
			if(!page) return ptep;
	第四步,根据题设,需要设置ref为1表示页面状态
			set_page_ref(page,1);			 // (4) set page reference
	第五步,调用page2pa得到page相应的线性地址
			uintptr_t pa = page2pa(page);		 // (5) get linear address of page
	第六步,将新分配的page清空内容
			memset(KADDR(pa),0,PGSIZE);              // (6) clear page content using memset
	第七步,在pa得到的线性地址传给pde,并且设定权限
			*pdep = pa | PTE_U | PTE_W | PTE_P;  	 // (7) set page directory entry's permission
		}
	第八步,通过映射得到的pte基址,然后加上偏移得到pte,进行返回
		ptep = KADDR(PDE_ADDR(*pdep));
		ptep += PTX(la);
		return ptep;					 // (8) return page table entry
	过程中由于疏忽遇到了2个小bug,第七步的石猴pdep忘了pdep是pde pointer,然后除错的时候报的是
		pa2page called with invalid pa
	的错误,backtrace以后发现是insert的时候得到值不对,最后映射的地址不合法.通过和答案对比,发现在cprintf出*pdep的结果不同,细看才发现漏了指针
	还有一个bug是在第八步里面的pte pointer的值
		assertion failed: get_pte(boot_pgdir, PGSIZE, 0) == ptep
	这次比较隐蔽,是由于KAADR返回void指针,加上的偏移大小倍数不对.通过先赋值强制转换再加上偏移后结果就对了.
```	
### 如何ucore执行过程中访问内存,出现了页访问异常,请问硬件要做哪些事情?
```
	首先是lab1中断相关,得到相应的中断(硬件自动搜索IDT),准备好中断栈,如果需要则设置好权限,
		#define T_PGFLT                 14  // page fault
	进入中断执行段.如果再次缺失,则会发出double fault.如果顺利会将需要的页换进内存(调用相关算法后),调用page_remove,page_insert(会继续调用get_pte),然后返回新的页.
```
### 如何ucore的缺页服务例程在执行过程中访问内存,出现了页访问异常,请问硬件要做哪些事情?
```
	会出现
		#define T_DBLFLT                8   // double fault
	的异常后退出系统.
```
## 练习3:释放某虚地址所在的页并取消对应二级页表项的映射

### 设计实现过程
```
	(1)和练习2一样.
		if (*ptep & PTE_P) {                          //(1) check if this page table entry is present
	(2)找到对应page.提示中的相关函数已经实现好.
			struct Page *page = pte2page(*ptep);  //(2) find corresponding page to pte
	(3)减少ref.提示中的相关函数已经实现好.
			page_ref_dec(page);                   //(3) decrease page reference
	(4)如果ref为0,freepage.提示中的相关函数已经实现好.
			if(!page_ref(page)) free_page(page);  //(4) and free this page when page reference reachs 0
	(5)清空pte表项.
			*ptep = 0;                            //(5) clear second page table entry
	(6)tlb中删除相关表项.提示中的相关函数已经实现好.
			tlb_invalidate(pgdir, la);            //(6) flush tlb
		}
```

### 数据结构Page的全局变量(其实是一个数组)的每一项与页表中的页目录项和页表项有无对应关系?如果有,其对应关系是啥?
```
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
```
### 如果希望虚拟地址与物理地址相等,则需要如何修改lab2,完成此事?	鼓励通过编程来具体完成这个问题
```
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
```
