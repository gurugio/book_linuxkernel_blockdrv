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
--> filemap_write_and_wait
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
find_get_page() searches the page cache of mapping and returns a page if there is a page including data at offset.
find_get_page() passes 0 to fgp_flags and gfp_mask of arguments of pagecache_get_page().
So pagecache_get_page() will return NULL if there is no such page.

Let's look into find_get_entry() which is called by pagecache_get_page().
It calls radix_tree_lookup_slot() to get page in the radix-tree and increases reference counter with page_cache_get_speculative().

There is page_cache_sync_readahead() in the callstack.
It means searching a page in the page cache by find_get_page() failed.
So page_cache_sync_readahead() reads data from a disk.
It reads more adjacent sectors than specified by upper layer.
Next data request is done without disk accessing.
page_cache_sync_readahead() calls read_pages() function that finally calls the operation of address_space: mapping->a_ops->readpages = blkdev_readpages.

blkdef_readpages calls mpage_readpages which are one of essential functions in block layer.
mpage_readpages sends bio to the request-queue of driver with submit_bio() and adds page into lru.
bio is created by do_mpage_readpage() and submit_bio() is called by mpage_bio_submit().

I've described some essential functions of the page cache very roughly.
Please read source code carefully and check other documents.
The page cache is never easy to understand because it applies many optimization techniques and supports many cores.
I don't understand every detail in the page cache.
Even-if I understood it, I could not describe it in one document.
Please take your time.

### add_to_page_cache_lru

It adds one page into the page cache and the lru list as following sequences.
1. radix_tree_maybe_preload: almost same to radix_tree_preload
1. page_cache_get: same to get_page, increase ref-count of the page
1. initialize mapping and index of the page
1. page_cache_tree_insert: add the page into radix-tree
1. radix_tree_preload_end: finish of radix_tree_preload
1. update statistics of each zone with ``__inc_zone_page_state()`` function.

You can check the statistics of each zone via /proc/zoneinfo file.
``__inc_zone_page_state`` updates this file.
nr_file_pages (NR_FILE_PAGES of ``__inc_zone_page_state()``) of /proc/zoneinfo is amount of pages used for the page cache in the zone.

After the page is added into the page cache, it adds the page with lru_page_add() that adds the page into per-cpu array, lru_add_pvec.

FYI, ``__add_to_page_cache_locked()`` increases the ref-count of the page and lru_cache_add() also increases the counter.
It means if the page is extracted from lru, the page will not be freed.
On the contrary, if you want to free a page in lru, you should extract it from lru and the page cache and decrease counter twice.

### do_mpage_readpage

It look very long and complex but the core of this function is creating bio with mpage_alloc().

Arguments of do_mpage_readpage() are:
* struct bio *bio: initial value is NULL, so new bio is created by mpage_alloc().
* struct page *page: just created, and added into the page and lru list. Data from disk will be stored in the page.
* unsigned nr_pages: amount of pages not mapped into bio yet
* sector_t *last_block_in_bio: sector number of the last block in bio
* struct buffer_head *map_bh: buffer head created for bio
* unsigned long *first_logical_block: the first block number of bio
* get_block_t get_block: A function changes the file offset into the block number and stores the block number at bh->b_blocknr field. File is virtually contigous data but not physically contiguous in the disk. So filesystem provides information which file block is stored where. If do_mpage_readpage is called for reading block device file, blkdev_get_block is passed as the get_block argument. Block device file considers the disk as a big contiguous file, so blkdev_get_block sets the block number at buffer head directly. Other filesystem, such as ext2, use its own mapping function, such as ext2_get_block.
* gfp_t gfp: page allocation flag for bio allocation

When do_mpage_readpage is called first, bio is NULL. So bio is allocated and a page is added.
Next call for do_mpage_readpage, bio is already created, so only page is added again and again.

### mpage_bio_submit

This function passes bio to the block layer via submit_bio().
The submit_bio() passes bio into mybrd via generic_make_request().
If bio processing is completed, the block layer calls callback function at bio->bi_end_io, which is set as mpage_end_io in mpage_bio_submit().
The mpage_bio_submit unlock the page, which is locked by add_to_page_cache_lru(), and set status of the page as uptodate.
And mpage_bio_submit releases bio object.

## block_write_begin: write data into the page cache

Let's look into block_write_begin() briefly.

block_write_begin() consists of grab_cache_page_write_begin() which calls pagecache_get_page, and __block_write_begin().

We already looked into pagecache_get_page() when we investigated find_get_page().
But here pagecache_get_page() is called with non-zero flags to allocate a page and add the page into the page cache.

What pagecache_get_page() does is:
* get FGP_LOCK|FGP_ACCESSED|FGP_WRITE|FGP_CREAT flag as argument
* FGP_LOCK: lock the page if page of specified index is in the page cache
* FGP_CREATE: allocate a page if there is no page in the page cache
 * locking is not necessary
 * set PG_referenced flag
 * add the page into the page cache and the lru list

Now it calls ``__block_write_begin()`` that does followings:
* the page should be locked
* create_page_buffers: allocate a buffer head and lock the page.
* blocksize is the size of a block. mybrd driver set the block size as PAGE_SIZE.
* block is the same to index.
* call get_block (blkdev_get_block for block device file) to find block number and set the buffer head
* set flags for the buffer head and the page

Now the data is included in the page cache.
Please notice that data is not written to disk yet.

## flush data from the page cache into the disk

We already checked that close system-call flushes the data from the page cach into the disk.
Closing the block device file releases the object of struct block_device and flush all data of the block device.
Of course, if there are other processes holding the block disk, data is not released.

Block layer calls callback function in mapping->a_ops->writepage that is blkdev_writepage().
And blkdev_writepage calls block_write_full_page() that calls ``__block_write_full_page()``.
Let's look into ``__block_write_full_page()`` briefly.

### ``__block_write_full_page``

First it calls create_page_buffers() to find buffer heads of the page.
And it locks all buffer heads and creates wbc object.

The wbc object represents how much page cache is freed.
For example, if we commands un-mounting of a disk, wbc is initialized to free all page cache of the disk and kernel will flush all pages.
If a page is locked, kernel will wait.
Or in normal case, kernel frees non-locked pages.

Then submit_bh_wbc() is called to flush each buffer head.
submit_bh_wbc() increases the record of wbc object and creates bio.
And submit_bh_wbc() calls submit_bio() to pass bio to the block layer.
The block layer function generic_make_request() will pass bio to mybrd driver.
