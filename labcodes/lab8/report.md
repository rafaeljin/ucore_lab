#### Lab8 report

###### 练习0:填写已有实验
###### 本实验依赖实验1/2/3/4/5/6/7。请把你做的实验1/2/3/4/5/6/7的代码填入本实验中代码中
有“LAB1”/“LAB2”/“LAB3”/“LAB4”/“LAB5”/“LAB6”	/“LAB7”的注释相应部分。并确保编译通过。注意:为了能够正确执行lab8的测试应用程序,可能需对已完成的实验1/2/3/4/5/6/7的代码进行进一步改进。
***
**meld**对比填写后，**proc.h**根据提示做了一些小改变后运行大部分测试通过。
***
###### 练习1:	完成读文件操作的实现(需要编码)
###### 首先了解打开文件的处理流程,然后参考本实验后续的文件读写操作的过程分析,编写在sfs_inode.c中sfs_io_nolock读文件中数据的实现代码。
***
打开文件的执行流程：
1. 调用**open(“名称”，选项)**函数获得文件描述符，代表文件链表中的位置；
2. 从用户态的open()中会调用**sys_open()**系统调用进入**syscall(SYS_OPEN, ...)**,从而进入**内核态**；
3. 进入内核态之后通过中断处理历程，**sys_open()->sysfile_open()->file_open()**,在这里做一些预处理(准备文件抽象结构file和inode，一些错误检测等)；
4. 在file_open()中关键函数**vfs_open()**。这里vfs只是一个抽象的处理，但这里主要做了2件事：**vfs_lookup()和vop_open()**，即找到对应文件路径的inode和打开操作；
5. vfs_lookup()中通过**get_device()**获得根目录对应的inode，然后使用vop_lookup()获得文件对应的inode；
6. 到上面为止，分析到的都是**vop**(virtual operations),没有进入实际代码。定位到**sfs_inode.c**中末尾，类似之前schedule的方式，定义了实际的函数，即用sfs实体化了vfs。对应于目录和文件的不同操作，**vop_open():sfs_opendir和sfs_openfile, vop_lookup():sfs_lookup()**;
7. sfs_lookup中，**sfs_lookup_once->(sfs_dirent_search_nolock和sfs_load_inode)**， **sfs_load_inode->sfs_rbuf（磁盘块儿缓存区）->sfs_rwblock_nolock（读取块）->dop_io（dev.h中操作）**；

sfs_io_nololock实现：首先根据函数名，可以知道是SFS的io操作函数，nolock指明了是互斥函数，需要保护。需要用到的核心函数：**sfs_bmap_load_nolock**（用于获取io操作目标的inode）和**sfs_buf_op**，**sfs_block_op**（底层的读写操作，根据读或者写指向具体的函数）。具体实现：

	blkoff = offset % SFS_BLKSIZE;
    int n_io;
    while(endpos > alen + offset) {
    	size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset - alen);
    	if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
    		goto out;
        }
    	if(blkoff != 0 || nblks == 0){
    		n_io = 1;
    		if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0) {
    			goto out;
    		}
    		blkoff = 0;
    	}else{
    		n_io = nblks;
    		if ((ret = sfs_block_op(sfs, buf, ino, n_io)) != 0) {
    			goto out;
            }
    	}
    	alen += n_io*size,buf += n_io*size, blkno += n_io, nblks -= n_io;
    }
* blkoff：中记录**第一块中实际起始位置偏离块起始位置**；在读写第一块时可能不为0，**其他时候都为0**；
* size：记录实际读写块大小；在不完整的块（结尾，开头）为实际大小，中间部分为实际块大小；
* n_io：记录本次迭代写入的块数；首尾都为1，中间为块数；

实现思路：根据判定条件**endpos > alen + offset**判定**起始位置+已读取**是否小于结尾。,第一块不对齐，条件为**blkoff!=0**，末尾不对齐为**blkoff!=0&&nblks==0**，中间部分为**blkoff!=0&&nblks!=0**。根据条件选择2个低层io函数。

和参考答案的区别：
* 参考答案的实现头，中间和末尾分别处理，这里集中在了一起；
* block的读取时，一次读取n块，减少了函数嵌套；
***
###### 练习2：	完成基于文件系统的执行程序机制的实现(需要编码)
###### 改写proc.c中的load_icode函数和其他相关函数,实现基于文件系统的执行程序机制。执行:make qemu。如果能看看到sh用户程序的执行界面,则基本成功了。如果在sh用户界面上可以执行”ls”,”hello”等其他放置在sfs文件系统中的其他执行程序,则可以认为本实验基本成功。(使用的是qemu-1.0.1)。
***
alloc_proc()和do_fork()不需要做出太多改变，因为在前者是用memset初始化，随着proc_struct结构的变化也会初始化新的量（为0），而后者中调用一个**copy_files()**可以完成filesp的复制。

load_icode()函数对比lab7可以发现传入参数改变了，不再是二进制可执行文件，而是一个文件描述符fd。可以在lab7的基础上实现。步骤如下：
1. 为进程建立内存管理器mm，和lab7一样，可以调用**mm_create()**实现；
2. 建立页目录表，通过**setup_pgdir()**设定，和lab7一样；
3. 从硬盘上读取程序到内存并建立对应的虚拟内存映射表：**load_icode_read(fd,...)**可以实现从文件句柄所指向的位置读取内容。这一步和lab7不同，但只需将读取的位置从**binary**改为用load_icode_read();
4. 建立用户栈：和lab7一样，调用**mm_map()**，把相关参数放入，并作检查；
5. 设置页目录：和lab7一样，调用**lcr3()**;
6. 设置程序argc和argv：遍历argv，将argv加入栈顶；最后将argc加入。值得注意的地方：由于栈向下扩展，要先获得argv在栈中最低的位置，然后向后依次加入；每个argv结尾需要'\0'；
7. 设置进程的中断帧：和lab5类似，不同的是由于存在argc和argv栈顶发生变化，使用上一步中得到的结果代替lab7中的USTACKTOP；
8. 处理错误：和lab7一样。

和参考答案的区别：没有考虑关闭fd，以及开头的assert。实现上相似。
***
