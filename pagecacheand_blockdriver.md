# The page cache

When we tested mybrd driver, something interrupted IOs and driver couldn't get IOs from kernel directly.
It was the page cache.
The page cache acts like a buffer.
It intercepts IOs and stores data in memory.
When free memory becomes low, kernel releases data into driver and makes more free memory.

Roughly there are several layers between user allocation and device.
* app -> system call (user-level) | (kernel-level) filesystem -> page cache -> block layer -> driver -> device

Let's look into the page cache roughly.
The page cache is very complicated part of kernel and critical to overall performance.
So it's not easy to understand the page cache in detail.
I hope this chapter would help you understand the code of the page cache.

## test the page cache

Let's do some test to understand what the page cache is.

First please boot the kernel including mybrd.
And run "free -k" and "cat /proc/zoneinfo | grep file_pages" commands like following.

```
/ # cat /proc/zoneinfo | grep file_pages
    nr_file_pages 0
    nr_file_pages 966
/ # free -k
             total       used       free     shared    buffers     cached
Mem:        114160      21408      92752       3864          0       3864
-/+ buffers/cache:      17544      96616
Swap:            0          0          0

```

free command shows

* total: total memory
* used: used memory
* free: free memory
* shared: shared memory
* buffers: used memory for metadata of filesystem
* cached: used memory for file data of filesystem

I had not known exactly what is different between buffer and cached fields.
I found good description at:
http://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/5/html/Tuning_and_Optimizing_Red_Hat_Enterprise_Linux_for_Oracle_9i_and_10g_Databases/chap-Oracle_9i_and_10g_Tuning_Guide-Memory_Usage_and_Page_Cache.html

We are discussing file cache, so let's look into cached value.
Current cached value is 3864k because -k option prints information in kilobyte unit.

``/proc/zoneinfo`` file shows status of each zone.
There are many fields but we look at nr_file_pages field which shows the amount of the page cache of each zone.

Let's pass some data from mybrd disk to file.

```
/ # dd if=/dev/mybrd of=./big
[  120.253239] mybrd: start mybrd_make_request_fn: block_device=ffff880006194340 mybrd=ffff8800065ab240
[  120.253945] CPU: 1 PID: 1053 Comm: dd Not tainted 4.4.0+ #76
[  120.254408] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS Ubuntu-1.8.2-1ubuntu1 04/01/2014


------------------- skip

4194304 bytes (4.0MB) copied, 1.999599 seconds, 2.0MB/s
/ # 
/ # cat /proc/zoneinfo | grep file_pages
    nr_file_pages 140
    nr_file_pages 1850
/ # free -k
             total       used       free     shared    buffers     cached
Mem:        114160      25572      88588       7960          0       7960
-/+ buffers/cache:      17612      96548
Swap:            0          0          0
```

dd command generates 4MB data because mybrd disk size is 4MB.
After executing dd, we can check cached value is increased by 4096K.
It means 4MB data is copied from the disk into memory.
The data in memory is the page cahe.

When we creates a file, we think a file is created in a disk.
But if a file is created in a disk whenever we creates a file, there should be so many IOs and disk has too much load.
Then system performance would be so poor because disk has the worst performance (even-if it is ssd).
But the page cache layer stores data into memory, not the disk.
So user application can response quickly and system throughput performance becomes better.
When system is idle, the page cache layer writes data into the disk.
User couldn't notice anything.

Let's check nr_file_pages field in zoneinfo.
My system has two zones, so there are two nr_file_pages fields.
Total increase of two nr_file_pages is 1024.
This field is in page unit, so 1024 means 4096MB.

Let's copy the file into another file.

```
/ # dd if=./big of=./big2

8192+0 records in
8192+0 records out
4194304 bytes (4.0MB) copied, 0.003766 seconds, 1.0GB/s
/ # cat /proc/zoneinfo | grep file_pages
    nr_file_pages 294
    nr_file_pages 2720
/ # free -k
             total       used       free     shared    buffers     cached
Mem:        114160      29512      84648      12056          0      12056
-/+ buffers/cache:      17456      96704
Swap:            0          0          0
```
A new 4MB file is created.
And the page cache is also increased by 4MB.

Now let's write the file data into mybrd disk.

```
/ # dd if=./big of=/dev/mybrd/ 

/ # free -k
             total       used       free     shared    buffers     cached
Mem:        114160      33788      80372      12056          0      12056
-/+ buffers/cache:      21732      92428
Swap:            0          0          0
```
Writing the page cache (file data) into disk does not release the page cache, so cached value is not changed.
But used value is increased by 4276KB, because mybrd driver allocates pages to store data and some memory to manage radix-tree, such as the tree node.

Even-if mybrd was a real disk, mybrd driver allocated memory to manage the disk and used value would be increased.
Why is the memory of driver counted as used and file data counted as cached?

File data can be released when system run out of memory because kernel can read the file later when user access it.
Original data of the file exists in the disk.
So we can memory for buffering of the file.
If kernel releases the file data, cached will be decreased and free will be increased.
Therefore file data is counted specially as cached.
The second line of the result of free command shows another used and free value after releasing cached memory.
(80372 + 12056 = 92428 and 33788 - 12056 = 21732)
So the second line shows the maximum free and minimum used memory size that system would be able to have.

Memory allocated by the driver will exist until the driver frees it.
Driver cannot release the memory temporarily because there is not any backup.
So driver memory is identified as used.

There are several properties for a page.
A page allocated by driver or kerne has unmovable property, while a page allocated for user process has movable property.
Compaction can move only movable pages.
Amounts of movable and unmovable page are shown in /proc/pagetypeinfo file as following.
Please refer other documents for the detail about movable page and compaction.

```
/ # cat /proc/pagetypeinfo 
Page block order: 9
Pages per block:  512

Free pages count per migrate type at order       0      1      2      3      4      5      6      7      8      9     10 
Node    0, zone      DMA, type    Unmovable     11      4      5      2      1      1      1      1      1      1      0 
Node    0, zone      DMA, type      Movable      0      2      3      2      3      3      3      1      0      0      1 
Node    0, zone      DMA, type  Reclaimable      5      1      7      3      1      0      0      1      1      1      0 
Node    0, zone      DMA, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0 
Node    0, zone    DMA32, type    Unmovable      1      2     11      6      6      2      2      2      2      1      0 
Node    0, zone    DMA32, type      Movable     12     11      7      5      7      5     10      4      2      2     13 
Node    0, zone    DMA32, type  Reclaimable      8      5     20     15      2      1      0      0      0      1      0 
Node    0, zone    DMA32, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0 

Number of blocks type     Unmovable      Movable  Reclaimable   HighAtomic 
Node 0, zone      DMA            3            3            2            0 
Node 0, zone    DMA32           10           42            4            0 
```

## struct address_space for the page cache management

Page cache is represented by struct address_space.
The object of struct address_space is stored in i_mapping field of inode of device.
When driver adds a disk with add_disk() function, the disk is registered with device number.
(I'll skip creating inode. Roughly when mknod or event manager (for example, systemd) creates a device node for the disk, a inode for the disk is created. Please refer filesystem chapter of other books. Or check ext4_mknod() function.)
There is another important field for the page cache in inode that is i_fop field.
Default i_fop for the block device is def_blk_fops as following.

```
const struct file_operations def_blk_fops = {
    .open		= blkdev_open,
	.release	= blkdev_close,
	.llseek		= block_llseek,
	.read_iter	= blkdev_read_iter,
	.write_iter	= blkdev_write_iter,
	.mmap		= blkdev_mmap,
	.fsync		= blkdev_fsync,
	.unlocked_ioctl	= block_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl	= compat_blkdev_ioctl,
#endif
	.splice_read	= generic_file_splice_read,
	.splice_write	= iter_file_splice_write,
};
```

When a process opens a file, kernel copies i_fop into f_fop field of struct file and calls f_op->open.
So blkdev_open() is called when a block device file is opened.
In blkdev_open(), kernel copies i_mapping of inode into f_mapping of struct file.
Therefore every process can share the page cache.
For example, if one process reads sector 0~10 of disk A, data of sector 0~10 is copied into memory.
Later if another process reads sector 5~0 of disk A, kernel doesn't read the disk and only returns data in memory.

struct address_space에서 가장 중요한 필드는 struct address_space_operations 입니다.
fs/block_dev.c 파일을 열어보면 아래와같이 file_operations 타입의 객체와 address_space_operations 타입의 객체가 정의되어있습니다.

One of important fields of struct address_space is ``struct address_space_operations a_ops`` that has a set of operations for data transfer between the disk and the page cache.
Following is the default operation of block device.

```
static const struct address_space_operations def_blk_aops = {
    .readpage	= blkdev_readpage,
	.readpages	= blkdev_readpages,
	.writepage	= blkdev_writepage,
	.write_begin	= blkdev_write_begin,
	.write_end	= blkdev_write_end,
	.writepages	= generic_writepages,
	.releasepage	= blkdev_releasepage,
	.direct_IO	= blkdev_direct_IO,
	.is_dirty_writeback = buffer_check_dirty_writeback,
};
```

As I already decribed, open system-call creates a file object.
And it calls f_op->open that is def_blk_fops->open(=blkdev_open) that set file->f_mapping as inode->i_mapping.
Therefore file->f_mapping->a_ops is a pointer to def_blk_aops.

Later when the process reads or writes data into disk, read/write system-call calls f_op->read_iter and f_op->write_iter which read/write data in the page cache.
If the process does not specify the direct writing, blkdev_write_iter stores data in the page cache and terminates the system call.
So writing data into disk can be finished fast and the process can go ahead.

If the process read data, data should be ready immediately.
So blkdev_read_iter checks the page cache and calls file->f_mapping->a_ops(=blkdev_readpages) to receive data from the driver of the disk if data is not in the page cache.
If data is in the page cache, reading data of the process can be done fast without IO processing of slow disk.

If the process wants direct IO, blkdev_write_iter bypass the page cache and do data transfer immediately.

The operations such like blkdev_readpages and blkdev_writepages generate bio and pass the bio to the block layer of kernel.
bio has a pointer to struct block_device, so the block layer can find a queue to which bio should be added.

Please notice that it's not easy to understand the data flow from the user process to the virtual filesystem, page cache, block layer and driver.
Please read the code and refer https://www.thomas-krenn.com/en/wiki/Linux_Storage_Stack_Diagram for the overall picture.

### mapping and index fields of struct page

The mapping and index fields of struct page are used for the page cache.
The mapping field has the pointer to the struct address_space and the index has the offset in logical device.
Yes, we had already used the index field of struct page as the offset of the mybrd block device.
So page management of mybrd driver (originally brd driver) is very similar to the page cache.
There is a comment in the git log of brd.c that using index field of the page like the page cache was Linux torvalds' idea.

Actually there is one more part in the page cache, buffer-block and buffer-head.
I skip it because we only use block deivce file.
It would be better if you understand the page cache of block device first because it's simpler.
Then please refer to other books.

## check callstack from the VFS to driver

Let's check how mybrd_make_request_fn() is called.
We can use dump_stack() to print call-stack on terminal.
The call-stack should print functions of the virtual filesystem, the page cache and the block layer.

I changed the queue_mode to MYBRD_Q_BIO for simpler call-stack.

```
diff --git a/mybrd.c b/mybrd.c
index 11fb9af..cf8b717 100644
--- a/mybrd.c
+++ b/mybrd.c
@@ -62,7 +62,7 @@ struct mybrd_device {
 };
 
 
-static int queue_mode = MYBRD_Q_MQ;
+static int queue_mode = MYBRD_Q_BIO;
 static int mybrd_major;
 struct mybrd_device *global_mybrd;
 #define MYBRD_SIZE_4M 4*1024*1024
@@ -295,7 +295,7 @@ static blk_qc_t mybrd_make_request_fn(struct request_queue *q, struct bio *bio)
        pr_warn("start mybrd_make_request_fn: block_device=%p mybrd=%p\n",
                bdev, mybrd);
 
-       //dump_stack();
+       dump_stack();
        
        // print info of bio
        sector = bio->bi_iter.bi_sector;
```

Then boot the kernel and run dd.
Following is what I wrote data into mybrd disk in my virtual machine.

```
[  194.612304] Call Trace:
[  194.612474]  [<ffffffff8130dadf>] dump_stack+0x44/0x55
[  194.612833]  [<ffffffff814fac32>] mybrd_make_request_fn+0x42/0x260
[  194.613306]  [<ffffffff812efc8e>] generic_make_request+0xce/0x1a0
[  194.613761]  [<ffffffff812efdc2>] submit_bio+0x62/0x140
[  194.614158]  [<ffffffff8119fa78>] submit_bh_wbc.isra.38+0xf8/0x130
[  194.614571]  [<ffffffff811a188d>] __block_write_full_page.constprop.43+0x10d/0x3a0
[  194.615098]  [<ffffffff810ac8d5>] ? add_timer_on+0xd5/0x130
[  194.615523]  [<ffffffff811a1e50>] ? I_BDEV+0x10/0x10
[  194.615858]  [<ffffffff811a1e50>] ? I_BDEV+0x10/0x10
[  194.616238]  [<ffffffff811a1c60>] block_write_full_page+0x140/0x160
[  194.616654]  [<ffffffff811a2853>] blkdev_writepage+0x13/0x20
[  194.617107]  [<ffffffff8112344e>] __writepage+0xe/0x30
[  194.617518]  [<ffffffff81123e67>] write_cache_pages+0x1d7/0x4c0
[  194.617942]  [<ffffffff81194a30>] ? __mark_inode_dirty+0x2c0/0x310
[  194.618412]  [<ffffffff81123440>] ? domain_dirty_limits+0x120/0x120
[  194.618798]  [<ffffffff8111a4ee>] ? unlock_page+0x5e/0x60
[  194.619222]  [<ffffffff8112418c>] generic_writepages+0x3c/0x60
[  194.619616]  [<ffffffff81125ed9>] do_writepages+0x19/0x30
[  194.619972]  [<ffffffff8111b2ec>] __filemap_fdatawrite_range+0x6c/0x90
[  194.620456]  [<ffffffff8111b357>] filemap_write_and_wait+0x27/0x70
[  194.620923]  [<ffffffff811a29e9>] __blkdev_put+0x69/0x220
[  194.621325]  [<ffffffff811a3156>] ? blkdev_write_iter+0xd6/0x100
[  194.621723]  [<ffffffff811a2f97>] blkdev_put+0x47/0x100
[  194.622102]  [<ffffffff811a3070>] blkdev_close+0x20/0x30
[  194.622502]  [<ffffffff8116f7e7>] __fput+0xd7/0x1e0
[  194.622851]  [<ffffffff8116f929>] ____fput+0x9/0x10
[  194.623227]  [<ffffffff810702ae>] task_work_run+0x6e/0x90
[  194.623585]  [<ffffffff810021a2>] exit_to_usermode_loop+0x92/0xa0
[  194.624041]  [<ffffffff81002b2e>] syscall_return_slowpath+0x4e/0x60
[  194.624567]  [<ffffffff8188f30c>] int_ret_from_sys_call+0x25/0x8f
```

Following is the result of reading mybrd.
```
[  143.111883]  [<ffffffff8130dadf>] dump_stack+0x44/0x55
[  143.113572]  [<ffffffff814fac32>] mybrd_make_request_fn+0x42/0x260
[  143.115554]  [<ffffffff811654cf>] ? kmem_cache_alloc+0x2f/0x130
[  143.117391]  [<ffffffff812efc8e>] generic_make_request+0xce/0x1a0
[  143.118880]  [<ffffffff812efdc2>] submit_bio+0x62/0x140
[  143.120195]  [<ffffffff81127f49>] ? lru_cache_add+0x9/0x10
[  143.121662]  [<ffffffff811a8e85>] mpage_readpages+0x135/0x150
[  143.123364]  [<ffffffff811a1e50>] ? I_BDEV+0x10/0x10
[  143.124759]  [<ffffffff811a1e50>] ? I_BDEV+0x10/0x10
[  143.126213]  [<ffffffff8115f497>] ? alloc_pages_current+0x87/0x110
[  143.128042]  [<ffffffff811a2818>] blkdev_readpages+0x18/0x20
[  143.129704]  [<ffffffff81126493>] __do_page_cache_readahead+0x163/0x1f0
[  143.131482]  [<ffffffff811265eb>] ondemand_readahead+0xcb/0x250
[  143.132949]  [<ffffffff81126959>] page_cache_sync_readahead+0x29/0x40
[  143.134541]  [<ffffffff8111b89a>] generic_file_read_iter+0x46a/0x570
[  143.136190]  [<ffffffff811a31b0>] blkdev_read_iter+0x30/0x40
[  143.137665]  [<ffffffff8116d7d2>] __vfs_read+0xa2/0xd0
[  143.139124]  [<ffffffff8116dfe1>] vfs_read+0x81/0x130
[  143.140442]  [<ffffffff8116ec61>] SyS_read+0x41/0xa0
[  143.141714]  [<ffffffff8188f1ae>] entry_SYSCALL_64_fastpath+0x12/0x71
```

Please compare kernel code and callstack for yourself.
You can see some static function are omitted and some callback functions are a little bit difficult to trace.
For example, blkdev_read_iter() is not called directly by ``__vfs_read()``.
As following, ``__vfs_read`` checks file->f_ops->read and file->f_ops->read_iter.
```
ssize_t __vfs_read(struct file *file, char __user *buf, size_t count,
    	   loff_t *pos)
{
	if (file->f_op->read)
		return file->f_op->read(file, buf, count, pos);
	else if (file->f_op->read_iter)
		return new_sync_read(file, buf, count, pos);
	else
		return -EINVAL;
}
EXPORT_SYMBOL(__vfs_read);
```

We cannot find out which is called, file->f_ops->read or new_sync_read() with only reading ``__vfs_read``.

Kernel developers designes interface between each layer very carefully becasue each layer is developed by different group of developers.
And each group consts of many developers from different countries, different time zones and different languages.

Therefore there are so many callback functions in kernel sources.
Sometime it's very difficult to investigate some kernel source if you're not used it.
For examle, many filesystems registeres its own file-operation set.
We cannot find out what operation is called only with seeing kernel code.

Fortunately we already know file->f_ops stores def_blk_fops.
So we can investigate call-stack like following.

For reading:
```
READ: Sys_read
--> __vfs_read
(--> new_sync_read)
--> def_blk_fops.read_iter = blkdev_read_iter
--> generic_file_read_iter
(--> do_generic_file_read)
--> (find_get_page &)page_cache_sync_readahead
--> ondemand_readahead
--> __do_page_cache_readahead
--> read_pages
--> mapping->a_ops->readpages = blkdev_readpages
--> mpage_readpages
--> submit_bio
--> generic_make_request
--> mybrd_make_request_fn
```

For writing:
```
WRITE: int_ret_from_sys_call
--> syscall_return_slowpath
--> exit_to_usermode_loop
--> task_work_run
--> __fput
--> blkdev_close
--> blkdev_put
--> __blkdev_put
--> filemape_write_and_wait
--> __filemap_fdatawrite_range
--> do_writepages
--> generic_writepages
--> write_cache_pages
--> __writepage
--> mapping->a_ops->writepage = blkdev_writepage
--> block_write_full_page
--> submit_bh_wbc
--> submit_bio
--> generic_make_request
--> mybrd_make_request_fn
```

The callstack of writing is very different to the callstack of reading.
The callstack of reading starts with a system-call Sys_read but writing starts with int_ret_from_syscall that is called at the end of system-call.

As I already described, reading data generates IOs and driver should handle IOs.
But when application writes data into a disk, data is not written into a disk but memory.
So driver is not called by write system-call.

Kernel has some policies when memory data should be written into a disk.
Those policies are beyond of this document.
Please refer to other books.

In this document, It would be enough if you understood that writing data is asynchronous.
And we need to seperate layers to two parts
1. write system-call ~ page-cache layer (synchronous to the system-call)
2. page-cache layer ~ block layer ~ driver

Fortunately we have the callstack for reading.
Functions for writing should be counter-part of functions in the callstack of reading.
For example, we can guess there is a counter-part of blkdev_read_iter that is blkdev_write_iter.
Finally I could find callstack like following for myself.

```
__vfs_write
--> new_sync_write
--> blkdev_write_iter
--> generic_perform_write 
--> a_ops->write_begin(&write_end) 
--> blkdev_write_begin(&blkdev_write_end) 
--> block_write_begin
```

Yes, the page cache layers calls the operation of the address_space, a_ops.
Now you know the flow of function.
You can start reading kernel code ;-)

## file operations and address_space operations of ext2

To understand the operation of block device, let's check the operations of ext2.

Let's find file operations of ext2 like following.
```
~/work/linux-torvalds $ /bin/grep file_operations fs/ext2/* -R
fs/ext2/dir.c:const struct file_operations ext2_dir_operations = {
fs/ext2/ext2.h:extern const struct file_operations ext2_dir_operations;
fs/ext2/ext2.h:extern const struct file_operations ext2_file_operations;
fs/ext2/file.c:const struct file_operations ext2_file_operations = {
fs/ext2/inode.c:    		inode->i_fop = &ext2_file_operations;
fs/ext2/inode.c:			inode->i_fop = &ext2_file_operations;
fs/ext2/namei.c:		inode->i_fop = &ext2_file_operations;
fs/ext2/namei.c:		inode->i_fop = &ext2_file_operations;
fs/ext2/namei.c:		inode->i_fop = &ext2_file_operations;
fs/ext2/namei.c:		inode->i_fop = &ext2_file_operations;
```

Yes, file.c has the definition of struct file_operations.

```
const struct file_operations ext2_file_operations = {
    .llseek		= generic_file_llseek,
	.read_iter	= generic_file_read_iter,
	.write_iter	= generic_file_write_iter,
	.unlocked_ioctl = ext2_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl	= ext2_compat_ioctl,
#endif
	.mmap		= ext2_file_mmap,
	.open		= dquot_file_open,
	.release	= ext2_release_file,
	.fsync		= ext2_fsync,
	.splice_read	= generic_file_splice_read,
	.splice_write	= iter_file_splice_write,
};
```
We already know read_iter and write_iter are called by system-call read and write
So let's trace generic_file_read_iter and generic_file_write_iter with source code.
After I investigate some source files, I found out following callstack.

```
generic_file_read_iter
-->do_generic_file_read
--> find_get_page
--> pagecache_get_page*

generic_file_write_iter
--> generic_perform_write 
--> a_ops->write_begin = ext2_write_begin 
--> block_write_begin 
--> grab_cache_page_write_begin 
--> pagecache_get_page*
```

And address_space_operations are defined in fs/ext2/inode.c like following because inode should have it.

```
const struct address_space_operations ext2_aops = {
    .readpage		= ext2_readpage,
	.readpages		= ext2_readpages,
	.writepage		= ext2_writepage,
	.write_begin		= ext2_write_begin,
	.write_end		= ext2_write_end,
	.bmap			= ext2_bmap,
	.direct_IO		= ext2_direct_IO,
	.writepages		= ext2_writepages,
	.migratepage		= buffer_migrate_page,
	.is_partially_uptodate	= block_is_partially_uptodate,
	.error_remove_page	= generic_error_remove_page,
};
```

There are duplicated functions in callstacks of mybrd driver and ext2 filesystem.
The duplicated functions consists of the virtual filesystem and block layer.

## do_generic_file_read: read data from mybrd device to the page cache

We know do_generic_file_read() is called for both of file reading and device reading.
I'll describe do_generic_file_read() briefly to understand data transfer from mybrd device to the page cache.

Please check kernel sources as read this document.

### arguments

Followings are arguments of do_generic_file_read().
* struct file *filep: pointer to a object of struct file. 
* loff_t *ppos: offset to start reading that can be set by lseek system-call
* struct iov_iter *iter: this is by file system, not the page cache
  * refer to https://lwn.net/Articles/625077/
* ssize_t written: amount of data already read before do_generic_file_read

### find_get_page (= pagecache_get_page)

find_get_page() is a wrapper of pagecache_get_page()

find_get_entry함수를 호출해서 패이지캐시에 이미 해당 offset이 있는지를 찾습니다. find_get_page에서 pagecache_get_page를 호출할 때 fgp_flags 값과 gfp_mask 값을 0으로 호출했으므로 결국 페이지캐시에 해당 offset이 없으면 null을 반환합니다.

만약 fgp_flags에 FGP_CREATE 플래스가 있었다면 페이지를 할당하고, 페이지를 lru리스트에 포함합니다.

find_get_entry를 잠깐 볼까요. 가장 먼저 radix_tree_lookup_slot함수를 호출해서 radix-tree의 트리에 저장된 page의 더블 포인터를 가져오고, radix_tree_deref_slot으로 더블포인터를 포인터로 바꾸고, page_cache_get_speculative로 페이지의 참조 카운터를 증가합니다.

mybrd 드라이버에서 radix-tree에 페이지를 추가할 때 radix_tree_lookup 함수 하나만 사용했었습니다. 그런데 왜 여기에서는 radix_tree_lookup_slot을 사용할까요? 그 이유는 page_cache_get_speculative 함수의 주석에 있습니다. "This is the interesting part of the lockless pagecache"라는 설명이 있습니다. 즉 페이지를 찾아보는데 페이지를 찾는 중간에 다른 쓰레드에서 페이지를 해지하거나 다른데 사용했다면, 다시 페이지를 찾습니다. 결국 rcu_read_lock()만으로 페이지캐시를 구현하게 됩니다.

사실 저도 왜 이 코드가 동작하는지 완벽하게 이해했다고 말할 수 없을것 같습니다. 정확한 설명은page_cache_get_speculative함수의 주석을 참고하시기 바랍니다. 어쨌든 제가 말씀드리고 싶은건 페이지캐시가 lockless로 구현됐다는 것입니다.
```
page_cache_sync_readahead (=ondemand_readahead=__do_page_cache_readahead)
```
mybrd의 콜스택을 보면 page_cache_sync_readahead가 호출됩니다. 페이지 캐시에 페이지가 없었다는 뜻입니다. page_cache_sync_readahead 코드를 보면 결국 __do_page_cache_readahead함수가 핵심입니다.

__do_page_cache_readahead 함수 인자중에 몇개의 페이지를 읽을지 nr_to_read 값이 있습니다. 최초로 읽을 offset부터 미리 여러개의 페이지를 읽어놓는 것입니다. 그럼 사용자가 파일을 계속 읽을 때마다 IO가 발생하지 않고 페이지캐시에서 바로 데이터를 가져갈 수 있겠지요. radix_tree_lookup으로 해당 위치의 데이터가 페이지캐시에 있나 확인하고 없으면 page_cache_alloc_readahead 함수로 페이지를 할당합니다. 그리고 각 페이지마다 page->index 필드에 offset을 씁니다.

그리고 read_pages 함수에서 이전에 할당한 페이지들에 블럭 장치의 데이터를 읽어옵니다. read_pages를 보면 mapping->a_ops->readpages와 mapping->a_ops->readpage를 호출합니다. mybrd는 블럭 장치이므로 def_blk_aops를 확인하면 어떤 함수가 호출될지 알 수 있습니다. def_blk_aops.readpages = blkdev_readpages 함수가 등록돼있으니 blkdev_readpages가 호출되겠네요.

blk_start/finish_plug 함수는 참고 자료를 확인하세요.
* http://nimhaplz.egloos.com/m/5598614
* http://studyfoss.egloos.com/5585801

이제 blkdef_readpages로 넘어왔습니다. 그리고 blkdev_readpages는 mpage_readpages를 호출합니다. mpage_readpages함수를 보면 주석이 매우 깁니다. 중요한 함수라는걸 알 수 있습니다. mpages_readpages 함수는 페이지를 lru에 추가하고, 임시로 사용할 bio를 생성해서 IO를 발생시킵니다. bio를 IO 스케줄러에 전달하는 함수가 바로 submit_bio함수입니다. bio를 생성하는 함수는 do_mpage_readpage이고, submit_bio를 호출하는 함수가 mpage_bio_submit입니다.

### add_to_page_cache_lru
페이지 하나를 페이지 캐시에도 넣고, lru 리스트에도 추가하는 함수입니다. 먼저 페이지를 lock합니다. 새로 할당된 페이지이므로 다른 쓰레드에서 사용할 염려가 없으므로 페이지가 잠겨있는지 확인할 필요가 없으니 __set_page_locked함수로 페이지를 잡급니다.

__add_to_page_cache_locked는 다음 순서로 동작합니다.
1. radix_tree_maybe_preload: radix_tree_preload와 같은 일을 하지만, 페이지 플래그에 따라 radix_tree_preload를 호출하지 않을 수도 있습니다.
1. page_cache_get: get_page와 같습니다. 페이지의 참조 카운터를 증가시킵니다.
1. page의 mapping, index 필드 설정
1. page_cache_tree_insert: 트리에 페이지를 넣는 함수인데 mapping->tree_lock을 잡고 있는 상태에서 왜 radix_tree_insert를 안쓰고 page_cache_tree_insert를 구현했는지를 잘 모르겠습니다. 어쨋든 page_cache_tree_insert 코드를 보면 radix_tree_insert와 유사합니다.
1. radix_tree_preload_end: radix_tree_preload를 사용했다면 radix_tree_preload_end를 꼭 호출해야합니다.
1. __inc_zone_page_state: 각 zone마다 몇개의 페이지가 있고 어떤 페이지들이 어떤 상태인지 /proc/zoneinfo 파일에 통계 정보를 가지고 있습니다. 이 통계 정보를 갱신하는 함수입니다. NR_FILE_PAGES는 해당 zone에서 몇개의 페이지가 페이지캐시로 사용되었는지를 알려주는 값입니다. /proc/zoneinfo 파일에서 nr_file_pages 값에 해당됩니다.

이제 페이지가 페이지캐시에 들어갔으니 lru_cache_add 함수로 페이지를 lru리스트에 추가합니다. lru_cache_add함수는 각 프로세별로 존재하는 lru_add_pvec 배열에 새로운 페이지를 추가합니다.

참고로 __add_to_page_cache_locked함수에서도 페이지의 참조 카운터를 증가시키고, lru_cache_add에서도 페이지의 참조 카운터를 증가시킵니다. 이 말은 lru 리스트와 페이지캐시가 별도로 동작한다는 것입니다. lru에서 빠진다고해도 페이지캐시에서 빠지는게 아니기 때문입니다.

### do_mpage_readpage
복잡한 함수입니다만 mpage_alloc를 호출해서 가장 핵심은 bio 객체를 만든다는 것만 알면 될것같습니다.

do_mpage_readpage인자를 보면
* struct bio *bio: 처음에는 NULL값입니다. mpage_alloc 함수로 새로 bio 객체를 만들으라는 의미입니다. 한번 bio를 만들고나면 다음 for 루프에서 계속 다음 페이지를 위한 정보를 추가합니다. 그래서 for 루프가 종료된 다음에는 모든 페이지의 IO를 위한 정보를 가지게됩니다.
* struct page *page: 새로 할당해서 페이지캐시와 lru 리스트에 추가된 페이지입니다. 장치로부터 데이터를 읽어서 이 페이지에 저장합니다.
* unsigned nr_pages: 남은 페이지 갯수
* sector_t *last_block_in_bio: bio에서 처리할 마지막 블럭의 섹터 번호
* struct buffer_head *map_bh: bio를 만들면서 생성된 버퍼 헤드
* unsigned long *first_logical_block: 첫번째 블럭 번호
* get_block_t get_block: 파일시스템에서 파일 오프셋을 실제 파일시스템의 블럭 번호로 바꿔서 bh->b_blocknr 필드에 저장하는 함수입니다. 파일이라는건 연속된 데이터이지만, 사실 파일이 디스크에 연속적으로 저장될 수는 없습니다. 디스크 여기저기에 데이터가 저장되고, 어떤 파일의 어떤 부분이 디스크의 어디에 저장되었는지를 관리하는게 파일시스템의 주된 역할입니다. 그러므로 파일시스템마다 다른 정보를 얻어올 수 있도록 함수포인터를 전달합니다. 블럭 장치의 경우 blkdev_get_block 함수 포인터를 전달합니다. 블럭 장치는 사실 파일시스템이 없고 디스크 전체가 연속된 데이터로 봅니다. 따라서 전달된 블럭 번호를 그대로 버퍼헤드에 기록합니다. ext2의 경우 ext2_get_block 함수가 사용되는데, 파일시스템의 슈퍼블럭을 읽는등 파일시스템 자체의 정보를 활용할 것입니다.
* gfp_t gfp: bio객체를 할당할 때 쓸 페이지 할당 플래그

처음 do_mpage_readpage가 호출될 때는 bio가 NULL이고 map_bh의 b_state, b_size 값들도 0이므로  mpage_alloc으로 bio를 할당합니다.

그 다음 bio_add_page가 호출되면서 bio의 bi_io_vec 필드에 새로운 페이지가 추가됩니다. 우리는 현재 블럭 장치의 페이지캐시를 만들고있으므로 블럭 크기가 곧 페이지 크기가 됩니다. 따라서 bi_io_vec에 추가될 IO 길이도 모두 4096이 됩니다. 한번에 한 페이지씩 읽는 것입니다. 드라이버에서 bio의 bi_io_vec 필드를 출력해봤을때 모두 길이가 4096인걸 확인했었습니다.

mpage_readpages에서 루프를 돌면서 다시 do_mpage_readpage를 호출했을 때도 페이지의 크기와 블럭의 크기가 같으므로 사실상 버퍼헤드를 수정할 일은 없습니다. 매번 bio_add_page가 호출되면서 bio에 새로운 페이지를 추가하는 일이 사실상 전부입니다.

참고로 mpage_alloc은 단순합니다. bio_alloc으로 bio를 만들고 꼭 필요한 필드를 셋팅합니다.
* bio_alloc: struct bio 객체 생성
* bio->bi_bdev: IO가 발생해야할 struct block_device 객체 포인터
* bio->bi_iter.bi_sector: 첫번째 섹터 번호

mpage_alloc은 첫번째 섹터 번호만 설정합니다. 추가 정보는 bio_add_page에서 추가합니다. bio_add_page 함수도 간단합니다. bio의 bi_io_vec 배열을 가져와서 bv_page, bv_len, bv_offset을 초기화하는데 블럭 장치는 한번에 한 페이지씩 읽으므로 bv_len은 항상 4096이되고 bv_offset은 0이 될 것입니다. bi_vcnt를 증가시켜서 bi_io_vec배열을 차례대로 초기화합니다.

### mpage_bio_submit

mpage_alloc으로 생성한 bio 객체를 submit_bio 함수에 전달합니다. 결국 submit_bio는 generic_make_request를 통해서 mybrd로 넘어갑니다. bio 처리가 끝나면 호출된 bio->bi_end_io 콜백함수는 mpage_end_io입니다. add_to_page_cache_lru에서 페이지를 잠궜으므로 mpage_end_io에서는 페이지 락을 풀고 페이지를 페이지의 데이터가 막 읽혀진 상태이니 uptodate 상태로 표시합니다. 그리고 다쓴 bio를 해지합니다.

### copy_page_to_iter

iter에는 유저 레벨의 버퍼에 대한 정보가 들어있습니다. page에 있는 데이터를 유저 레벨 버퍼로 복사합니다.
iov_iter_count에서 iter->count 필드가 0이되면 do_generic_file_read가 종료됩니다.

## block_write_begin: write data into the page cache

block_write_begin함수만 간략하게 분석해보겠습니다.

block_write_begin은 grab_cache_page_write_begin와 __block_write_begin로 이루어져있습니다. grab_cache_page_write_begin은 pagecache_get_page를 호출합니다.

pagecache_get_page는 이전에 find_get_page를 분석할 때 나온 함수입니다. 차이가 있다면 find_get_page에서는 fgp_flags가 0이고, gfp_mask가 0인데, grab_cache_page_write_begin에서는 fgp_flags와 gfp_mask에 값을 전달한다는 것입니다. find_get_page는 페이지캐시에 찾는 페이지가 없으면 NULL을 반환합니다. 하지만 grab_cache_page_write_begin은 pagecache_get_page에서 페이지를 할당해서 페이지캐시에 추가하기 때문에, 페이지 할당 플래그도 필요하고, 페이지를 할당하도록 FGP_CREAT 등의 플래그도 필요합니다.

pagecache_get_page가 하는 일은 간단히보면
* FGP_LOCK|FGP_ACCESSED|FGP_WRITE|FGP_CREAT 플래그를 받음
* FGP_LOCK: 페이지캐시에 페이지가 있으면 페이지 잠금
* FGP_CREATE: 페이지캐시에 페이지가 없으면 페이지 할당
 * 새로 할당된 페이지이므로 락이 필요없음
 * PG_referenced 플래그 셋팅
 * 페이지캐시에 추가하고 lru 리스트에도 추가

이제 페이지캐시에 페이지가 있는 상태에서 __block_write_begin이 호출됩니다.
* 페이지는 반드시 잠겨있어야합니다.
* create_page_buffers: 버퍼헤드를 하나 할당받아서 버퍼헤드에 페이지 포인터를 저장합니다. head는 페이지안에 저장된 첫번째 버퍼를 관리하는 버퍼헤드의 주소입니다. 아직 버퍼헤드에는 아무런 정보도 없습니다.
* blocksize는 블럭의 크기인데 mybrd는 장치파일이므로 페이지 크기가 됩니다.
* block은 index 값과 같습니다.
* get_block (블럭 장치의 경우 blkdev_get_block)을 호출해서 버퍼헤드에 디스크 블럭 번호를 씁니다.
* 페이지나 버퍼해드의 플래그들을 설정합니다. 

페이지 캐시에 데이터를 쓸 준비가 끝났지만, 페이지캐시의 데이터를 장치에 쓰지는 않습니다.

## 페이지캐시에 있는 페이지 해지
### /proc/sys/vm/drop_caches

/proc/ 디렉토리는 커널의 정보와 현재 실행중인 프로세스들의 정보가 있는 곳입니다. 이중에서 유명한게 페이지캐시들을 해지시켜서 가용 메모리를 확보하는 /proc/sys/vm/drop_caches가 있습니다. 이 파일이 어떻게 사용되는지를 알면 언제 어떻게 페이지캐시를 해지하는지를 알 수 있겠지요.

일단 /proc/sys/ 디렉토리를 만드는 코드는 kernel/sysctl.c 에 있는 sysctl_init 함수입니다. 이 함수에서 sysctl_base_table이라는 테이블을 사용하는데 이게 /proc/sys/ 디렉토리 밑에 생성할 디렉토리들의 테이블입니다. 그럼 우리는 vm이라는 디렉토리를 만드는 vm_table을 봐야겠네요.

vm_table이라는 테이블을 보면 /proc/sys/vm/ 디렉토리에 생성할 파일들의 이름과 속성, 그리고 처리 함수의 이름 등이 있습니다. 우리가 봐야할건 drop_caches 항목입니다.
```
    {
		.procname	= "drop_caches",
		.data		= &sysctl_drop_caches,
		.maxlen		= sizeof(int),
		.mode		= 0644,
		.proc_handler	= drop_caches_sysctl_handler,
		.extra1		= &one,
		.extra2		= &four,
	},
```
참고로 제가 어떻게 sysctl_base_table이라는 테이블이 존재하는지, vm_table이라는게 존재하는지 찾을 수 있었을까요? 이 강좌를 쓰기전에는 사실 어딘가 그런 테이블이 있겠지라고만 생각했었습니다. 당연히 어딘가에 "drop_caches"라는 파일 이름이 소스 파일에 써있을거라고만 생각했습니다. 파일 이름을 동적으로 만들지는 않을거니까요. 이렇게 뭔가 검색할 거리가 있으면 일단 grep으로 찾아보는 거지요. grep으로 검색하면 소스가 아닌 txt 파일이나 기타 임시 파일들도 검색합니다. 그래서 cscope나 global등의 태깅툴에서 소스에서만 검색하는 기능을 써는게 좋을때도 있습니다. 아래는 제가 검색해본 결과입니다.
```
$ grep -IR drop_caches *
Documentation/sysctl/vm.txt:- drop_caches
Documentation/sysctl/vm.txt:drop_caches
Documentation/sysctl/vm.txt:    echo 1 > /proc/sys/vm/drop_caches
Documentation/sysctl/vm.txt:	echo 2 > /proc/sys/vm/drop_caches
Documentation/sysctl/vm.txt:	echo 3 > /proc/sys/vm/drop_caches
Documentation/sysctl/vm.txt:`sync' prior to writing to /proc/sys/vm/drop_caches.  This will minimize the
Documentation/sysctl/vm.txt:	cat (1234): drop_caches: 3
Documentation/sysctl/vm.txt:with your system.  To disable them, echo 4 (bit 3) into drop_caches.
Documentation/cgroups/memory.txt:A sync followed by echo 1 > /proc/sys/vm/drop_caches will help get rid of
Documentation/cgroups/blkio-controller.txt:	echo 3 > /proc/sys/vm/drop_caches
drivers/gpu/drm/i915/i915_debugfs.c:i915_drop_caches_get(void *data, u64 *val)
drivers/gpu/drm/i915/i915_debugfs.c:i915_drop_caches_set(void *data, u64 val)
drivers/gpu/drm/i915/i915_debugfs.c:DEFINE_SIMPLE_ATTRIBUTE(i915_drop_caches_fops,
drivers/gpu/drm/i915/i915_debugfs.c:			i915_drop_caches_get, i915_drop_caches_set,
drivers/gpu/drm/i915/i915_debugfs.c:	{"i915_gem_drop_caches", &i915_drop_caches_fops},
fs/drop_caches.c:int sysctl_drop_caches;
fs/drop_caches.c:int drop_caches_sysctl_handler(struct ctl_table *table, int write,
fs/drop_caches.c:    	if (sysctl_drop_caches & 1) {
fs/drop_caches.c:		if (sysctl_drop_caches & 2) {
fs/drop_caches.c:			pr_info("%s (%d): drop_caches: %d\n",
fs/drop_caches.c:				sysctl_drop_caches);
fs/drop_caches.c:		stfu |= sysctl_drop_caches & 4;
fs/Makefile:obj-$(CONFIG_SYSCTL)		+= drop_caches.o
fs/btrfs/inode.c:	 * echo 2 > /proc/sys/vm/drop_caches   # evicts inode
include/linux/mm.h:extern int sysctl_drop_caches;
include/linux/mm.h:int drop_caches_sysctl_handler(struct ctl_table *, int,
kernel/futex.c:	 * prevents drop_caches from setting mapping to NULL beneath us.
kernel/sysctl.c:		.procname	= "drop_caches",
kernel/sysctl.c:		.data		= &sysctl_drop_caches,
kernel/sysctl.c:		.proc_handler	= drop_caches_sysctl_handler,
kernel/sysctl_binary.c:	{ CTL_INT,	VM_DROP_PAGECACHE,		"drop_caches" },
System.map:ffffffff811c40c0 T drop_caches_sysctl_handler
System.map:ffffffff8143cd30 t i915_drop_caches_get
System.map:ffffffff814437b0 t i915_drop_caches_fops_open
System.map:ffffffff81443ca0 t i915_drop_caches_set
System.map:ffffffff81a793a0 r i915_drop_caches_fops
System.map:ffffffff820fc864 B sysctl_drop_caches
tools/testing/selftests/vm/run_vmtests:		echo 3 > /proc/sys/vm/drop_caches
```
Document/sysctl/ 디렉토리에 관련된 문서가 있다는걸 찾을 수 있습니다. vm.txt를 읽어보면 사용법이나 구현에 대한 설명 등이 있을것 같습니다. 그 외에 drivers 디렉토리에 있는 파일들은 당연히 우리와 상관이 없겠지요. 우리는 커널이 제공하는 기능을 찾는 것이니까요. 그럼 fs/drop_caches.c나 kernel/sysctl.c 파일 등이 남는데요 이 파일들을에 drop_caches이라는 파일을 만드는 코드가 있나 봅니다. 당연히 create("drop_caches")라는 코드는 없을겁니다. 뭔가 파일 이름을 정의하는 데이터구조와 파일을 생성하는 코드가 나눠져있을 것입니다. 커널 개발자들은 작은것 하나라도 나중에 변경될 수 있는 것은 반드시 데이터로 분리합니다. 코드안에 create("drop_caches")라고 데이터를 박아넣지 않습니다. 그런 철학을 생각하면서 찾다보면 여러가지를 배울 수 있습니다.

어쨌든 이제 drop_caches_sysctl_handler이 호출하는 함수를 따라가보면서 페이지캐시와 관련된게 있는지 찾아보면 됩니다. 

Document/sysctl/vm.txt 파일을 보면 drop_caches 파일에 1을 쓰면 페이지캐시를 제거한다고 합니다. 따라서 iterate_supers(drop_pagaecache_sb, NULL) 코드가 우리가 찾는 코드입니다. 뭔가 이름도 페이지캐시와 관련이 있을것 같습니다. 이런식으로 약간은 추리를 하면서 코드를 추적할 필요도 있습니다.

###delete_from_page_cache

drop_caches_sysctl_handler를 분석하는걸 일일이 설명할 필요는 없을것 같습니다. 바로 페이지캐시의 페이지 하나를 해지를 처리하는 delete_from_page_cache를 간단히 보겠습니다.

크게 __delete_from_page_cache와 mapping->a_ops->freepage 두개의 함수 호출로 이루어져있습니다.

__delete_from_page_cache를 간략하게 보면
* 페이지는 이미 잠겨있는 상태여야합니다.
* page_cache_tree_delete: radix-tree에서 해당 페이지를 빼내야겠지요.
* page->mapping = NULL: adress_space 포인터는 이제 필요없으니 지웁니다.
* __dec_zone_page_state(page, NR_FILE_PAGES): /proc/zoneinfo의 통계 정보에서 페이지캐시의 크기를 줄입니다.

블럭 장치의 address_space_operations는 def_blk_aops 입니다. 확인해보면 freepage는 정의하지 않았네요. 페이지가 잠겨있고 radix-tree에서 빠졌으면, 다른 곳에서 페이지를 사용하지 않는다면 해지해도 됩니다. 따라서 마지막으로 페이지에 대한 참조를 줄이는 page_cache_release를 호출합니다. 참조카운터가 0이되면 자동으로 해지가 되서 버디리스트에 들어가겠네요. page_cache_get을 언제 호출했는지 기억이 나시나요?

### 페이지캐시에서 장치로 데이터 쓰기

delete_from_page_cache를 보면 페이지를 해지하기전에 페이지의 데이터를 장치로 보내는 부분이 없습니다. 페이지에 만약 새로운 데이터가 있고, 아직 장치에 쓰기 전이라면 페이지를 해지하기전에 데이터를 flush해야할텐데, 그건 어디에서 할까요?

이전에 mybrd의 mybrd_make_request_fn 함수에 dump_stack을 호출하도록 해서 콜스택을 확인했었습니다. 그때 어플의 데이터가 페이지캐시에 써질때와 페이지캐시에서 드라이버로 써질때가 다르다는걸 확인했었습니다. 또 방금 페이지캐시를 해지할때 페이지의 데이터를 장치에 쓰지 않는다는걸 알았습니다. 그러므로 뭔가 페이지의 데이터를 장치에 쓰는 별도의 루틴이 있다는 것을 알게됩니다. 

정확하게따지면 커널이 블럭 장치의 데이터를 flush하는 몇가지 시점이 있습니다. 그 시점들에 대한 설명은 약간 블럭 장치의 범위를 벋어나므로 블럭 장치의 flush에만 집중하기 위해서 fsync 시스템콜을 간단하게 알아보겠습니다.

약간 커널 소스를 검색하는 요령이 생기신 분들은 fsync로 검색하실 수도 있겠지만 저는 이게 시스템 콜인걸 알고있으므로 먼저 SYSCALL_DEFINE을 검색해보는것도 방법이라고 생각합니다.
```
$ grep SYSCALL_DEFINE * -IR | grep sync
arch/s390/kernel/compat_linux.c:COMPAT_SYSCALL_DEFINE6(s390_sync_file_range, int, fd, u32, offhigh, u32, offlow,
arch/tile/kernel/compat.c:COMPAT_SYSCALL_DEFINE6(sync_file_range2, int, fd, unsigned int, flags,
fs/sync.c:SYSCALL_DEFINE0(sync)
fs/sync.c:SYSCALL_DEFINE1(syncfs, int, fd)
fs/sync.c:SYSCALL_DEFINE1(fsync, unsigned int, fd)
fs/sync.c:SYSCALL_DEFINE1(fdatasync, unsigned int, fd)
fs/sync.c:SYSCALL_DEFINE4(sync_file_range, int, fd, loff_t, offset, loff_t, nbytes,
fs/sync.c:SYSCALL_DEFINE4(sync_file_range2, int, fd, unsigned int, flags,
mm/msync.c:SYSCALL_DEFINE3(msync, unsigned long, start, size_t, len, int, flags)
```
sync 시스템콜도 있지만 이건 시스템 전체의 데이터를 flush하므로 더 복잡할 것이니 fsync만 생각하겠습니다.

fsync는 바로 do_sync를 호출하네요. 계속 따라가다보면
```
vfs_fsync
vfs_fsync_range
file->f_ops->fsync = blkdev_fsync
filemap_write_and_wait_range & blkdev_issue_flush
```

이런 순서로 함수들이 나타납니다.

사실 filemap_write_and_wait_range는 이전에 콜스택을 확인할 때 나왔던 함수입니다. 그리고 blkdev_issue_flush는 아무런 정보도 없는 dummy bio를 만들고, request-queue를 비우는 WRITE_FLUSH 명령을 request-queue로 전달하는 매우 간단한 함수입니다. 그러니 filemap_write_and_wait_range가 페이지캐시와 관련이 있겠네요.

커널의 콜스택은 이미 이전에 확인했으니 가장 핵심적인 함수 __block_write_full_page만 간략하게 보겠습니다.

####__block_write_full_page

함수가 시작되면 우선 create_page_buffers 함수로 해당 페이지의 버퍼헤드를 찾습니다. 페이지하나에 여러개의 블럭이 있다면 버퍼헤드도 여러개이겠지만, 장치 파일의 페이지에는 하나의 버퍼헤드만 있을 것입니다.

어쨌든 버퍼헤드를 확인하고 버퍼헤드에 락을 거는 등의 사전 작업을 한 후 submit_bh_wbc를 호출합니다. submit_bh_wbc에서 bio를 할당하고 bio의 필드를 셋팅하고 submit_bio를 호출하는 익숙한 코드를 실행합니다.

struct writeback_control 타입의 wbc라는 객체가 있는데 이 객체는 페이지 캐시를 얼마나 완료했는지를 관리하는 객체입니다. 상황에 따라 특정 파일에 속한 페이지 캐시를 모두 없애야할 때도 있고, 약간의 페이지 캐시만 없애야할 때도 있습니다.

만약 동적으로 마운트된 장치가 umount될때라면 해당 블럭 장치의 페이지 캐시를 모두 없애야할 것입니다. 하지만 시스템에 가용 메모리가 부족한 상황일때, 모든 페이지 캐시를 없앤다면 순간적으로 시스템이 정지된것처럼 보일 수도 있고, 시스템의 성능이 순간적으로 낮아질 수 있습니다. 따라서 최대한 페이지 캐시를 유지하면서 약간의 페이지 캐시만을 없애서 필요한 메모리만 할당될 수 있도록 균형을 맞춰야합니다. 그럴때 얼마의 메모리를 확보해야하는지 등등의 정보를 전달하는게 wbc 객체입니다. fsync이외에도 메모리 할당이 실패하는 등 페이지캐시가 해지되는 경우는 많습니다만 결국엔 같은 함수로 처리될 것이고 wbc 객체의 정보만 달라질 것입니다.


# experiement

```
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <string.h>

int main(void)
{
    int fd;
    char buf[40960];
    ssize_t ret;

    fd = open("/dev/mybrd", O_RDWR);
    if (fd < 0) {
        perror("fail to open");
        return 1;
    }

    ret = read(fd, buf, 40960);
    printf("read %d-bytes\n", (int)ret);

    printf("\n\n------------------- ioctl -----------\n");
    ioctl(fd, 0x1234);
    printf("\n\n------------------- ioctl -----------\n\n");


    memset(buf, 0xa5, 40960);
    lseek(fd, 0, SEEK_SET);
    ret = write(fd, buf, 40960);
    printf("write %d-bytes\n", (int)ret);
    
    printf("\n\n------------------- ioctl -----------\n");
    ioctl(fd, 0x1234);
    printf("\n\n------------------- ioctl -----------\n\n");

    close(fd);

    return 0;
}
```

```
/ # ./a.out
[    5.076712] mybrd: start queue_rq: request-ffff88007f910000 priv-ffff88013a330f50 request->special=          (null)
[    5.077463] CPU: 0 PID: 1036 Comm: a.out Not tainted 4.4.0-eudyptula+ #157
[    5.077939] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS 1.10.1-1ubuntu1 04/01/2014
[    5.078517]  ffff88013a330f50 ffff88007f08bab0 ffffffff8130fec0 ffff88007f910000
[    5.079026]  ffff88007f08bad0 ffffffff814ff623 ffff88007f9a9400 ffff88007f08baf8
[    5.079536]  ffff88007f08bb58 ffffffff812fb989 ffff88007f890000 ffff88007f9a9408
[    5.080041] Call Trace:
[    5.080208]  [<ffffffff8130fec0>] dump_stack+0x44/0x64
[    5.080536]  [<ffffffff814ff623>] mybrd_queue_rq+0x43/0xb0
[    5.080886]  [<ffffffff812fb989>] __blk_mq_run_hw_queue+0x1b9/0x340
[    5.081287]  [<ffffffff812fb7b7>] blk_mq_run_hw_queue+0x87/0xa0
[    5.081666]  [<ffffffff812fcbd9>] blk_mq_insert_requests+0xe9/0x170
[    5.082064]  [<ffffffff812fd6c4>] blk_mq_flush_plug_list+0x114/0x130
[    5.082469]  [<ffffffff812f384c>] blk_flush_plug_list+0xcc/0x230
[    5.082907]  [<ffffffff812f3d07>] blk_finish_plug+0x27/0x40
[    5.083403]  [<ffffffff811272d3>] __do_page_cache_readahead+0x183/0x1f0
[    5.083860]  [<ffffffff81127409>] ondemand_readahead+0xc9/0x240
[    5.084252]  [<ffffffff81127769>] page_cache_sync_readahead+0x29/0x40
[    5.084659]  [<ffffffff8111c558>] generic_file_read_iter+0x468/0x570
[    5.085062]  [<ffffffff811a4b40>] blkdev_read_iter+0x30/0x40
[    5.085422]  [<ffffffff8116ebd4>] __vfs_read+0xa4/0xd0
[    5.085765]  [<ffffffff8116f3c1>] vfs_read+0x81/0x120
[    5.086084]  [<ffffffff81170021>] SyS_read+0x41/0xa0
[    5.086413]  [<ffffffff81895b2e>] entry_SYSCALL_64_fastpath+0x12/0x6d
[    5.086812] mybrd: queue-rq: req=ffff88007f910000 len=126976 rw=READ
[    5.087208] mybrd:     sector=0 bio-info: len=4096 p=ffffea0001fc3680 offset=0
[    5.087666] mybrd: lookup: page-          (null) index--1 sector-0
[    5.088136] mybrd: copy: ffff88007f0da000 <- 0 (4096-bytes)
[    5.088498] mybrd: 0 0 0 0 0 0 0 0
[    5.088727] mybrd:     sector=8 bio-info: len=4096 p=ffffea0001fc3640 offset=0
[    5.089207] mybrd: lookup: page-          (null) index--1 sector-8
[    5.089619] mybrd: copy: ffff88007f0d9000 <- 0 (4096-bytes)
[    5.089975] mybrd: 0 0 0 0 0 0 0 0
[    5.090199] mybrd:     sector=16 bio-info: len=4096 p=ffffea0001fc1140 offset=0
[    5.090666] mybrd: lookup: page-          (null) index--1 sector-16
[    5.091067] mybrd: copy: ffff88007f045000 <- 0 (4096-bytes)
[    5.091426] mybrd: 0 0 0 0 0 0 0 0
[    5.091647] mybrd:     sector=24 bio-info: len=4096 p=ffffea0001fc0e80 offset=0
[    5.092114] mybrd: lookup: page-          (null) index--1 sector-24
[    5.092512] mybrd: copy: ffff88007f03a000 <- 0 (4096-bytes)
[    5.092869] mybrd: 0 0 0 0 0 0 0 0
[    5.093089] mybrd:     sector=32 bio-info: len=4096 p=ffffea0001fc1380 offset=0
[    5.093556] mybrd: lookup: page-          (null) index--1 sector-32
[    5.093957] mybrd: copy: ffff88007f04e000 <- 0 (4096-bytes)
[    5.094315] mybrd: 0 0 0 0 0 0 0 0
[    5.094536] mybrd:     sector=40 bio-info: len=4096 p=ffffea0001fc1100 offset=0
[    5.095003] mybrd: lookup: page-          (null) index--1 sector-40
[    5.095418] mybrd: copy: ffff88007f044000 <- 0 (4096-bytes)
[    5.095813] mybrd: 0 0 0 0 0 0 0 0
[    5.096029] mybrd:     sector=48 bio-info: len=4096 p=ffffea0001fc2700 offset=0
[    5.096512] mybrd: lookup: page-          (null) index--1 sector-48
[    5.096901] mybrd: copy: ffff88007f09c000 <- 0 (4096-bytes)
[    5.097250] mybrd: 0 0 0 0 0 0 0 0
[    5.097465] mybrd:     sector=56 bio-info: len=4096 p=ffffea0001feec80 offset=0
[    5.097958] mybrd: lookup: page-          (null) index--1 sector-56
[    5.098369] mybrd: copy: ffff88007fbb2000 <- 0 (4096-bytes)
[    5.098763] mybrd: 0 0 0 0 0 0 0 0
[    5.098978] mybrd:     sector=64 bio-info: len=4096 p=ffffea0001feeac0 offset=0
[    5.099458] mybrd: lookup: page-          (null) index--1 sector-64
[    5.099849] mybrd: copy: ffff88007fbab000 <- 0 (4096-bytes)
[    5.100245] mybrd: 0 0 0 0 0 0 0 0
[    5.100479] mybrd:     sector=72 bio-info: len=4096 p=ffffea0001feed40 offset=0
[    5.100946] mybrd: lookup: page-          (null) index--1 sector-72
[    5.101349] mybrd: copy: ffff88007fbb5000 <- 0 (4096-bytes)
[    5.101711] mybrd: 0 0 0 0 0 0 0 0
[    5.101932] mybrd:     sector=80 bio-info: len=4096 p=ffffea0001feea00 offset=0
[    5.102397] mybrd: lookup: page-          (null) index--1 sector-80
[    5.102792] mybrd: copy: ffff88007fba8000 <- 0 (4096-bytes)
[    5.103194] mybrd: 0 0 0 0 0 0 0 0
[    5.103415] mybrd:     sector=88 bio-info: len=4096 p=ffffea0001fc0e00 offset=0
[    5.103881] mybrd: lookup: page-          (null) index--1 sector-88
[    5.104283] mybrd: copy: ffff88007f038000 <- 0 (4096-bytes)
[    5.104642] mybrd: 0 0 0 0 0 0 0 0
[    5.104864] mybrd:     sector=96 bio-info: len=4096 p=ffffea0001fc36c0 offset=0
[    5.105331] mybrd: lookup: page-          (null) index--1 sector-96
[    5.105734] mybrd: copy: ffff88007f0db000 <- 0 (4096-bytes)
[    5.106089] mybrd: 0 0 0 0 0 0 0 0
[    5.106320] mybrd:     sector=104 bio-info: len=4096 p=ffffea0001fc3600 offset=0
[    5.106814] mybrd: lookup: page-          (null) index--1 sector-104
[    5.107227] mybrd: copy: ffff88007f0d8000 <- 0 (4096-bytes)
[    5.107586] mybrd: 0 0 0 0 0 0 0 0
[    5.107809] mybrd:     sector=112 bio-info: len=4096 p=ffffea0001fc3700 offset=0
[    5.108287] mybrd: lookup: page-          (null) index--1 sector-112
[    5.108696] mybrd: copy: ffff88007f0dc000 <- 0 (4096-bytes)
[    5.109054] mybrd: 0 0 0 0 0 0 0 0
[    5.109279] mybrd:     sector=120 bio-info: len=4096 p=ffffea0001fc3780 offset=0
[    5.109755] mybrd: lookup: page-          (null) index--1 sector-120
[    5.110165] mybrd: copy: ffff88007f0de000 <- 0 (4096-bytes)
[    5.110523] mybrd: 0 0 0 0 0 0 0 0
[    5.110744] mybrd:     sector=128 bio-info: len=4096 p=ffffea0001fc37c0 offset=0
[    5.111217] mybrd: lookup: page-          (null) index--1 sector-128
[    5.111625] mybrd: copy: ffff88007f0df000 <- 0 (4096-bytes)
[    5.111986] mybrd: 0 0 0 0 0 0 0 0
[    5.112212] mybrd:     sector=136 bio-info: len=4096 p=ffffea0001fc3c80 offset=0
[    5.112686] mybrd: lookup: page-          (null) index--1 sector-136
[    5.113097] mybrd: copy: ffff88007f0f2000 <- 0 (4096-bytes)
[    5.113456] mybrd: 0 0 0 0 0 0 0 0
[    5.113686] mybrd:     sector=144 bio-info: len=4096 p=ffffea0001fc3cc0 offset=0
[    5.114160] mybrd: lookup: page-          (null) index--1 sector-144
[    5.114566] mybrd: copy: ffff88007f0f3000 <- 0 (4096-bytes)
[    5.114921] mybrd: 0 0 0 0 0 0 0 0
[    5.115144] mybrd:     sector=152 bio-info: len=4096 p=ffffea0001feeb00 offset=0
[    5.115617] mybrd: lookup: page-          (null) index--1 sector-152
[    5.116058] mybrd: copy: ffff88007fbac000 <- 0 (4096-bytes)
[    5.116451] mybrd: 0 0 0 0 0 0 0 0
[    5.116751] mybrd:     sector=160 bio-info: len=4096 p=ffffea0001feeb40 offset=0
[    5.117295] mybrd: lookup: page-          (null) index--1 sector-160
[    5.117746] mybrd: copy: ffff88007fbad000 <- 0 (4096-bytes)
[    5.118108] mybrd: 0 0 0 0 0 0 0 0
[    5.118330] mybrd:     sector=168 bio-info: len=4096 p=ffffea0001feeb80 offset=0
[    5.118802] mybrd: lookup: page-          (null) index--1 sector-168
[    5.119211] mybrd: copy: ffff88007fbae000 <- 0 (4096-bytes)
[    5.119568] mybrd: 0 0 0 0 0 0 0 0
[    5.119789] mybrd:     sector=176 bio-info: len=4096 p=ffffea0001feebc0 offset=0
[    5.120263] mybrd: lookup: page-          (null) index--1 sector-176
[    5.120670] mybrd: copy: ffff88007fbaf000 <- 0 (4096-bytes)
[    5.121026] mybrd: 0 0 0 0 0 0 0 0
[    5.121249] mybrd:     sector=184 bio-info: len=4096 p=ffffea0001fc3000 offset=0
[    5.121722] mybrd: lookup: page-          (null) index--1 sector-184
[    5.122128] mybrd: copy: ffff88007f0c0000 <- 0 (4096-bytes)
[    5.122484] mybrd: 0 0 0 0 0 0 0 0
[    5.122705] mybrd:     sector=192 bio-info: len=4096 p=ffffea0001fc3040 offset=0
[    5.123179] mybrd: lookup: page-          (null) index--1 sector-192
[    5.123585] mybrd: copy: ffff88007f0c1000 <- 0 (4096-bytes)
[    5.123942] mybrd: 0 0 0 0 0 0 0 0
[    5.124166] mybrd:     sector=200 bio-info: len=4096 p=ffffea0001fc3080 offset=0
[    5.124637] mybrd: lookup: page-          (null) index--1 sector-200
[    5.125042] mybrd: copy: ffff88007f0c2000 <- 0 (4096-bytes)
[    5.125399] mybrd: 0 0 0 0 0 0 0 0
[    5.125627] mybrd:     sector=208 bio-info: len=4096 p=ffffea0001fc30c0 offset=0
[    5.126099] mybrd: lookup: page-          (null) index--1 sector-208
[    5.126509] mybrd: copy: ffff88007f0c3000 <- 0 (4096-bytes)
[    5.126864] mybrd: 0 0 0 0 0 0 0 0
[    5.127084] mybrd:     sector=216 bio-info: len=4096 p=ffffea0001fc2800 offset=0
[    5.127556] mybrd: lookup: page-          (null) index--1 sector-216
[    5.127984] mybrd: copy: ffff88007f0a0000 <- 0 (4096-bytes)
[    5.128335] mybrd: 0 0 0 0 0 0 0 0
[    5.128550] mybrd:     sector=224 bio-info: len=4096 p=ffffea0001fc2840 offset=0
[    5.129010] mybrd: lookup: page-          (null) index--1 sector-224
[    5.129414] mybrd: copy: ffff88007f0a1000 <- 0 (4096-bytes)
[    5.129764] mybrd: 0 0 0 0 0 0 0 0
[    5.129979] mybrd:     sector=232 bio-info: len=4096 p=ffffea0001fc2880 offset=0
[    5.130440] mybrd: lookup: page-          (null) index--1 sector-232
[    5.130834] mybrd: copy: ffff88007f0a2000 <- 0 (4096-bytes)
[    5.131181] mybrd: 0 0 0 0 0 0 0 0
[    5.131395] mybrd:     sector=240 bio-info: len=4096 p=ffffea0001fc28c0 offset=0
[    5.131851] mybrd: lookup: page-          (null) index--1 sector-240
[    5.132299] mybrd: copy: ffff88007f0a3000 <- 0 (4096-bytes)
[    5.132670] mybrd: 0 0 0 0 0 0 0 0
[    5.132912] mybrd: end queue_rq
[    5.133130] mybrd: start queue_rq: request-ffff88007f910200 priv-ffff88013a330f50 request->special=          (null)
[    5.133793] CPU: 0 PID: 1036 Comm: a.out Not tainted 4.4.0-eudyptula+ #157
[    5.134232] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS 1.10.1-1ubuntu1 04/01/2014
[    5.134861]  ffff88013a330f50 ffff88007f08bab0 ffffffff8130fec0 ffff88007f910200
[    5.135400]  ffff88007f08bad0 ffffffff814ff623 ffff88007f9a9400 ffff88007f08baf8
[    5.135907]  ffff88007f08bb58 ffffffff812fb989 ffff88007f890000 ffff88007f9a9408
[    5.136427] Call Trace:
[    5.136592]  [<ffffffff8130fec0>] dump_stack+0x44/0x64
[    5.136922]  [<ffffffff814ff623>] mybrd_queue_rq+0x43/0xb0
[    5.137279]  [<ffffffff812fb989>] __blk_mq_run_hw_queue+0x1b9/0x340
[    5.137729]  [<ffffffff812fb7b7>] blk_mq_run_hw_queue+0x87/0xa0
[    5.138111]  [<ffffffff812fcbd9>] blk_mq_insert_requests+0xe9/0x170
[    5.138511]  [<ffffffff812fd6c4>] blk_mq_flush_plug_list+0x114/0x130
[    5.138919]  [<ffffffff812f384c>] blk_flush_plug_list+0xcc/0x230
[    5.139310]  [<ffffffff812f3d07>] blk_finish_plug+0x27/0x40
[    5.139659]  [<ffffffff811272d3>] __do_page_cache_readahead+0x183/0x1f0
[    5.140072]  [<ffffffff81127409>] ondemand_readahead+0xc9/0x240
[    5.140446]  [<ffffffff81127769>] page_cache_sync_readahead+0x29/0x40
[    5.140851]  [<ffffffff8111c558>] generic_file_read_iter+0x468/0x570
[    5.141274]  [<ffffffff811a4b40>] blkdev_read_iter+0x30/0x40
[    5.141633]  [<ffffffff8116ebd4>] __vfs_read+0xa4/0xd0
[    5.141957]  [<ffffffff8116f3c1>] vfs_read+0x81/0x120
[    5.142278]  [<ffffffff81170021>] SyS_read+0x41/0xa0
[    5.142591]  [<ffffffff81895b2e>] entry_SYSCALL_64_fastpath+0x12/0x6d
[    5.142995] mybrd: queue-rq: req=ffff88007f910200 len=4096 rw=READ
[    5.143406] mybrd:     sector=248 bio-info: len=4096 p=ffffea0001fc3d00 offset=0
[    5.143882] mybrd: lookup: page-          (null) index--1 sector-248
[    5.144294] mybrd: copy: ffff88007f0f4000 <- 0 (4096-bytes)
[    5.144654] mybrd: 0 0 0 0 0 0 0 0
[    5.144882] mybrd: end queue_rq
read 40960-bytes


------------------- ioctl -----------
[    5.145536] mybrd: asp: nrpages=32 host=ffff88013ac25e30
[    5.145907] mybrd: radix-tree: height=1 rnode=ffff880139880249 rnode-count=0
[    5.146369] mybrd: index=0 page=ffffea0001fc3680 pindex=0
[    5.146715] mybrd: index=1 page=ffffea0001fc3640 pindex=1
[    5.147062] mybrd: index=2 page=ffffea0001fc1140 pindex=2
[    5.147411] mybrd: index=3 page=ffffea0001fc0e80 pindex=3
[    5.147759] mybrd: index=4 page=ffffea0001fc1380 pindex=4
[    5.148109] mybrd: index=5 page=ffffea0001fc1100 pindex=5
[    5.148456] mybrd: index=6 page=ffffea0001fc2700 pindex=6
[    5.148797] mybrd: index=7 page=ffffea0001feec80 pindex=7
[    5.149145] mybrd: index=8 page=ffffea0001feeac0 pindex=8
[    5.149525] mybrd: index=9 page=ffffea0001feed40 pindex=9
[    5.149905] mybrd: bh: index=0 state=21 page=ffffea0001fc3680 blocknr=0 size=4096
[    5.150492] mybrd: bh: index=1 state=21 page=ffffea0001fc3640 blocknr=1 size=4096
[    5.151027] mybrd: bh: index=2 state=21 page=ffffea0001fc1140 blocknr=2 size=4096
[    5.151501] mybrd: bh: index=3 state=21 page=ffffea0001fc0e80 blocknr=3 size=4096
[    5.151981] mybrd: bh: index=4 state=21 page=ffffea0001fc1380 blocknr=4 size=4096
[    5.152460] mybrd: bh: index=5 state=21 page=ffffea0001fc1100 blocknr=5 size=4096
[    5.152938] mybrd: bh: index=6 state=21 page=ffffea0001fc2700 blocknr=6 size=4096
[    5.153417] mybrd: bh: index=7 state=21 page=ffffea0001feec80 blocknr=7 size=4096
[    5.153908] mybrd: bh: index=8 state=21 page=ffffea0001feeac0 blocknr=8 size=4096
[    5.154388] mybrd: bh: index=9 state=21 page=ffffea0001feed40 blocknr=9 size=4096


------------------- ioctl -----------
write 40960-bytes


------------------- ioctl -----------
[    5.155612] mybrd: asp: nrpages=32 host=ffff88013ac25e30
[    5.155955] mybrd: radix-tree: height=1 rnode=ffff880139880249 rnode-count=0
[    5.156419] mybrd: index=0 page=ffffea0001fc3680 pindex=0
[    5.156766] mybrd: index=1 page=ffffea0001fc3640 pindex=1
[    5.157117] mybrd: index=2 page=ffffea0001fc1140 pindex=2
[    5.157467] mybrd: index=3 page=ffffea0001fc0e80 pindex=3
[    5.157818] mybrd: index=4 page=ffffea0001fc1380 pindex=4
[    5.158169] mybrd: index=5 page=ffffea0001fc1100 pindex=5
[    5.158518] mybrd: index=6 page=ffffea0001fc2700 pindex=6
[    5.158866] mybrd: index=7 page=ffffea0001feec80 pindex=7
[    5.159216] mybrd: index=8 page=ffffea0001feeac0 pindex=8
[    5.159564] mybrd: index=9 page=ffffea0001feed40 pindex=9
[    5.159913] mybrd: bh: index=0 state=23 page=ffffea0001fc3680 blocknr=0 size=4096
[    5.160396] mybrd: bh: index=1 state=23 page=ffffea0001fc3640 blocknr=1 size=4096
[    5.160877] mybrd: bh: index=2 state=23 page=ffffea0001fc1140 blocknr=2 size=4096
[    5.161357] mybrd: bh: index=3 state=23 page=ffffea0001fc0e80 blocknr=3 size=4096
[    5.161851] mybrd: bh: index=4 state=23 page=ffffea0001fc1380 blocknr=4 size=4096
[    5.162320] mybrd: bh: index=5 state=23 page=ffffea0001fc1100 blocknr=5 size=4096
[    5.162786] mybrd: bh: index=6 state=23 page=ffffea0001fc2700 blocknr=6 size=4096
[    5.163256] mybrd: bh: index=7 state=23 page=ffffea0001feec80 blocknr=7 size=4096
[    5.163723] mybrd: bh: index=8 state=23 page=ffffea0001feeac0 blocknr=8 size=4096
[    5.164195] mybrd: bh: index=9 state=23 page=ffffea0001feed40 blocknr=9 size=4096


------------------- ioctl -----------
[    5.165084] mybrd: start queue_rq: request-ffff88007f910200 priv-ffff88013a330f50 request->special=          (null)
[    5.165771] CPU: 0 PID: 1036 Comm: a.out Not tainted 4.4.0-eudyptula+ #157
[    5.166293] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS 1.10.1-1ubuntu1 04/01/2014
[    5.166877]  ffff88013a330f50 ffff88007f08bb20 ffffffff8130fec0 ffff88007f910200
[    5.167396]  ffff88007f08bb40 ffffffff814ff623 ffff88007f9a9400 ffff88007f08bb68
[    5.167907]  ffff88007f08bbc8 ffffffff812fb989 ffff88007f890000 ffff88007f9a9408
[    5.168421] Call Trace:
[    5.168587]  [<ffffffff8130fec0>] dump_stack+0x44/0x64
[    5.168921]  [<ffffffff814ff623>] mybrd_queue_rq+0x43/0xb0
[    5.169280]  [<ffffffff812fb989>] __blk_mq_run_hw_queue+0x1b9/0x340
[    5.169694]  [<ffffffff812fb7b7>] blk_mq_run_hw_queue+0x87/0xa0
[    5.170079]  [<ffffffff812fcbd9>] blk_mq_insert_requests+0xe9/0x170
[    5.170489]  [<ffffffff812fd6c4>] blk_mq_flush_plug_list+0x114/0x130
[    5.170900]  [<ffffffff812f384c>] blk_flush_plug_list+0xcc/0x230
[    5.171290]  [<ffffffff812f3d07>] blk_finish_plug+0x27/0x40
[    5.171651]  [<ffffffff81125048>] generic_writepages+0x48/0x60
[    5.172026]  [<ffffffff81126d09>] do_writepages+0x19/0x30
[    5.172372]  [<ffffffff8111bfac>] __filemap_fdatawrite_range+0x6c/0x90
[    5.172792]  [<ffffffff8111c017>] filemap_write_and_wait+0x27/0x70
[    5.173194]  [<ffffffff811a4379>] __blkdev_put+0x69/0x220
[    5.173543]  [<ffffffff811a4927>] blkdev_put+0x47/0x100
[    5.173884]  [<ffffffff811a4a00>] blkdev_close+0x20/0x30
[    5.174229]  [<ffffffff81170bd7>] __fput+0xd7/0x1e0
[    5.174545]  [<ffffffff81170d19>] ____fput+0x9/0x10
[    5.174859]  [<ffffffff810705e3>] task_work_run+0x73/0x90
[    5.175209]  [<ffffffff810021a2>] exit_to_usermode_loop+0x92/0xa0
[    5.175599]  [<ffffffff81002b2e>] syscall_return_slowpath+0x4e/0x60
[    5.176000]  [<ffffffff81895c88>] int_ret_from_sys_call+0x25/0x8f
[    5.176404] mybrd: queue-rq: req=ffff88007f910200 len=40960 rw=WRITE
[    5.176811] mybrd:     sector=0 bio-info: len=4096 p=ffffea0001fc3680 offset=0
[    5.177275] mybrd: lookup: page-          (null) index--1 sector-0
[    5.177690] mybrd: lookup: page-          (null) index--1 sector-0
[    5.178099] mybrd: insert: page-ffffea0001fc3fc0 index=0 sector-0
[    5.178490] mybrd: lookup: page-ffffea0001fc3fc0 index-0 sector-0
[    5.178879] mybrd: copy: ffff88007f0ff000 <- ffff88007f0da000 (4096-bytes)
[    5.179322] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.179586] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.179851] mybrd:     sector=8 bio-info: len=4096 p=ffffea0001fc3640 offset=0
[    5.180317] mybrd: lookup: page-          (null) index--1 sector-8
[    5.180712] mybrd: lookup: page-          (null) index--1 sector-8
[    5.181112] mybrd: insert: page-ffffea0001fc3f80 index=1 sector-8
[    5.181502] mybrd: lookup: page-ffffea0001fc3f80 index-1 sector-8
[    5.181892] mybrd: copy: ffff88007f0fe000 <- ffff88007f0d9000 (4096-bytes)
[    5.182344] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.182609] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.182900] mybrd:     sector=16 bio-info: len=4096 p=ffffea0001fc1140 offset=0
[    5.183449] mybrd: lookup: page-          (null) index--1 sector-16
[    5.183927] mybrd: lookup: page-          (null) index--1 sector-16
[    5.184345] mybrd: insert: page-ffffea0001fc3f40 index=2 sector-16
[    5.184763] mybrd: lookup: page-ffffea0001fc3f40 index-2 sector-16
[    5.185165] mybrd: copy: ffff88007f0fd000 <- ffff88007f045000 (4096-bytes)
[    5.185613] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.185879] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.186149] mybrd:     sector=24 bio-info: len=4096 p=ffffea0001fc0e80 offset=0
[    5.186628] mybrd: lookup: page-          (null) index--1 sector-24
[    5.187031] mybrd: lookup: page-          (null) index--1 sector-24
[    5.187460] mybrd: insert: page-ffffea0001fc3f00 index=3 sector-24
[    5.187857] mybrd: lookup: page-ffffea0001fc3f00 index-3 sector-24
[    5.188257] mybrd: copy: ffff88007f0fc000 <- ffff88007f03a000 (4096-bytes)
[    5.188698] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.188964] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.189234] mybrd:     sector=32 bio-info: len=4096 p=ffffea0001fc1380 offset=0
[    5.189712] mybrd: lookup: page-          (null) index--1 sector-32
[    5.190117] mybrd: lookup: page-          (null) index--1 sector-32
[    5.190517] mybrd: insert: page-ffffea0001fc3ec0 index=4 sector-32
[    5.190912] mybrd: lookup: page-ffffea0001fc3ec0 index-4 sector-32
[    5.191310] mybrd: copy: ffff88007f0fb000 <- ffff88007f04e000 (4096-bytes)
[    5.191750] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.192015] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.192282] mybrd:     sector=40 bio-info: len=4096 p=ffffea0001fc1100 offset=0
[    5.192750] mybrd: lookup: page-          (null) index--1 sector-40
[    5.193170] mybrd: lookup: page-          (null) index--1 sector-40
[    5.193592] mybrd: insert: page-ffffea0001fc3e80 index=5 sector-40
[    5.193990] mybrd: lookup: page-ffffea0001fc3e80 index-5 sector-40
[    5.194392] mybrd: copy: ffff88007f0fa000 <- ffff88007f044000 (4096-bytes)
[    5.194843] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.195113] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.195381] mybrd:     sector=48 bio-info: len=4096 p=ffffea0001fc2700 offset=0
[    5.195851] mybrd: lookup: page-          (null) index--1 sector-48
[    5.196258] mybrd: lookup: page-          (null) index--1 sector-48
[    5.196670] mybrd: insert: page-ffffea0001fc3e40 index=6 sector-48
[    5.197065] mybrd: lookup: page-ffffea0001fc3e40 index-6 sector-48
[    5.197461] mybrd: copy: ffff88007f0f9000 <- ffff88007f09c000 (4096-bytes)
[    5.197893] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.198160] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.198425] mybrd:     sector=56 bio-info: len=4096 p=ffffea0001feec80 offset=0
[    5.198892] mybrd: lookup: page-          (null) index--1 sector-56
[    5.199296] mybrd: lookup: page-          (null) index--1 sector-56
[    5.199739] mybrd: insert: page-ffffea0001fc3e00 index=7 sector-56
[    5.200142] mybrd: lookup: page-ffffea0001fc3e00 index-7 sector-56
[    5.200537] mybrd: copy: ffff88007f0f8000 <- ffff88007fbb2000 (4096-bytes)
[    5.200969] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.201237] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.201503] mybrd:     sector=64 bio-info: len=4096 p=ffffea0001feeac0 offset=0
[    5.201974] mybrd: lookup: page-          (null) index--1 sector-64
[    5.202378] mybrd: lookup: page-          (null) index--1 sector-64
[    5.202781] mybrd: insert: page-ffffea0001fc3dc0 index=8 sector-64
[    5.203182] mybrd: lookup: page-ffffea0001fc3dc0 index-8 sector-64
[    5.203578] mybrd: copy: ffff88007f0f7000 <- ffff88007fbab000 (4096-bytes)
[    5.204017] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.204284] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.204549] mybrd:     sector=72 bio-info: len=4096 p=ffffea0001feed40 offset=0
[    5.205017] mybrd: lookup: page-          (null) index--1 sector-72
[    5.205424] mybrd: lookup: page-          (null) index--1 sector-72
[    5.205841] mybrd: insert: page-ffffea0001fc3d80 index=9 sector-72
[    5.206240] mybrd: lookup: page-ffffea0001fc3d80 index-9 sector-72
[    5.206653] mybrd: copy: ffff88007f0f6000 <- ffff88007fbb5000 (4096-bytes)
[    5.207125] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.207393] mybrd: a5 a5 a5 a5 a5 a5 a5 a5
[    5.207655] mybrd: end queue_rq
[    5.207909] a.out (1036) used greatest stack depth: 13424 bytes left
```
