# Lab1 erport

## 练习1:理解通过make生成执行文件的过程。(要求在报告中写出对下述问题的回答)
## 列出本实验各练习中对应的OS原理的知识点,并说明本实验中的实现部分如何对应和体现了原理中的基本概念和关键知识点。
## 在此练习中,大家需要通过静态分析代码来了解:

### 1.1:操作系统镜像文件ucore.img是如何一步一步生成的?(需要比较详细地解释Makefile中每一条相关命令和命令参数的含
义,以及说明命令导致的结果)

```
	关于makefile粗略看了
		http://www.cs.colby.edu/maxwell/courses/tutorials/maketutor/ ；
		http://www.cnblogs.com/hnrainll/archive/2011/04/12/2013377.html ；
		http://www.gnu.org/software/make/manual/make.html
	内容，有了粗略的了解.
	makefile开头对几个macro进行定义，使用了':='(相对于'=',':='单次替换(expand)比较安全).
	之后对GCCPREFIX和QEMU的定义则相对复杂,下文中对应的标识符用 :=$(...)中的内容代替.其中一些标识的意义(piazza中一部分以及自己整理的一小部分),
		-ggdb 生成可供gdb使用的调试信息。这样才能用qemu+gdb来调试bootloader or ucore
		-m32 生成适用于32位环境的代码。我们用的模拟硬件是32bit的80386，所以ucore也要是32位的软件
		-gstabs 生成stabs格式的调试信息。这样要ucore的monitor可以显示出便于开发者阅读的函数调用栈信息
		-nostdinc 不使用标准库。标准库是给应用程序用的，我们是编译ucore内核，OS内核是提供服务的，所以所有的服务要自给自足
		-fno-stack-protector 不生成用于检测缓冲区溢出的代码。这是for 应用程序的，我们是编译内核，ucore内核好像还用不到此功能
		-Os 为减小代码大小而进行优化。根据硬件spec，主引导扇区只有512字节，我们写的简单bootloader的最终大小不能大于510字节。
		(s对应优化等级)
		-I 添加搜索头文件的路径
		(默认的是当前路径)
		i386-elf(Executable and Linkable Format) 表示硬件相关以及文件系统
		-objdump 表示显示object相关信息
		-grep 是 globally search a regular expression and print 的缩写
		-T <scriptfile>  让连接器使用指定的脚本
	一些makefile中的用法:
		'$' 用于展开选定的内容;
		'|' 管道符号表示前面的结果作为后面的输入(grep前表示在前面部分中搜索);
		'>' 表示重定向 '2>&1'的用法表示 将stderr(2)重定向道stdout(1)中去,&表示区分'1'是stdout而不是文件名"1";
		echo 是输出命令;
		其他的和其他语言类似.
	.SUFFIXES 中强制规定了文件类型集合;.DELETE_ON_ERROR为除错删除提示;
	接下来定义了HOSTCC 和 CC 编译器的相关属性,理解得不是很深入;
	接下来是kernel所需要的一些库(使用了kern文件夹中的大部分文件):
		INCLUDE	+= libs/

		CFLAGS	+= $(addprefix -I,$(INCLUDE))

		LIBDIR	+= libs

		$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)

		# -------------------------------------------------------------------
		# kernel

		KINCLUDE	+= kern/debug/ \
				kern/driver/ \
				kern/trap/ \
				kern/mm/

		KSRCDIR		+= kern/init \
				kern/libs \
				kern/debug \
				kern/driver \
				kern/trap \
				kern/mm

		KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

		$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

		KOBJS	= $(call read_packet,kernel libs)
	然后是kernel的具体生成代码:
		kernel = $(call totarget,kernel)

		$(kernel): tools/kernel.ld

		$(kernel): $(KOBJS)
			@echo + ld $@
			$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
			@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
			@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

		$(call create_target,kernel)
	根据makefile的语法格式,需要的文件在':'右边,之后用一个tab分开为执行代码;需要的文件为kernel.ld和KOBJS;而KOBJS根据上面的部分可以知道,read_packet,read_packet是lab1/tools/function.mk中定义的函数,访问的目录KSRCDIR中kern下的 init,libs,debug,driver,trap,mm以及INCLUDE中的libs,根据对应的.c和.s文件,可以知道还需要的文件为 init.o,readline.o,stdio.o,kdebug.o,kmonitor.o,panic.o,clock.o,console.o,intr.o,picirq.o,trap.o,trapentry.o,vectors.o,pmm.o,printfmt.o,string.o
		$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
	为.o的生成指令.
		bootfiles = $(call listf_cc,boot)
		$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

		bootblock = $(call totarget,bootblock)

		$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
			@echo + ld $@
			$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
			@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
			@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
			@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
			@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

		$(call create_target,bootblock)
	为bootblock的生成代码,bootfiles中说明了要使用的文件在boot目录下,为bootasm.o和bootmain.o;"$(bootblock):"中有说明了totarget还需要sign,而sign的生成代码为:
		$(call add_files_host,tools/sign.c,sign,sign)
		$(call create_target_host,sign,sign)
	最后,生成ucore.img的代码为:
		UCOREIMG	:= $(call totarget,ucore.img)

		$(UCOREIMG): $(kernel) $(bootblock)
			$(V)dd if=/dev/zero of=$@ count=10000
			$(V)dd if=$(bootblock) of=$@ conv=notrunc
			$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

		$(call create_target,ucore.img)
	根据第三行"$(UCOREIMG): $(kernel) $(bootblock)"可以知道需要前面生成的kernel和bootblock来生成ucore.img.dd指令用于convert and copy files,即对文件和硬盘进行读写等操作,if,of分别代表input file和output file; $@表示引用,这里为ucore.img;第一条指令表示向ucore.img中用0写入10000块,每块使用默认的512字节;第二条从bootblock中读入,并且往ucore.img写入;第三条则是从kernel读入,在seek1,即第1块(0为bootblock)中写入.
		
```

### 1.2:一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?
```
	继续1.1可以查询bootblock的生成过程是先生成2个bootasm.o和bootmain.o,通过call function.mk中的
		totarget = $(addprefix $(BINDIR)$(SLASH),$(1))
	然后又call outfile,通过调用sign的函数,将之前生成好的代码(文件句柄作为sign的第一个参数)写入到sign的buf中,然后通过buf输出(到第二个参数文件句柄)bootblock中. 
	而sign.c中
		char buf[512];
		buf[510] = 0x55;
		buf[511] = 0xAA;
	的代码显示MBR有512字节,并且最后2个字节分别为0x55和0xAA.
```

## 练习2:使用qemu执行并调试lab1中的软件。(要求在报告中简要写出练习过程)
## 了熟悉使用qemu和gdb进行的调试工作,我们进行如下的小练习:

### 2.1 从CPU加电后执行的第一条指令开始,单步跟踪BIOS的执行。
```
	初始的时候使用了手册中说的make lab1-mon,经过查询:
		lab1-mon: $(UCOREIMG)
			$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -monitor stdio -hda $< -serial null"
			$(V)sleep 2
			$(V)$(TERMINAL) -e "gdb -q -x tools/lab1init"
	但是出现了localhost not found的错误,后来突然就好了,也不知道怎么回事.
	出了错以后,在piazza助教提示下修改
		lab1/tools/gdbinit,
			set architecture i8086
			target remote :1234
		make debug
	后进入到了非常清晰的一个类图形化的调试界面.还有一种实现方法是lab1答案中修改debug内容,经过测试一致.
	尝试单步调试的时候出现了问题:
	我设置了断点 
		break *0xFFFFFFF0
		break *0x7c00
		break kern_init 
	然后进入断点以后continue后出现
		Program received signal SIGTRAP, Trace/breakpoint trap
	应该是因为设置断点后产生了trap,按next或者step会出现
		Cannot find bounds of current function
	去掉断点以后,next和step依然如此,但是continue之后,直接跳到了下一个断点.
	在0xfff0处continue之后,到达0x7c00,一直si到达7ccc无法再si,通过continue到达kern_init的断点.
		
```
	
### 2.2 在初始化位置0x7c00设置实地址断点,测试断点正常。
```
	在gdbinit中添加
		break *0x7c00
	在0x7c00设置断点和
		x /10i $pc
	来查看.x为examine的gdb简写i代表instruction,10表示显示10条,目标是pc寄存器的值.结果make debug得到:
		
		Breakpoint 1, 0x00007c00 in ?? ()
		=> 	0x7c00:      cli    
			0x7c01:      cld    
			0x7c02:      xor    %eax,%eax
			0x7c04:      mov    %eax,%ds		
			0x7c06:      mov    %ax,%es
			0x7c08:      mov    %ax,%ss
			0x7c0a:      in     $0x64,%al
			0x7c0c:      test   $0x2,%al
			0x7c0e:      jne    0x7c0a
			0x7c10:      mov    $0xd1,%al
	与bootasm.s中前10条指令:
		cli                                             # Disable interrupts
		cld                                             # String operations increment

		# Set up the important data segment registers (DS, ES, SS).
		xorw %ax, %ax                                   # Segment number zero
		movw %ax, %ds                                   # -> Data Segment
		movw %ax, %es                                   # -> Extra Segment
		movw %ax, %ss                                   # -> Stack Segment

		# Enable A20:
		#  For backwards compatibility with the earliest PCs, physical
		#  address line 20 is tied low, so that addresses higher than
		#  1MB wrap around to zero by default. This code undoes this.
		seta20.1:
		inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
		testb $0x2, %al
		jnz seta20.1

		movb $0xd1, %al                                 # 0xd1 -> port 0x64
	语义上一致,在该处正常.

```

### 2.3	从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和bootblock.asm进行比较。
```
	在makefile中添加了debug-from-lab1-answer选项中,尝试了lab1答案中提供的QEMU调试选项
		-d in_asm -D $(BINDIR)/q.log
	结果在bin目录下的q.log中可以得到的结果与第一题一致(即与bootasm.s的代码语义上一致)
	
```

### 2.4	自己找一个bootloader或内核中的代码位置,设置断点并进行测试。
```
	在 7c1e设置断点,期待得到gdt初始化部分的代码,结果得到
		0x7c1e:      lgdtw  0x7c6c
		0x7c23:      mov    %cr0,%eax
	显然一致.
```

## 练习3:分析bootloader进入保护模式的过程。(要求在报告中写出分析)
## BIOS将通过读取硬盘主引导扇区到内存,并转跳到对应内存中的位置执行bootloader。请分析bootloader是如何完成从实模式进入保护模式的。

### 3.0 初始化
```
		# start address should be 0:7c00, in real mode, the beginning address of the running bootloader
		.globl start
		start:
		.code16                                             # Assemble for 16-bit mode
			cli                                             # Disable interrupts
			cld                                             # String operations increment

			# Set up the important data segment registers (DS, ES, SS).
			xorw %ax, %ax                                   # Segment number zero
			movw %ax, %ds                                   # -> Data Segment
			movw %ax, %es                                   # -> Extra Segment
			movw %ax, %ss                                   # -> Stack Segment
	由汇编代码和注释可以知道,start为全局符号,cli禁止中断,cld: "Clears the direction flag; affects no other flags or registers. Causes all subsequent string operations to increment the index registers, (E)SI and/or (E)DI, used during the operation" 初始化.并将几个段符号清零.

```

### 3.1 为何开启A20,以及如何开启A20
```
		# Enable A20:
		#  For backwards compatibility with the earliest PCs, physical
		#  address line 20 is tied low, so that addresses higher than
		#  1MB wrap around to zero by default. This code undoes this.
		seta20.1:
			inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
			testb $0x2, %al
			jnz seta20.1

			movb $0xd1, %al                                 # 0xd1 -> port 0x64
			outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

		seta20.2:
			inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
			testb $0x2, %al
			jnz seta20.2

			movb $0xdf, %al                                 # 0xdf -> port 0x60
			outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
	材料中指出,为了兼容之前的超过20位后回卷的机制,在初始启动的时候A20 GATE是开着的,代码将A20线置为高,使得32位地址线都可以正常使用.
```

### 3.2 如何初始化GDT表
```
		lgdt gdtdesc
	该指令将已经准备好的GDT载入.
```

### 3.3 如何使能和进入保护模式
```
			movl %cr0, %eax
			orl $CR0_PE_ON, %eax
			movl %eax, %cr0

		# Jump to next instruction, but in 32-bit code segment.
		# Switches processor into 32-bit mode.
			ljmp $PROT_MODE_CSEG, $protcseg

		.code32                                             # Assemble for 32-bit mode
		protcseg:
		# Set up the protected-mode data segment registers
			movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
			movw %ax, %ds                                   # -> DS: Data Segment
			movw %ax, %es                                   # -> ES: Extra Segment
			movw %ax, %fs                                   # -> FS
			movw %ax, %gs                                   # -> GS
			movw %ax, %ss                                   # -> SS: Stack Segment

		# Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
			movl $0x0, %ebp
			movl $start, %esp
			call bootmain
	前三句通过将CR0中的PE置为1开启保护模式;
	ljmp长跳道32位下的CS;
	下面6句用于长跳,重新设置各个段寄存器,从PROT_MODE_CSEG中得到.
	最后三局ebp置0,start(即7c00)放入新建立的栈,call bootmain执行bootmain的内容.		
```

### 3.4 保护措施
```
		spin:
			jmp spin
	bootmain出错调回的话,可以方便debug出地址.
```

## 练习4:分析bootloader加载ELF格式的OS的过程。
## 通过阅读bootmain.c,了解bootloader如何加载ELF文件。通过分析源代码和通过qemu来运行并调试bootloader&OS,

### 4.1 bootloader如何读取硬盘扇区的?
```
	和读取硬盘扇区相关代码.
	对大小进行相关定义:
		#define SECTSIZE        512
		#define ELFHDR          ((struct elfhdr *)0x10000)      // scratch space
	读取1F7,观察硬盘是否准备好:
		/* waitdisk - wait for disk ready */
		static void
		waitdisk(void) {
			while ((inb(0x1F7) & 0xC0) != 0x40)
			/* do nothing */;
		}
	readsect函数,0x1F2输出扇区号,0x1F3最低8位,0x1F4,接下来8位,0x1F5再接下来8位,0x1F6最高3位 '|1' 强制置1,高到低第四位置0,最后4位读取secno.0x1F7输出读取磁盘命令,然后通过insl读取到dst.
		/* readsect - read a single sector at @secno into @dst */
		static void
		readsect(void *dst, uint32_t secno) {
			// wait for disk to be ready
			waitdisk();

			outb(0x1F2, 1);                         // count = 1
			outb(0x1F3, secno & 0xFF);
			outb(0x1F4, (secno >> 8) & 0xFF);
			outb(0x1F5, (secno >> 16) & 0xFF);
			outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
			outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

			// wait for disk to be ready
			waitdisk();

			// read a sector
			insl(0x1F0, dst, SECTSIZE / 4);
		}
	查询实验手册发现:
		IO地址 功能
			0x1f0 读数据,当0x1f7不为忙状态时,可以读。
			0x1f2 要读写的扇区数,每次读写前,你需要表明你要读写几个扇区。最小是1个扇区
			0x1f3 如果是LBA模式,就是LBA参数的0-7位
			0x1f4 如果是LBA模式,就是LBA参数的8-15位
			0x1f5 如果是LBA模式,就是LBA参数的16-23位
			0x1f6 第0~3位:如果是LBA模式就是24-27位	第4位:为0主盘;为1从盘
			0x1f7 状态和命令寄存器。操作时先给命令,再读取,如果不是忙状态就从0x1f0端口读数据
	readseg对readsect进行包装后,读入虚拟地址,长度,以及偏移,形成一个方便调用的接口.
		/* *
		* readseg - read @count bytes at @offset from kernel into virtual address @va,
		* might copy more than asked.
		* */
		static void
		readseg(uintptr_t va, uint32_t count, uint32_t offset) {
			uintptr_t end_va = va + count;

			// round down to sector boundary
			va -= offset % SECTSIZE;

			// translate from bytes to sectors; kernel starts at sector 1
			uint32_t secno = (offset / SECTSIZE) + 1;

			// If this is too slow, we could read lots of sectors at a time.
			// We'd write more to memory than asked, but it doesn't matter --
			// we load in increasing order.
			for (; va < end_va; va += SECTSIZE, secno ++) {
				readsect((void *)va, secno);
			}
		}
	然后在bootmain中diaoyongreadseg进行读取扇区.
```
### 4.2 bootloader是如何加载ELF格式的OS? 
```
	读取ELF的e_magic,判断格式,除错跳转到bad标志.
		// is this a valid ELF?
		if (ELFHDR->e_magic != ELF_MAGIC) {
			goto bad;
		}
	对头部信息进行摘:
		struct proghdr *ph, *eph;
	
		// load each program segment (ignores ph flags)
		ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
		eph = ph + ELFHDR->e_phnum;
	将头部信息指示下的相应代码读入到目标位置:	
		for (; ph < eph; ph ++) {
			readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
		}
	载入内核入口:
		// call the entry point from the ELF header
		// note: does not return
		((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	输出相应错误信息:
		bad:
			outw(0x8A00, 0x8A00);
			outw(0x8A00, 0x8E00);

		/* do nothing */
		while (1);
	
```

## 练习5:实现函数调用堆栈跟踪函数
## 我们需要在lab1中完成kdebug.c中函数print_stackframe的实现,可以通过函数print_stackframe来跟踪函数调用堆栈中记录的返回地址。

### 5.1
```
	出现过的问题:
		1,由于C99标准,无法在for循环声明;
		2,使用指针的时候忘记了uint_32的指针+1已经是4,导致结果不一致(一样的错犯了2次!!!);
		3,在最后一步pop的时候不小心将指针当做数据,导致出错;
		4,和答案对比后发现了ebp!=0的条件,经过阅读,初始栈底设为0,确实应该增加这个条件;
		5,由于2,导致print_debuginfo中的一个判断条件错误,eip unknown;
	通过手动跟踪发现init.c中,kern_init()通过调用grade_backtrace()调用grade_backtrace0()调用grade_backtrace1()调用grade_backtrace2()调用mon_backtrace调用()print_stackframe(),发动的.
	过程为,调用的时候ebp指向栈最新的base,eip为指令指针,每次调用的时候将eip入栈(上一个入栈的eip为现在的ebp+4).args表示函数调用的参数,+0为栈底,+1为上一级的eip,+2之后的为(如果存在的话)的参数.
	通过输出结果可以看到确实和手动在eclipse中搜索函数调用过程一致,层层调用,并且可以清楚看到参数内容,kern_init上面的是0x7d68.根据代码分析,该处应该是bootmain,x /i 0x7d68 发现是 mov $0x8a00,%ax 与 bootmain.c中的代码,跳转后回来的是bad:outw(0x8A00, 0x8A00)相符合.
```

## 练习6:完善中断初始化和处理

### 6.1 中断向量表中一个表项占多少字节?其中哪几位代表中断处理代码的入口?
```
	和GDT中一样,每个表项占8个字节.2-3字节16位表示SS(段选择子),用来选择GDT或者LDT的段,而0-1字节和6-7字节拼成32位作为段中的offset得到线性地址.
```

### 6.2 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中,依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏,填充idt数组内容。注意除了系统调用中断(T_SYSCALL)以外,其它中断均使用中断门描述符,权限为内核态权限;而系统调用中断使用异常,权限为陷阱门描述符。每个中断的入口由tools/vectors.c生成,使用trap.c中声明的vectors数组即可。
```
	首先发现gatedesc类型和SETGATE所需要的GATE一样,idt中的项即为SETGATE的第一个参数;第二个参数是判断trap还是interrupt,刚开始没有考虑到0x80系统调用,看到piazza同学的发言,补上;下一个参数是Code segment selector,在老师提示下发现memlayout.h中
		#define GD_KTEXT    ((SEG_KTEXT) << 3)        // kernel text
	观察答案发现还是漏了T_SWITCH_TOK.
	
	
```

### 6.3 请编程完善trap.c中的中断处理函数trap,在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分,使操作系统每遇到100次时钟中断后,调用print_ticks子程序,向屏幕上打印一行文字”100 ticks”。
```
	比较容易,见代码.
```

## 练习7:扩展proj4,增加syscall功能,即增加一用户态函数(可执行一特定系统调用:获得时钟计数值),当内核初始完毕后,可从内核态返回到用户态的函数,而用户态的函数又通过系统调用得到内核态的服务

### 7.1 

```
	参考了答案(完全独立实现有困难,很多的定义不太清楚),顺着答案的意思做了.
	init.c中找到lab1_switch_test()函数,然后找到要实现的2个函数.
	(这块感觉可以想到)第一个是lab1_switch_to_user().从从ring0到ring3,栈中要多压入2,即ss和esp的位置,第一条指令为sub 8.
	第二条指令通过int发动中断,在trap中实现具体的改变.我觉得这块不看答案自己做的话可能就会从栈中一个一个读,对照表格进行位操作然后放回去.答案里的做法方便了很多.本来想试着这么写一边,但觉得太浪费时间,也害怕无意中为后面引入bug,就不没事找事了.
	然后下一步(假定在trap中已经完成了更改各个权限的操作)是从栈中恢复,继续程序.即,将esp拉倒ebp的地方,清空栈中相关项.
	lab1_switch_to_kernel()相似,由于第一步需要从tss中得到ss0和esp0,交给中断处理来解决.然后恢复栈.
	
	
	找到trapframe的定义:
		struct trapframe {
			struct pushregs tf_regs;
			uint16_t tf_gs;
			uint16_t tf_padding0;
			uint16_t tf_fs;
			uint16_t tf_padding1;
			uint16_t tf_es;
			uint16_t tf_padding2;
			uint16_t tf_ds;
			uint16_t tf_padding3;
			uint32_t tf_trapno;
			/* below here defined by x86 hardware */
			uint32_t tf_err;
			uintptr_t tf_eip;
			uint16_t tf_cs;
			uint16_t tf_padding4;
			uint32_t tf_eflags;
			/* below here only when crossing rings, such as from user to kernel */
			uintptr_t tf_esp;
			uint16_t tf_ss;
			uint16_t tf_padding5;
		} __attribute__((packed));
	是产生中断的时候的定义的栈有关的frame.参考答案中的实现复杂啰嗦,其实中断的处理在代码中的体现并不多,包括对数据的恢复和iret.事实上iret应该会读取返回地址,而esp很快也会被改写.只需要判断当前cs是否是在内核态,然后相应地赋值就可以了.
		if (tf->tf_cs != USER_CS) {
			tf -> tf_cs = USER_CS;
			tf -> tf_ds = USER_DS;
			tf -> tf_es = USER_DS;
			tf -> tf_ss = USER_DS;
			tf -> tf_gs = USER_DS;
			tf -> tf_fs = USER_DS;
			tf -> tf_eflags |= FL_IOPL_MASK;
		}
	同样,觉得result的做法稍微啰嗦,自己写的测试通过了,但是不太放心,保留着result的注释:
		if (tf->tf_cs != KERNEL_CS) {
			tf->tf_cs = KERNEL_CS;
			tf->tf_ds = tf->tf_es = KERNEL_DS;
			tf->tf_eflags &= ~FL_IOPL_MASK;
			struct trapframe  *switchu2k;
			memmove(tf-8, tf, sizeof(struct trapframe) - 8);
			//switchu2k = (struct trapframe *)(tf->tf_esp - (sizeof(struct trapframe) - 8));
			//memmove(switchu2k, tf, sizeof(struct trapframe) - 8);
			//*((uint32_t *)tf - 1) = (uint32_t)switchu2k;
		}

```
