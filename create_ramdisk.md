# Ramdisk

In previous chapter, we found out what bio is.
In this chapter, we will make a ramdisk and look into how bio for the ramdisk is processed.
Ramdisk is, as the name is, a kind of disk that stores data in memory(RAM).
You can think it as a mimic of the hard-disk, but we will lose data after reboot.

Following link is a complete source of the ramdisk driver.

source file: https://github.com/gurugio/mybrd/blob/ch03-ramdisk/mybrd.c

## page management with the radix tree

### introduction to radix tree data structure

reference: https://lwn.net/Articles/175432/

Radix tree is used for page handling in Linux kernel. A driver allocates a page and adds into the radix tree with key value.
Key value could be any number such as page number.
So we will use the number of sector stored in the page.
When we need to read a sector, we will pass the number of sector to the radix tree and get a page which has data of the sector.

The detail for implementation of the radix tree is beyond of this document.
Please read "Understanding the Linux kernel" book that describe the implementation of the radix tree in detail.
We will not care about the internal of the radix tree implementation, but only use it.

### struct radix_tree_root

``struct radix_tree_root`` describes the root of the radix tree.
That is initialized with INIT_RADIX_TREE macro.

### radix_tree_preload()/_end() & radix_tree_insert()

We can add a page into the radix tree via radix_tree_insert() function.
It's simple to use, only pass the pointer to the root, key value and pointer to the page to that function.
Any unique value can be used for the key value.
We will use the sector number for the key value because the sector number is unique in a disk.

One more thing we need to notice is calling radix_tree_repload() and radix_tree_reload_end() function.
It's not mandatory but common to call them.
radix_tree_preload function helps radix_tree_insert succeed.
How? We need to allocate memory for the tree node when we add a page into the radix tree.
The tree node usually has the pointer to page, key value and so on.
radix_tree_preload() allocates several tree nodes in advance, so that radix_tree_insert() does not fail to allocate memory.
The reason we use the tree is that the tree is fast.
But if we should allocate a node whenever we add a node into the tree, it would be much slower.
So radix_tree_preload() allocates tree nodes in advance and stores the nodes at per-cpu space.
So allocation the tree node in radix_tree_insert() cannot fail.

If radix_tree_preload fails, we don't access the radix tree because there are very less memory.
We don't need to try to lock the radix tree.
If we would lock the radix tree and fail to allocate memory, the kernel can make the process sleep.
If a process hold the lock and sleep, it could generate deadlock.
It's critical to system.

So conclusion is calling radix_tree_preload() before calling radix_tree_insert().
And radix_tree_preload_end() should be called after radix_tree_insert()..

The best way to check how a function is used is checking other code.
Please find other code which calls radix_tree_preload().
Whenever you find a new kernel API and are not sure how to use it, you need to check other code.
Document and comment could be out of date but code is always the latest and never be expired.

### radix_tree_lookup()

It finds a page with the specific key in the tree.
It's easy to use.
It takes the pointer of a root node and key value and returns the pointer to the page.

### mybrd_lookup_page()

All IOs are handled sector by sector. Each IO has information about which and how many sectors it should be read/write.
Therefore the key value for the radix-tree should be sector number.

mybrd_lookup_page() function searches the tree and finds a page including the specified sector.
It takes the sector number as an argument and returns the pointer to the page.

Let's check the implementation.
The first code is calling rcu_read_lock().
Describing rcu functions is far beyond this document.
Please read following document if you want to know rcu in detail.

https://lwn.net/Articles/262464/

rcu_read_lock() allows multi-thread to access the tree concurrently.
Since searching a page in the radix-tree doesn't modify the radix-tree, we can use rcu_read_lock() here.

Next we create a key value with the sector number.
One page is 4096-byte and one sector 512-byte, so 8-sector can be stored in one page.
The first sector in a page should be used for the key creation.

Following is an example of pair of sector number and key value.


```
0 0

7 0

8 1

9 1

16 2
```

Therefore we can make a equation: ``key_value = (sector_number * 512) / 4096``
And that equation could be coded like: ``key = sector >> (12 - 9)``
Page size could be differect on each platform.
So kernel defines a page size with PAGE_SIZE macro and bit size with PAGE_SHIFT.
Final code would be: ``key = sector >> (PAGE_SHIFT - 9)``

Then we call radix_tree_lookup() with the key value to find a page.
Finding a page can be failed.
It means that any data is not written to the specified sector yet.

### mybrd_insert_page()

mybrd_inser_page() adds a page into the radix-tree.

It checks that the specified sector is already in the radix-tree.
If there is not the specified sector, it allocated a page.
It calls preload() and locks the spin-lock.
Then it creates a key with the sector number and save the key in the index field of struct page.
The page is added into the radix-tree with radix_tree_insert() function.

FYI, let's think about the page flag for a while.
We are making a driver for IO handling, so we MUST use GFP_NOIO flag.

I'll explain why briefly.
For example, let's assume that there is a function to allocate buffer_head for block IO.
If that function is called when system doesn't have enough free memory, page reclaim mechanism in kernel will be activated.
If there are many dirty pages, the page reclaiming will flush dirty pages into hard-disk and make free pages.
For dirty page sync, kernel need to allocate buffer_head to write data into hard-disk.
Therefore the function allocating buffer_head is called recursively.
GFP_NOIO and GFP_NOFS flags allocate page without generating IO, so they can prevent this situation.

In other words, if we don't use GFP_NOIO in block device driver, a driver is called recursively infinitely.

## data transferring between application and disk

So far, we investigated some sub-routines for the radix-tree.
Those sub-routines are used to transfer a sector to the radix-tree.
Now let's see how we can pass arbitrary-size data to the radix-tree.

And don't forget that the radix-tree is the representation of a ramdisk.
Whenever we pass data to the radix-tree, it simulates passing data to ramdisk.

### copy_from_user_to_mybrd()

This function writes data of one sector from kernel to ramdisk which is represented by the radix-tree.
Function arguments are

* src_page: a page including data
* len: size of data
* src_offset: offset of data in src_page
* sector: sector number of the data

Let's look into the source code.
We can find a page in the radix-tree with the specified sector number.
But we will not use the entire page.
We will use only 512-byte of the page.
So we need to calculate where to store data in the page.
That is the target_offset value.
The sector location in the page is ``sector % 8``.
``512 * (sector % 8)`` is the offset in the page because one sector is 512-byte.

The variable, copy, stores how many bytes will be copied.
One bio can store maximum 4096 bytes because we have set the block size of the disk as 4096.
Segment is stored in one block, so segment cannot be bigger than a block which is 4096 bytes.

Please notice We have a special case.
If offset is 2048 and data size is 4096, we need to copy data into two pages.
So we need to check the size of the first copy.

Next we find a page including the specified sector.
If that sector is not accessed before, finding the page have to be failed.
For that case, new pages should be added into the radix-tree.
If data will be stored only in a page, only one page is added.
If not, two page are added.

Finally data in the kernel page is copied into radix-tree.
Until now, we have only pages, so we should use kmap() to get kernel addresses of each page, and start memory copy.

### copy_from_mybrd_to_user()

This function gets data from ramdisk.
Kernel already informs the page where data should be copied.
Driver should find data inside of the radix-tree and copy data into the kernel page.

If the sector is not accessed before, there should not be any page.
In that case, data must be zero.
So we clear the kernel page with zero.

## bio handling

### add data handling at mybrd_make_request_fn()

We finished to implement sub-routines.
Now let's add data handling mybrd_make_request_fn() with the sub-routines.

We already have bio_for_each_segment() loop.
So we can add calling copy_from_mybrd_to_user() for data reading and copy_from_user_to_mybrd() for data writing.

Please notice that we should increase the sector number.
The bio has only the first sector number and bvec can have multi-sector data.
If data size of the bio is bigger than one sector size, we should increase the sector number in the loop.

### cache handling

There is one thing very important but it's easy to forget and hard to debug.
It is cache handling.

Please read following documents before going ahead.

* http://www.infradead.org/~mchehab/kernel_docs/unsorted/cachetlb.html
* http://www.linuxjournal.com/article/7105

Driver handles user-requested data.
The pages, driver should handle, are mapped to user address-space.
They are user pages but kernel mapped them into kernel address-space and provide them to driver.
Even-after kernel updates the pages. processor cache cannot be updated because pages are mapped to two address-spaces.
Therefore it's essential to invalidate cache.

For reading disk, driver copies its data into user page and page contents are updated.
After copy, driver should invalidate cache.
If user read the page, the updated data in memory will be sent to processor cache and user can see the updated data.

For writing disk, driver should access user's page.
Before accessing, driver should invalidate cache to read the latest data in user page.
As you can see in the driver source, driver calles cache invalidating function before copy_from_user_to_mybrd().

Cache handling depends on hardware platform and cache policy.
Above cache invalication would not be necessary for a certain platform.

Kernel always provides interfaces common for all platforms, so we should call cache handling without caring platform.

## experiement for data read/write

### check bio and data writing

We found out that data writing is easier when we tested bio.
Let's test data writing first.

```
/ # dd if=/dev/urandom of=/dev/mybrd bs=4096 count=2
[   56.168167] random: dd urandom read with 52 bits of entropy available
[   56.169718] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[   56.170757] mybrd: bio-info: sector=0 end_sector=8 rw=WRITE
[   56.171345] mybrd: segment-info: len=4096 p=ffffea0000005340 offset=0
[   56.171992] mybrd: lookup: page-          (null) index--1 sector-0
[   56.172631] mybrd: lookup: page-          (null) index--1 sector-0
[   56.173274] mybrd: insert: page-ffffea0000198740 index=0 sector-0
[   56.174157] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[   56.174780] mybrd: copy: ffff88000661d000 <- ffff88000014d000 (4096-bytes)
[   56.175493] mybrd: d7 22 c2 77 c3 dc ec ed
[   56.175913] mybrd: d7 22 c2 77 c3 dc ec ed
[   56.176353] mybrd: end mybrd_make_request_fn
[   56.176791] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[   56.177729] mybrd: bio-info: sector=8 end_sector=16 rw=WRITE
[   56.178326] mybrd: segment-info: len=4096 p=ffffea00001988c0 offset=0
[   56.179001] mybrd: lookup: page-          (null) index--1 sector-8
[   56.179666] mybrd: lookup: page-          (null) index--1 sector-8
[   56.180456] mybrd: insert: page-ffffea0000196900 index=1 sector-8
[   56.181155] mybrd: lookup: page-ffffea0000196900 index-1 sector-8
[   56.181836] mybrd: copy: ffff8800065a4000 <- ffff880006623000 (4096-bytes)
[   56.182535] mybrd: 3f 3 a0 b9 38 8e a4 d
[   56.182833] mybrd: 3f 3 a0 b9 38 8e a4 d
[   56.183148] mybrd: end mybrd_make_request_fn
2+0 records in
2+0 records out
8192 bytes (8.0[   56.183787] dd (1048) used greatest stack depth: 13832 bytes left
KB) copied, 0.015329 seconds, 521.9KB/s
```
2개의 write용 bio가 발생했습니다. 섹터는 0번과 8번입니다. 각각 8개의 섹터니까 4096바이트 크기네요. 세그먼트 정보와도 일치합니다. 섹터 0에 해당하는 페이지를 찾아봤지만 실패했다는게 나왔고 그래서 새로운 페이지를 할당해서 트리에 추가했고, 그걸 다시 읽어와서 데이터를 복사했다는걸 알 수 있습니다. 데이터를 /dev/urandom에서 읽어왔습니다. 이 장치는 난수를 발생시키는 장치입니다. 커널 로그에도 d7 22 등등 난수들이 저장된걸 확인할 수 있습니다.

이제 데이터가 잘 저장된건가 확인하기위해 파일을 읽어보겠습니다.
```
/ # dd if=/dev/mybrd of=/dev/null bs=4096 count=1
[  316.710833] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  316.711772] mybrd: bio-info: sector=0 end_sector=32 rw=READ
[  316.712281] mybrd: segment-info: len=4096 p=ffffea0000004b40 offset=0
[  316.712833] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  316.713373] mybrd: copy: ffff88000012d000 <- ffff88000661d000 (4096-bytes)
[  316.713963] mybrd: d7 22 c2 77 c3 dc ec ed
[  316.714339] mybrd: d7 22 c2 77 c3 dc ec ed
[  316.714702] mybrd: segment-info: len=4096 p=ffffea0000005300 offset=0
[  316.715270] mybrd: lookup: page-ffffea0000196900 index-1 sector-8
[  316.715791] mybrd: copy: ffff88000014c000 <- ffff8800065a4000 (4096-bytes)
[  316.716400] mybrd: 3f 3 a0 b9 38 8e a4 d
[  316.716740] mybrd: 3f 3 a0 b9 38 8e a4 d
[  316.717101] mybrd: segment-info: len=4096 p=ffffea0000005340 offset=0
[  316.717659] mybrd: lookup: page-          (null) index--1 sector-16
[  316.718212] mybrd: copy: ffff88000014d000 <- 0 (4096-bytes)
[  316.718696] mybrd: 0 0 0 0 0 0 0 0
[  316.718992] mybrd: segment-info: len=4096 p=ffffea0000005380 offset=0
[  316.719561] mybrd: lookup: page-          (null) index--1 sector-24
[  316.720111] mybrd: copy: ffff88000014e000 <- 0 (4096-bytes)
[  316.720598] mybrd: 0 0 0 0 0 0 0 0
[  316.720895] mybrd: end mybrd_make_request_fn
1+0 records in
1+0 records out
4096 bytes (4.0KB) copied, 0.010485 seconds, 381.5KB/s
```
1개 페이지만 읽어봤는데 이전과 마찬가지로 read-ahead가 실행되서 32개의 섹터를 읽어들이네요. 0번 섹터에 해당하는 페이지를 읽었고 써진 값이 d7 22 등 이전에 저장한 값과 같습니다. 써진 값을 제대로 읽어왔네요. 다음 페이지도 써진 값이 다시 읽혀진걸 확인할 수 있습니다. 그 다음 페이지들은 아직 데이터가 없는 페이지이므로 0으로 반환되었습니다.

###파일시스템 생성 실험

그냥 dd만 가지고 실험하면 좀 심심하니까 본격적으로 mybrd를 진짜 디스크라고 생각하고 파일시스템을 생성해보겠습니다. 방법은 간단합니다. mkfs.vfat를 쓰면 간단하게 vfat 파일시스템을 생성할 수 있습니다.
```
/ # mkfs.vfat /dev/mybrd 
[  449.918935] mybrd: start mybrd_ioctl
[  449.919710] mybrd: end mybrd_ioctl
mkfs.vfat: for t[  449.920526] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
his device secto[  449.922345] mybrd: bio-info: sector=0 end_sector=8 rw=WRITE
r size is 4096
[  449.923659] mybrd: segment-info: len=4096 p=ffffea00001a1d40 offset=0
[  449.925387] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  449.926476] mybrd: copy: ffff88000661d000 <- ffff880006875000 (4096-bytes)
[  449.927712] mybrd: eb 58 90 6d 6b 64 6f 73
[  449.928443] mybrd: eb 58 90 6d 6b 64 6f 73
[  449.929119] mybrd: end mybrd_make_request_fn
[  449.929821] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.931139] mybrd: bio-info: sector=8 end_sector=16 rw=WRITE
[  449.931984] mybrd: segment-info: len=4096 p=ffffea00001a7b80 offset=0
[  449.932824] mybrd: lookup: page-ffffea0000196900 index-1 sector-8
[  449.933496] mybrd: copy: ffff8800065a4000 <- ffff8800069ee000 (4096-bytes)
[  449.934241] mybrd: 52 52 61 41 0 0 0 0
[  449.934731] mybrd: 52 52 61 41 0 0 0 0
[  449.935031] mybrd: end mybrd_make_request_fn
[  449.935373] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.936096] mybrd: bio-info: sector=16 end_sector=24 rw=WRITE
[  449.936557] mybrd: segment-info: len=4096 p=ffffea00001a3700 offset=0
[  449.937062] mybrd: lookup: page-          (null) index--1 sector-16
[  449.937586] mybrd: lookup: page-          (null) index--1 sector-16
[  449.938085] mybrd: insert: page-ffffea00001abf80 index=2 sector-16
[  449.938580] mybrd: lookup: page-ffffea00001abf80 index-2 sector-16
[  449.939067] mybrd: copy: ffff880006afe000 <- ffff8800068dc000 (4096-bytes)
[  449.939605] mybrd: 0 0 0 0 0 0 0 0
[  449.939879] mybrd: 0 0 0 0 0 0 0 0
[  449.940149] mybrd: end mybrd_make_request_fn
[  449.940490] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.941199] mybrd: bio-info: sector=24 end_sector=32 rw=WRITE
[  449.941650] mybrd: segment-info: len=4096 p=ffffea00001af8c0 offset=0
[  449.942152] mybrd: lookup: page-          (null) index--1 sector-24
[  449.942654] mybrd: lookup: page-          (null) index--1 sector-24
[  449.943150] mybrd: insert: page-ffffea00001902c0 index=3 sector-24
[  449.943599] mybrd: lookup: page-ffffea00001902c0 index-3 sector-24
[  449.944123] mybrd: copy: ffff88000640b000 <- ffff880006be3000 (4096-bytes)
[  449.944619] mybrd: eb 58 90 6d 6b 64 6f 73
[  449.944905] mybrd: eb 58 90 6d 6b 64 6f 73
[  449.945290] mybrd: end mybrd_make_request_fn
[  449.945654] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.946371] mybrd: bio-info: sector=32 end_sector=40 rw=WRITE
[  449.946832] mybrd: segment-info: len=4096 p=ffffea0000190240 offset=0
[  449.947320] mybrd: lookup: page-          (null) index--1 sector-32
[  449.947816] mybrd: lookup: page-          (null) index--1 sector-32
[  449.948311] mybrd: insert: page-ffffea0000195540 index=4 sector-32
[  449.948793] mybrd: lookup: page-ffffea0000195540 index-4 sector-32
[  449.949313] mybrd: copy: ffff880006555000 <- ffff880006409000 (4096-bytes)
[  449.949891] mybrd: 52 52 61 41 0 0 0 0
[  449.950180] mybrd: 52 52 61 41 0 0 0 0
[  449.950475] mybrd: end mybrd_make_request_fn
[  449.950815] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.951515] mybrd: bio-info: sector=40 end_sector=48 rw=WRITE
[  449.951961] mybrd: segment-info: len=4096 p=ffffea0000196980 offset=0
[  449.952454] mybrd: lookup: page-          (null) index--1 sector-40
[  449.952936] mybrd: lookup: page-          (null) index--1 sector-40
[  449.953426] mybrd: insert: page-ffffea00001af880 index=5 sector-40
[  449.954024] mybrd: lookup: page-ffffea00001af880 index-5 sector-40
[  449.954523] mybrd: copy: ffff880006be2000 <- ffff8800065a6000 (4096-bytes)
[  449.955044] mybrd: 0 0 0 0 0 0 0 0
[  449.955303] mybrd: 0 0 0 0 0 0 0 0
[  449.955568] mybrd: end mybrd_make_request_fn
[  449.955897] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.956591] mybrd: bio-info: sector=48 end_sector=56 rw=WRITE
[  449.957028] mybrd: segment-info: len=4096 p=ffffea0000198140 offset=0
[  449.957519] mybrd: lookup: page-          (null) index--1 sector-48
[  449.958053] mybrd: lookup: page-          (null) index--1 sector-48
[  449.958659] mybrd: insert: page-ffffea00001a1dc0 index=6 sector-48
[  449.959227] mybrd: lookup: page-ffffea00001a1dc0 index-6 sector-48
[  449.959676] mybrd: copy: ffff880006877000 <- ffff880006605000 (4096-bytes)
[  449.960163] mybrd: f0 ff ff f ff ff ff ff
[  449.960452] mybrd: f0 ff ff f ff ff ff ff
[  449.960744] mybrd: end mybrd_make_request_fn
[  449.961056] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.961705] mybrd: bio-info: sector=56 end_sector=64 rw=WRITE
[  449.962120] mybrd: segment-info: len=4096 p=ffffea0000196480 offset=0
[  449.962582] mybrd: lookup: page-          (null) index--1 sector-56
[  449.963034] mybrd: lookup: page-          (null) index--1 sector-56
[  449.963602] mybrd: insert: page-ffffea00001955c0 index=7 sector-56
[  449.964043] mybrd: lookup: page-ffffea00001955c0 index-7 sector-56
[  449.964487] mybrd: copy: ffff880006557000 <- ffff880006592000 (4096-bytes)
[  449.964980] mybrd: 0 0 0 0 0 0 0 0
[  449.965228] mybrd: 0 0 0 0 0 0 0 0
[  449.965476] mybrd: end mybrd_make_request_fn
[  449.965782] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.966430] mybrd: bio-info: sector=64 end_sector=72 rw=WRITE
[  449.966842] mybrd: segment-info: len=4096 p=ffffea0000198080 offset=0
[  449.967300] mybrd: lookup: page-          (null) index--1 sector-64
[  449.967750] mybrd: lookup: page-          (null) index--1 sector-64
[  449.968197] mybrd: insert: page-ffffea00001987c0 index=8 sector-64
[  449.968637] mybrd: lookup: page-ffffea00001987c0 index-8 sector-64
[  449.969073] mybrd: copy: ffff88000661f000 <- ffff880006602000 (4096-bytes)
[  449.969568] mybrd: 0 0 0 0 0 0 0 0
[  449.969817] mybrd: 0 0 0 0 0 0 0 0
[  449.970064] mybrd: end mybrd_make_request_fn
[  449.970368] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.971016] mybrd: bio-info: sector=72 end_sector=80 rw=WRITE
[  449.971414] mybrd: segment-info: len=4096 p=ffffea0000196fc0 offset=0
[  449.971860] mybrd: lookup: page-          (null) index--1 sector-72
[  449.972314] mybrd: lookup: page-          (null) index--1 sector-72
[  449.972746] mybrd: insert: page-ffffea0000198800 index=9 sector-72
[  449.973276] mybrd: lookup: page-ffffea0000198800 index-9 sector-72
[  449.973695] mybrd: copy: ffff880006620000 <- ffff8800065bf000 (4096-bytes)
[  449.974162] mybrd: 0 0 0 0 0 0 0 0
[  449.974398] mybrd: 0 0 0 0 0 0 0 0
[  449.974634] mybrd: end mybrd_make_request_fn
[  449.974935] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.975531] mybrd: bio-info: sector=80 end_sector=88 rw=WRITE
[  449.975879] mybrd: segment-info: len=4096 p=ffffea00001de9c0 offset=0
[  449.976330] mybrd: lookup: page-          (null) index--1 sector-80
[  449.976814] mybrd: lookup: page-          (null) index--1 sector-80
[  449.977320] mybrd: insert: page-ffffea0000198300 index=10 sector-80
[  449.977745] mybrd: lookup: page-ffffea0000198300 index-10 sector-80
[  449.978185] mybrd: copy: ffff88000660c000 <- ffff8800077a7000 (4096-bytes)
[  449.978711] mybrd: f0 ff ff f ff ff ff ff
[  449.979069] mybrd: f0 ff ff f ff ff ff ff
[  449.979359] mybrd: end mybrd_make_request_fn
[  449.979693] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.980305] mybrd: bio-info: sector=88 end_sector=96 rw=WRITE
[  449.980721] mybrd: segment-info: len=4096 p=ffffea00001a44c0 offset=0
[  449.981190] mybrd: lookup: page-          (null) index--1 sector-88
[  449.981647] mybrd: lookup: page-          (null) index--1 sector-88
[  449.982113] mybrd: insert: page-ffffea00001988c0 index=11 sector-88
[  449.982739] mybrd: lookup: page-ffffea00001988c0 index-11 sector-88
[  449.983186] mybrd: copy: ffff880006623000 <- ffff880006913000 (4096-bytes)
[  449.983654] mybrd: 0 0 0 0 0 0 0 0
[  449.983894] mybrd: 0 0 0 0 0 0 0 0
[  449.984153] mybrd: end mybrd_make_request_fn
[  449.984499] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.985157] mybrd: bio-info: sector=96 end_sector=104 rw=WRITE
[  449.985567] mybrd: segment-info: len=4096 p=ffffea00001dbc40 offset=0
[  449.986018] mybrd: lookup: page-          (null) index--1 sector-96
[  449.986459] mybrd: lookup: page-          (null) index--1 sector-96
[  449.986965] mybrd: insert: page-ffffea00001dad40 index=12 sector-96
[  449.987478] mybrd: lookup: page-ffffea00001dad40 index-12 sector-96
[  449.987904] mybrd: copy: ffff8800076b5000 <- ffff8800076f1000 (4096-bytes)
[  449.988375] mybrd: 0 0 0 0 0 0 0 0
[  449.988611] mybrd: 0 0 0 0 0 0 0 0
[  449.988881] mybrd: end mybrd_make_request_fn
[  449.989179] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.989797] mybrd: bio-info: sector=104 end_sector=112 rw=WRITE
[  449.990220] mybrd: segment-info: len=4096 p=ffffea00001d8700 offset=0
[  449.990684] mybrd: lookup: page-          (null) index--1 sector-104
[  449.991119] mybrd: lookup: page-          (null) index--1 sector-104
[  449.991557] mybrd: insert: page-ffffea00001dd400 index=13 sector-104
[  449.992019] mybrd: lookup: page-ffffea00001dd400 index-13 sector-104
[  449.992547] mybrd: copy: ffff880007750000 <- ffff88000761c000 (4096-bytes)
[  449.993018] mybrd: 0 0 0 0 0 0 0 0
[  449.993273] mybrd: 0 0 0 0 0 0 0 0
[  449.993511] mybrd: end mybrd_make_request_fn
[  449.993807] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.994423] mybrd: bio-info: sector=112 end_sector=120 rw=WRITE
[  449.994832] mybrd: segment-info: len=4096 p=ffffea00001dff40 offset=0
[  449.995271] mybrd: lookup: page-          (null) index--1 sector-112
[  449.995706] mybrd: lookup: page-          (null) index--1 sector-112
[  449.996156] mybrd: insert: page-ffffea0000001900 index=14 sector-112
[  449.996591] mybrd: lookup: page-ffffea0000001900 index-14 sector-112
[  449.997057] mybrd: copy: ffff880000064000 <- ffff8800077fd000 (4096-bytes)
[  449.997524] mybrd: 0 0 0 0 0 0 0 0
[  449.997760] mybrd: 0 0 0 0 0 0 0 0
[  449.997995] mybrd: end mybrd_make_request_fn
/ # mount /[  453.806303] random: nonblocking pool is initialized
dev/mybrd ./mnt
[  459.351005] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.352306] mybrd: bio-info: sector=0 end_sector=8 rw=READ
[  459.353042] mybrd: segment-info: len=4096 p=ffffea00001dfd00 offset=0
[  459.353898] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  459.354478] mybrd: copy: ffff8800077f4000 <- ffff88000661d000 (4096-bytes)
[  459.354939] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.355230] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.355485] mybrd: end mybrd_make_request_fn
[  459.355765] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.356347] mybrd: bio-info: sector=0 end_sector=8 rw=READ
[  459.356680] mybrd: segment-info: len=4096 p=ffffea0000001940 offset=0
[  459.357107] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  459.357486] mybrd: copy: ffff880000065000 <- ffff88000661d000 (4096-bytes)
[  459.357907] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.358177] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.358436] mybrd: end mybrd_make_request_fn
[  459.358712] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.359314] mybrd: bio-info: sector=0 end_sector=8 rw=READ
[  459.359647] mybrd: segment-info: len=4096 p=ffffea0000001940 offset=0
[  459.360033] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  459.360405] mybrd: copy: ffff880000065000 <- ffff88000661d000 (4096-bytes)
[  459.360819] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.361067] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.361327] mybrd: end mybrd_make_request_fn
[  459.361668] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.362466] mybrd: bio-info: sector=0 end_sector=8 rw=READ
[  459.362841] mybrd: segment-info: len=4096 p=ffffea0000008100 offset=0
[  459.363316] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  459.363821] mybrd: copy: ffff880000204000 <- ffff88000661d000 (4096-bytes)
[  459.364423] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.364708] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.364990] mybrd: end mybrd_make_request_fn
[  459.365291] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.365891] mybrd: bio-info: sector=8 end_sector=16 rw=READ
[  459.366287] mybrd: segment-info: len=4096 p=ffffea0000008140 offset=0
[  459.366734] mybrd: lookup: page-ffffea0000196900 index-1 sector-8
[  459.367210] mybrd: copy: ffff880000205000 <- ffff8800065a4000 (4096-bytes)
[  459.367681] mybrd: 52 52 61 41 0 0 0 0
[  459.367938] mybrd: 52 52 61 41 0 0 0 0
[  459.368191] mybrd: end mybrd_make_request_fn
[  459.368501] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.369152] mybrd: bio-info: sector=48 end_sector=56 rw=READ
[  459.369550] mybrd: segment-info: len=4096 p=ffffea0000008180 offset=0
[  459.369992] mybrd: lookup: page-ffffea00001a1dc0 index-6 sector-48
[  459.370456] mybrd: copy: ffff880000206000 <- ffff880006877000 (4096-bytes)
[  459.370935] mybrd: f0 ff ff f ff ff ff ff
[  459.371211] mybrd: f0 ff ff f ff ff ff ff
[  459.371500] mybrd: end mybrd_make_request_fn
[  459.371839] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.372486] mybrd: bio-info: sector=112 end_sector=120 rw=READ
[  459.372934] mybrd: segment-info: len=4096 p=ffffea00000081c0 offset=0
[  459.373377] mybrd: lookup: page-ffffea0000001900 index-14 sector-112
[  459.373842] mybrd: copy: ffff880000207000 <- ffff880000064000 (4096-bytes)
[  459.374327] mybrd: 0 0 0 0 0 0 0 0
[  459.374570] mybrd: 0 0 0 0 0 0 0 0
[  459.374810] mybrd: end mybrd_make_request_fn
[  459.375118] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.375745] mybrd: bio-info: sector=0 end_sector=8 rw=WRITE
[  459.376106] mybrd: segment-info: len=4096 p=ffffea0000008100 offset=0
[  459.376505] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  459.377127] mybrd: copy: ffff88000661d000 <- ffff880000204000 (4096-bytes)
[  459.377757] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.378095] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.378424] mybrd: end mybrd_make_request_fn
/ # mount
rootfs on / type rootfs (rw,size=53748k,nr_inodes=13437)
none on /proc type proc (rw,relatime)
none on /sys type sysfs (rw,relatime)
/dev/mybrd on /mnt type vfat (rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro)
```

좀 깁니다. 그만큼 데이터를 많이 읽고쓰고 있네요. mount 명령으로 잘 마운트가 됩니다. 파일시스템이 생겼으니 당연히 파일을 한번 만들어봐야지요.
```
/ # cd mnt
/mnt # ls
/mnt # touch ddd
/mnt # cat > ddd
asdf
asdf
/mnt # 
```
파일이 생기고 파일에 데이터도 썼는데 드라이버가 아무런 동작도 하지 않습니다. 왜 그럴까요? 파일데이터가 바로 파일에 써지는게 아님을 알 수 있습니다. 우리가 파일에 데이터를 쓰면 파일시스템은 데이터를 메모리에 저장합니다. 이걸 버퍼 캐시라고 합니다.

데이터가 메모리에 저장되므로 드라이버에는 아직 데이터를 쓰라는 명령이 안온겁니다. 그럼 메모리의 데이터를 디스크에 저장하는 sync 프로그램을 실행해보겠습니다.
```
/mnt # sync
[  742.641735] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.644677] mybrd: bio-info: sector=48 end_sector=56 rw=WRITE
[  742.646476] mybrd: segment-info: len=4096 p=ffffea00001ff140 offset=0
[  742.648129] mybrd: lookup: page-ffffea0000198500 index-6 sector-48
[  742.649640] mybrd: copy: ffff880006614000 <- ffff880007fc5000 (4096-bytes)
[  742.651147] mybrd: f0 ff ff f ff ff ff ff
[  742.652050] mybrd: f0 ff ff f ff ff ff ff
[  742.653033] mybrd: end mybrd_make_request_fn
[  742.654187] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.656014] mybrd: bio-info: sector=80 end_sector=88 rw=WRITE
[  742.657160] mybrd: segment-info: len=4096 p=ffffea0000008400 offset=0
[  742.658498] mybrd: lookup: page-ffffea0000198600 index-10 sector-80
[  742.659742] mybrd: copy: ffff880006618000 <- ffff880000210000 (4096-bytes)
[  742.661136] mybrd: f0 ff ff f ff ff ff ff
[  742.661932] mybrd: f0 ff ff f ff ff ff ff
[  742.662620] mybrd: end mybrd_make_request_fn
[  742.663187] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.664410] mybrd: bio-info: sector=112 end_sector=120 rw=WRITE
[  742.665205] mybrd: segment-info: len=4096 p=ffffea0000062700 offset=0
[  742.666068] mybrd: lookup: page-ffffea0000198700 index-14 sector-112
[  742.666858] mybrd: copy: ffff88000661c000 <- ffff88000189c000 (4096-bytes)
[  742.667724] mybrd: 41 64 0 64 0 64 0 0
[  742.668159] mybrd: 41 64 0 64 0 64 0 0
[  742.668678] mybrd: end mybrd_make_request_fn
[  742.669284] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.670541] mybrd: bio-info: sector=120 end_sector=128 rw=WRITE
[  742.671346] mybrd: segment-info: len=4096 p=ffffea00000081c0 offset=0
[  742.672218] mybrd: lookup: page-ffffea0000198740 index-15 sector-120
[  742.673046] mybrd: copy: ffff88000661d000 <- ffff880000207000 (4096-bytes)
[  742.673962] mybrd: 61 73 64 66 a 61 73 64
[  742.674514] mybrd: 61 73 64 66 a 61 73 64
[  742.675022] mybrd: end mybrd_make_request_fn
[  742.675591] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.676769] mybrd: bio-info: sector=8 end_sector=16 rw=WRITE
[  742.677538] mybrd: segment-info: len=4096 p=ffffea00000778c0 offset=0
[  742.678383] mybrd: lookup: page-ffffea0000001cc0 index-1 sector-8
[  742.679156] mybrd: copy: ffff880000073000 <- ffff880001de3000 (4096-bytes)
[  742.680024] mybrd: 52 52 61 41 0 0 0 0
[  742.680516] mybrd: 52 52 61 41 0 0 0 0
[  742.681004] mybrd: end mybrd_make_request_fn
[  742.681545] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.682698] mybrd: bio-info: sector=112 end_sector=120 rw=WRITE
[  742.683450] mybrd: segment-info: len=4096 p=ffffea0000062700 offset=0
[  742.684271] mybrd: lookup: page-ffffea0000198700 index-14 sector-112
[  742.685103] mybrd: copy: ffff88000661c000 <- ffff88000189c000 (4096-bytes)
[  742.685970] mybrd: 41 64 0 64 0 64 0 0
[  742.686459] mybrd: 41 64 0 64 0 64 0 0
[  742.686908] mybrd: end mybrd_make_request_fn
/mnt # ls
ddd
/mnt # cat ddd
asdf
asdf
```
이제야 데이터가 드라이버에 전달됩니다. 파일에 쓴 데이터는 잘 전달됐을까요? asdf라고 썼으니 각각 아스키코드가 0x61 0x73 0x64 0x66입니다. 커널 로그 중간에 같은 값들이 전달된게 보이네요. 그 다음 값이 0xa인걸보니 개행문자까지도 저장된게 보입니다. asdf를 두줄썼으니까 또 0x61 0x73 0x64가 보이네요.

마지막으로 통계 정보를 확인해보겠습니다.
```
/ # cat /sys/block/mybrd/stat
      10        0      104       39       18        0      144       91        0       93       93
```
