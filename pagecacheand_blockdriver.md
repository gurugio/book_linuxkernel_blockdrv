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

페이지캐시를 관리하는 데이터 구조체는 struct address_space입니다. 이 구조체의 객체는 가장 먼저 inode의 i_mapping 필드에 저장됩니다. inode는 파일시스템에서 생성하겠지요. 우리가만든 mybrd 드라이버는 디스크를 등록하면 장치 파일이 생성됩니다. 이때 파일이 생성된다는 것은 곧 inode도 생성된다는 것입니다. 디스크를 등록하는 add_disk 함수의 어딘가에 inode를 생성하는 코드가 숨어있습니다. 그리고 디스크의 장치 파일의 inode->i_mapping 필드는 모두 def_blk_aops가 저장됩니다.

파일이 열릴때 open 시스템 콜에서 inode의 address_space 객체가 file의 f_mapping 필드에 inode의 i_mapping값을 저장합니다. 그리고나면 read/write 등 모든 시스템 콜에서 사용하는 file 객체에 address_space 객체가 사용되는 것이지요. inode는 파일이 열릴때만 참조되고, 그 이후로는 항상 file 객체만 사용됩니다. 그래서 같은 파일을 여러번 열 수 있고, 공유할 수 있는 것입니다.

struct address_space에서 가장 중요한 필드는 struct address_space_operations 입니다.

fs/block_dev.c 파일을 열어보면 아래와같이 file_operations 타입의 객체와 address_space_operations 타입의 객체가 정의되어있습니다.
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

open 시스템콜은 방금 말한대로 단지 file 구조체의 객체만 생성합니다. file의 객체의 f_op 필드에는 def_blk_fops의 포인터가 저장되고, file->f_mapping->a_ops 필드에 def_blk_aops의 포인터가 저장되는 것입니다. 그 다음에 read/write 등 실제 데이터를 처리하는 시스템콜이 호출되면 먼저 file_operaions에서 해당 콜백 함수가 호출되고, 그 콜백 함수에서 다시 address_space_operations의 콜백 함수가 호출되는 것입니다.

예를 들면 read 시스템콜은 vfs_read 함수등을 거쳐서 def_blk_fops.read_iter 를 호출합니다. 그러면 blkdev_read_iter가 호출될거고, blkdev_read_iter는 어느순간에 file->f_mapping->a_ops->readpages를 호출합니다. 그러면 blkdev_readpages가 호출되고, blkdev_readpages는 디스크에 접근합니다.

address_space 객체의 host 필드에 inode의 포인터가 있고, inode에는 해당 파일이 블럭 장치 파일일 경우 struct block_device 에 대한 포인터를 저장하고 있으므로 결국 address_space 객체만 있으면 현재 페이지캐시가 어떤 블럭 장치의 데이터를 저장하고 있는지를 알 수 있습니다. 따라서 IO를 실행하기 위해 드라이버로 전달할 bio 객체를 만들 때도 mybrd가 생성한 request-queue에 bio를 전달할 수 있는 것이지요.

###struct page의 mapping 과 index 필드

struct page의 mapping과 index 필드도 페이지 캐시를 위해 사용됩니다. mapping 필드는 address_space의 포인터가 저장됩니다. index 필드는 파일에서 현재 페이지의 offset를 저장합니다. index필드는 mybrd에서 만든 것과 동일하게 사용되는 것입니다. brd.c 패치 히스토리를 읽다보면 페이지캐시의 데이터 저장 방식을 따라서 만들었는데, Linus Torvalds의 아이디어였다라는 기록이 있습니다.

아직은 감이 안오실건데 코드를 보다보면 순서가 들어오실 겁니다.

문서를 단순화하기위해 버퍼헤드에 대한 내용은 생략하겠습니다. 블럭 장치의 블럭 크기는 페이지 크기와 같으므로 버퍼헤드가 별다른 역할을 하지 않기 때문입니다. 블럭 장치의 페이지캐시가 눈에 익어지면 좀더 복잡한 일반 파일의 페이지캐시도 좀더 쉽게 접근이 될것같습니다.

##커널 콜스택 확인
어플에서 시스템 콜을 호출하면 커널 레벨로 진입하고, 커널 레벨로 진입한 이후의 함수 호출들은 dump_stack() 함수를 써서 확인할 수 있습니다. 드라이버를 만들때도 써봤지요.

mybrd_make_request_fn()함수에 dump_stack()을 넣고 실행해보겠습니다. queue_mode값을 MYBRD_Q_BIO로 바꾸면 콜스택을 조금 줄일 수 있습니다.
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
이제 커널을 부팅하고 dd 명령을 이용해서 읽기쓰기를 해보면 콜스택이 출력됩니다.

다음은 제가 쓰기를 했을 때 제 환경에서 출력된 콜스택입니다.
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

다음은 읽기를 했을 때 콜스택입니다.
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

커널 소스를 따라가면서 한번 콜스택대로 함수가 호출되는지 확인해보세요. 중간중간에 static으로 선언된 함수들은 콜스택에 나타나지 않는다는 것도 알 수 있고, 어떻게 호출되는지 알 수 없는 콜백함수들도 있습니다.

예를들어 blkdev_read_iter함수는 ```__vfs_read()``` 함수에 직접적으로 호출되는게 아닙니다. ```__vfs_read()```함수를 보면
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
file 구조체에서 f_op값을 읽는데 file 구조체에 뭐가 들어있는지 그냥 봐서는 알 수가 없습니다.

커널은 인터페이스 디자인을 매우 신경써서 구현합니다. 왜냐면 파일시스템 개발자와 페이지 캐시 개발자가 다르고, 페이지 캐시 개발자와 블럭 레이어 개발자가 다르기 때문입니다. 그냥 다른게 아니라 나라도 다르고, 일하는 시간대도 다르고, 소속도 다르고, 같은건 커널을 개발한다는 것 뿐인 개발자들이 함께 개발하는 코드이므로 모듈화를 잘해야하고 인터페이스 정의도 매우 깐깐하게 합니다. 그래야 서로 다른 레이어/모듈간에 독립적으로 개발될 수 있겠지요. 만약 이쪽에서 개발한걸 다른쪽에서 그대로 써야한다면 수많은 개발자들이 서로의 결과물을 기다리다가 데드락이 걸릴 것입니다.

그래서 커널 소스를 보면 수많은 콜백함수들을 보게됩니다. 그럴때마다 코드를 처음 분석하는 입장에서는 막막할 때가 많습니다. 제가 주로 쓰는 방법은 구조체 이름으로 검색해보는 것입니다. struct file_operations타입의 객체를 어디선가 정적으로 정의하고 있기때문에 file 구조체에서 가져다쓰고있는 것이겠지요. 그러니 일단 커널 소스 전체에서 struct file_operations를 한번 검색해보는 것입니다. 그럼 분명 어디선가 struct file_operations를 정의하고있고, 정의된 객체를 file 구조체에 등록하는 함수가 있을 것입니다. 한번 찾아보겠습니다.
```
~/work/linux-torvalds $ /bin/grep file_operations * -R | wc
   3195   20545  264089
```
이번에는 운이 안좋았습니다. 너무 많네요. 3195개의 file_operations 정의가 있습니다. cscope등으로 검색해봐도 너무 많습니다. 이럴때는 별수없이 다른 책이나 구글 등을 통해서 파악할 수밖에 없습니다.

다행히 file_operations 구조체는 많이 쓰이는 것만큼 설명 자료가 많습니다. 우리는 블럭 장치를 분석하고 있으므로 블럭 장치만 생각한다면 결국은 struct def_blk_fops가 우리가 찾는 객체입니다. 자세한 설명은 Understanding the linux kernel 책을 참고하세요. 너무나 오랫동안 많이 사용되는 구조체이므로 자세한 설명이 있습니다.

어쨌든 지금은 def_blk_fop가 file->f_ops에 저장되어있다는 것만 생각하고 넘어가겠습니다. 그러면 결국은 다음과 같은 콜스택을 얻을 수 있게 됩니다.
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
그런데 쓰기에서 콜스택이 이상한게 보입니다. Sys_write같이 뭔가 시스템콜같은 함수 이름이 나타나야하는데 뜬금없이 시스템 콜이 끝나는것 같은 int_ret_from_sys_call이라는 함수가 호출됩니다.

여기서 또 커널을 분석할 때 자주 막히는 지점이 나타납니다. 어떤 처리가 synchronouse하게 되면 그냥 함수들의 호출 관계가 바로 나타납니다. 데이터를 읽는 것은 어플에게 당장 데이터를 줘야하므로 처리를 지연시킬 수 없습니다. 바로 그 순간에 데이터를 읽어와야합니다. 그게 메모리에있는 버퍼에서 읽어오던, 장치를 읽어서 읽어오던 데이터를 가져와야합니다. 그런데 쓰기는 다릅니다. 어플은 그냥 쓰기만하면 되고, 커널은 이 데이터를 언제 최종 블럭 장치에 써야할지 결정할 수 있습니다. 데이터를 잃어버리지만 않는다면 당장 블럭 장치에 접근할 필요가 없어집니다. 따라서 이 데이터가 언제 블럭 장치에 들어갈지는 분석하기가 어려워집니다.

비동기 데이터 처리를 설명하자면 제 지식도 한계가 있으므로 지금은 일단 write의 시스템 콜부터 페이지 캐시까지의 콜스택만 따로 뽑아보겠습니다. 이때는 mybrd 드라이버가 호출되는게 아니므로 mybrd 드라이버를 아무리 돌려봐야 알 수가 없습니다. 그냥 read의 콜스택을 보고 코드를 따라가보는게 빠릅니다. blkdev_read_iter라는 함수가 있으면 당연히 blkdev_write_iter라는 함수도 있을거라는 계산을 가지고 코드를 따라가보면 대강 다음과 같은 콜스택을 얻을 수 있습니다.
```
__vfs_write
--> new_sync_write
--> blkdev_write_iter
--> generic_perform_write 
--> a_ops->write_begin(&write_end) 
--> blkdev_write_begin(&blkdev_write_end) 
--> block_write_begin
```
읽기에서 얻은 콜스택과 완전히 대칭되는 함수들이 있어서 별로 어렵지 않게 찾을 수 있었습니다.

그리고 콜 스택 중간에 알수없는 a_ops 객체가 나타납니다. 이것은 일단 block_dev.c 파일에 있는 def_blk_aops 구조체라고 생각하고 넘기겠습니다. 뒤에 페이지 캐시를 제대로 분석할 때 제대로 소개하겠습니다.

## ext2에서의 콜스택

파일시스템과 페이지캐시의 관계를 아주아주 간단하게 한번 추적해보겠습니다. 제일 단순한 ext2 파일시스템의 경우만 보겠습니다.

ext2의 file_operations를 한번 볼까요.
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
소스를 뒤져보니 바로 file.c라는 파일에 정의되어있다는걸 알 수 있습니다.
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
시스템콜에서 read_iter와 write_iter를 호출할 것이니, generic_file_read_iter와 generic_file_write_iter부터 코드를 따라가보면 되겠네요.
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
참고로 a_ops는 inode.c 파일에 정의되어있습니다.
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
mybrd의 콜스택을 분석해본 결과와 중복되는 콜스택이 나타납니다.

읽기의 경우에는 do_generic_file_read부터, 쓰기에는 block_write_begin부터가 중복됩니다. 따라서 우리는 이 함수들부터 코드를 읽어보겠습니다.

##장치에서 페이지캐시로 데이터 읽어오기
데이터를 읽을 때 콜스택을 보면 mybrd블럭 장치를 직접 읽을 경우와 파일시스템에 존재하는 파일을 읽을 경우 모두 do_generic_file_read을 호출하는걸 알 수 있습니다. 이 함수를 간단하게 분석해보면 페이지캐시가 어떻게 동작하는지, 언제 페이지캐시에서 블럭 장치를 읽어서 페이지캐시를 추가하는지 등을 알아보겠습니다.

리눅스 소스를 보시면서 글을 읽으시길 바랍니다. 소스를 하나하나 복사하는건 의미가 없으니까요. 나중에 4.10이 되든, 5.0이 나오든 소스는 바뀝니다. 하지만 디자인과 코드의 목표는 남습니다. 페이지캐시를 구현하는 방법은 달라지지만 페이지캐시의 목적과 큰 디자인은 오래가겠지요. 그러니 소스 한줄한줄보다는 왜 이렇게 구현했는가 목적이 뭔가를 생각하는게 처음 커널을 분석할때 필요한 태도인것 같습니다. 나중에 실무에서나 취미로나 버그를 잡을 때 특정 버전의 코드를 더 깊게 봐야할때가 있겠지요.

###do_generic_file_read

블럭 장치 파일(/dev/mybrd)이든 파일시스템의 파일이든 데이터를 읽을 때는 공통적으로 do_generic_file_read가 호출됩니다. 길고 복잡한 함수이므로 핵심적인 동작들만 따져보겠습니다.

####함수 인자

함수인자부터 뭔지 보겠습니다.

* struct file *filep: 읽을 파일에 해당하는 file 객체입니다. file 구조체의 f_mapping이 struct address_space 객체를 가르키는 포인터입니다.
* loff_t *ppos: read시스템콜을 호출하기전에 lseek 시스템콜을 써서 파일의 어디부터 읽을지를 선택합니다. lseek시스템콜은 실질적으로 어떤 처리를 하는게 아니라 struct file 객체의 f_pos 필드에 위치를 기록했다가 read나 write 시스템 콜이 호출되었을 때 읽어서 사용합니다.
 * 페이지 캐시는 페이지단위로 데이터가 저장되어있습니다. 따라서 페이지캐시의 radix-tree에서 사용할 키값은 ppos 값을 페이지 크기로 나눈 값이 되겠지요.
* struct iov_iter *iter: 파일시스템에서 사용하는 자료구조입니다. 페이지캐시에서 사용되는건 아니므로 http://revdev.tistory.com/55 를 참고하시기 바랍니다.
 * 참고 추가: https://lwn.net/Articles/625077/
* ssize_t written: generic_file_read_iter에서 direct IO가 발생해서 읽기가 일부 처리된 경우에 얼마나 읽기가 끝났는지 알려주는 값입니다. 그냥 함수가 호출될때의 값은 0으로 생각해도 됩니다. 데이터를 읽으면서 written의 값이 증가하고 읽기가 끝나면 written 값을 반환합니다.

####find_get_page (= pagecache_get_page)

find_get_page는 pagecache_get_page를 호출하는 wrapper 함수입니다.

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

####add_to_page_cache_lru
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

####do_mpage_readpage
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

####mpage_bio_submit

mpage_alloc으로 생성한 bio 객체를 submit_bio 함수에 전달합니다. 결국 submit_bio는 generic_make_request를 통해서 mybrd로 넘어갑니다. bio 처리가 끝나면 호출된 bio->bi_end_io 콜백함수는 mpage_end_io입니다. add_to_page_cache_lru에서 페이지를 잠궜으므로 mpage_end_io에서는 페이지 락을 풀고 페이지를 페이지의 데이터가 막 읽혀진 상태이니 uptodate 상태로 표시합니다. 그리고 다쓴 bio를 해지합니다.

####copy_page_to_iter

iter에는 유저 레벨의 버퍼에 대한 정보가 들어있습니다. page에 있는 데이터를 유저 레벨 버퍼로 복사합니다.
iov_iter_count에서 iter->count 필드가 0이되면 do_generic_file_read가 종료됩니다.

##페이지캐시에 데이터 쓰기
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

##페이지캐시에 있는 페이지 해지
###/proc/sys/vm/drop_caches

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

###페이지캐시에서 장치로 데이터 쓰기

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
