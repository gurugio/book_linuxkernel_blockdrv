# Create a disk

Finally we can start developing a block device.
Until last chapter, we set up an environment to test our driver and created a skeleton code.
In this chapter, we will created a virtual disk and check how data is transferred between kernel and the block driver.

reference: https://lwn.net/Articles/25711/
driver source: https://github.com/gurugio/mybrd/blob/ch02-disk/mybrd.c

## request_queue and gendisk

We will create two object of ``struct request_queue`` and ``struct gendisk`` that are the essential for the block driver.
Please read following each section and source code in parallel.

### struct mybrd_device

``struct mybrd_device`` is selection of data that is used for whole mybrd driver.
mybrd_lock is lock for synchronization.
mybrd_queue and mybrd_disk are the objects of request_queue and gendisk data structures.
I'll explain them in following section with the functions which create them.
Please note again that this chapter is dedicated for creating request_queue and gendisk objects because they are essential for the block driver.

### mybrd_init() / mybrd_alloc()

mybrd_init() calls register_blkdev() to register itself and calls mybrd_alloc() to create object of mybrd_device structure.
And mybrd_alloc() also creates objects of struct request_queue and struct gendisk.
I'll introduce what each object is for in following sections.

#### mybrd_device object

struct mybrd_device is a set of spin-lock, request_queue and gendisk.
Those three objects are used by almost every function in the driver.
Therefore I make struct mybrd_device to pass them to each function easily.

#### request_queue object

Let me introduce a request_queue object of struct request_queue.
Queue is a data structure to store data on one side and extract data on another side.
So request_queue is a queue to store requests.
Requests in the queue is extracted by kernel and consumed by mybrd_make_request_fn() function as I will introduce later again.

At this point, you should know there is a queue so-called request-queue and it should be initialized as source code does.
There are many fields in struct request_queue.
You don't need to understand all of the fields right now.
Let's skip the detail and just use the code as it is.

Please note one thing.
blk_queue_make_request() creates a connection between request_queue object and mybrd_make_request_fn() function.
Every queue has a pair of producer and consumer which add data in the queue and extract data from the queue, respectively.
Can you guess what the producer and consumer of request_queue are?
Kernel already has both of the producer and consumer.
Because they are common for all block device, kernel developers implements the generic framework to handle request_queue for device driver developers.
(One of kernel developer's job is making a common framework for device drivers.
It can reduce the total size of kernel code. Less code generates less bug.
And it's also comfortable for driver developers.
Kernel code is reviewd by so many open source developers and tested by so many machines all of the world.
But device driver is usually developed by a company.
Which could be more reliable? Of course, kernel is reviewd more and tested more.
So driver should use kernel function as much as it can.
If you become driver developer, please remember that you should use kernel function.
And if your function looks common, please report it to kernel community.
You will be welcomed.)

blk_queue_make_request() function introduces mybrd_queue and mybrd_make_request_fn() to kernel.
Then kernel can add a request into mybrd_queue and extract a request and pass it to mybrd_make_request_fn() function.
(NOTE. You should not understand what kernel does and what driver should do at the moment.
In this chaper, please focus only on driver.
Please understand what driver should do.
In later chapters, we will read the block layer code of kernel and understand how kernel use the queue of the driver and how kernel pass the request to the request-handler of the driver.)

There is a pair of functions to create and destroy the request_queue: blk_alloc_queue_node() and blk_cleanup_queue.
NEVER use other memory allocation functions like kmalloc.
Creating and destroying the request_queue is common, so kernel already has functions for them.
Many kernel objects have thier own creating and destroying functions.
Whenever you want to use a kernel object, you must check there is dedicated function for it.

Kernel handles queue. What should the driver do?
Driver gets the request from kernel and processes the request.
If our driver is the hard-disk driver, the driver get the request and check where the data is, disk or memory according to data direction, where the data should be written or read and so on.
So now we can understand what information is in a request.
In brief, the request has all information to read&write data between memory and block device, for instance, where the data is, how big the data is and so on.
I'll explain mybrd_make_request_fn() soon.
You can understand the request in detail then.

#### gendisk object

In previous section, I've made request_queue object.
Next step is making gendisk object.
The gendisk object represents a disk and has all information for the disk management.
And gendisk object must be allocated by alloc_disk() function.
The argument of alloc_disk() is 1 that means the allocated disk has one partition.
So if you want to use the whole disk as it is, you can pass 1 to alloc_disk().

You can see there is setting of the major and minor number when gendisk is initialized.
So you can understand a device file is created now.
The minor number is set as 111 in disk->first_minor.
Soon we will check the minor number of the device file.
It must be 111.
If we pass 2 to alloc_disk() and set disk->first_minor as 111, there would be two device files with minor numbers 111 and 112.
You can change the source file and test it.

disk->queue is set to request-queue we just created with blk_alloc_queue_node().
Disk name is set to disk->disk_name.
Disk size is set with set_capacity.
The unit of the disk size is sector which is 512-bytes, so we need to divide the 4M with 512.

The disk structure has fops field.
You can see read, write, open, close and ioctl function pointers in the definition of struct block_device_operations.
They are called when user calls read, write, open, close and ioctl system call.
Our driver, mybrd, created /dev/mybrd device file.
So if user opens /dev/mybrd file and calls ioctl system call, mybrd_ioctl function in the driver file is called.
Please try to make a application to open mybrd device file and call ioctl().
You will be able to find a message "start mybrd_ioctl" and "end mybrd_ioctl".
dump_stack() function prints back-trace of the current function in the kernel log.
So you can add dump_stack() in mybrd_ioctl() to see how a system call is handled in the kernel mode.

Now we created the gendisk object.
Next we should inform the kernel about our gendisk object.
The kernel gets the gendisk object and make device files, sysfs entry and so on.
We also created the request_queue but we didn't inform the kernel about the request_queue.
We store a pointer of the request_queue in gendisk object and pass only gendisk object to kernel.
So gendisk object is the essential object in block driver and has almost every information about a disk.
A function to pass the gendisk object is add_disk().

After adding the gendisk object with add_disk(), a device file /dev/mybrd and a sysfs directory /sys/block/mybrd are created.
And kernel starts generating I/O to access the disk.
Therefore the disk should be ready to handle I/O before calling add_disk().

# IO processing

Let's read mybrd_make_request_fn() to understand what a driver should do for IO processing.

## bio object

I told you that driver created a queue of requests, so-called request-queue, and kernel sends I/O request via that queue.
We will make a request later but in this chapter I will introduce a bio object first, because it's more simple and the most basic unit of I/O processing.
As we see how kernel processes the bio object, we can understand the basic concepts of I/O processing of the kernel.
A request consists of several bio objects.
So we first need to understand the bio processing to understand the request processing.

### struct bio & struct bio_vec

The most essential information for disk IO is the location and size of the data.
And the basic unit of disk IO is a sector, so the hard disk can read/write sector by sector.
So struct bio is a representation of the location and the number of sectors.

Let's first look into what is disk briefly.

https://en.wikipedia.org/wiki/Hard_disk_drive

Physically a hard disk is made of several platters.
A platter consists of 512-byte sectors.

In kernel's a point of view, the hard disk is an array of sector.
So the mybrd driver should inform kernel how many sectors, where the sectors are.
And kernel passes the location and number of sectors to mybrd driver.
If mybrd driver is for actual physical disk, it must have information about physical compinents of the disk and mapping table for sector and physical location on platters.
But we don't have any physical disk and our purpose is also not to make a driver for a certain hard disk.
Our purpose is understanding how the kernel processes the IO and how to make a generic block driver.
A block driver is not only physical disk driver but also virtual disk driver such as RAID device.
Therefore mybrd driver handles only sectors.

You must've implemented some user application to read/write file on the disk.
You didn't specify any information about sector when you call read()/write() and other system calls.
You only specify a file, offset and size.
The filesystem layer and block layer of kernel calculates the address and number of sectors with the information from user.
And kernel passes only sector information to driver.
The sector information is represented by struct bio.

Following link show the structure of struct bio.

http://www.makelinux.net/books/lkd2/ch13lev1sec3

One bio object consists of several bio_vec that is representation of a segment.
A segment indicates a page that includes data.
Each bio_vec can have maximum 8 sectors because it should exist inside a page.

It's a little bit confusing. But the code is simple. We need only information in the bio:
* number of the first sector: bio->bi_iter.bi_sector
* how many sectors: bio_sectors(bio)
* read or write: bio_rw(bio)

We don't need to check each bio_vec objects in bio object.
Kernel developers already made an macro, bio_for_each_segment, to extract the bio_vec one by one from bio.
We can get each bio_vec bio_for_each_segment and get where data is in a page.
* bvec.bv_len: data size
* bvec.bv_page: a page including data
* bvec.bv_offset: an offset where data begins inside-of the page

Current driver code only prints the information of each bio_vec.
But what should actual driver do in bio_for_each_segment() loop?
It usually does DMA transfer between disk and memory.
And DMA transfer is done page by page.
That's why bio_vec has information about page.
One bio can be passed to the scatter-gatter DMA transfer.

Nevertheless actual data transfer is done in bio_for_each_segment() loop.
And next step is finishing the bio object.
It would be allocated somewhere, so it should be freed.
That is done by bio_endio() function.

Next is updating statistics of the disk with generic_start/end_io_acct() functions.
There are several files for statistics in sysfs of each disk, for instance /sys/block/mybrd/stat file.

The last is returning BLK_QC_T_NONE value.
Kernel calls mybrd_make_request_fn() function, so it informs kernel that there is no error.

You can find more detail information in kernel documentation: https://www.kernel.org/doc/Documentation/block/biodoc.txt

# test mybrd driver

```
[    0.499199] 
[    0.499199] 
[    0.499199] mybrd: module loaded
[    0.499199] 
[    0.499199] 
[    0.499199] 
[    0.500069] mybrd: mybrd major=253
[    0.500292] mybrd: start mybrd_alloc
[    0.500559] mybrd: create mybrd:ffff8800065438c0
[    0.501045] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff8800065438c0
[    0.501723] Buffer I/O error on dev mybrd, logical block 0, async page read
[    0.502222] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff8800065438c0
[    0.502933] Buffer I/O error on dev mybrd, logical block 0, async page read
[    0.503366] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff8800065438c0
[    0.503964] Buffer I/O error on dev mybrd, logical block 0, async page read
[    0.504415] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff8800065438c0
[    0.505025] Buffer I/O error on dev mybrd, logical block 0, async page read
[    0.505463] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff8800065438c0
[    0.506074] Buffer I/O error on dev mybrd, logical block 0, async page read
[    0.506586] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff8800065438c0
[    0.507263] Buffer I/O error on dev mybrd, logical block 0, async page read
[    0.507735] Dev mybrd: unable to read RDB block 0
[    0.508074] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff8800065438c0
[    0.508671] Buffer I/O error on dev mybrd, logical block 0, async page read
[    0.509109] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff8800065438c0
[    0.509711] Buffer I/O error on dev mybrd, logical block 0, async page read
[    0.510176]  mybrd: unable to read partition table
[    0.510540] mybrd: end mybrd_alloc
[    0.510801] mybrd: global-mybrd=ffff8800065438c0
```

Let's check the bio that driver receives from kernel.

We have built kernel with empty driver in previous chapter.
Now let's build and run kernel again with new driver source.
We can see some messages from mybrd_init() and mybrd_alloc().
And there are many messages from mybrd_make_request_fn().
I said that mybrd_make_request_fn() will print the information of bio object but there are only I/O errors: "Buffer I/O error on dev mybrd, logical block 0, async page read".
Why?

As I said before, I/O will be generated just after add_disk() passes gendisk object to kernel.
The mybrd driver allocates mybrd_device object with mybrd_alloc() and stores it into global_mybrd variable.
So add_disk() is called first and next global_mybrd variable is set.
mybrd_make_request_fn() compares mybrd and global_mybrd and calls bio_io_error() if mybrd is not global_mybrd: ``if(mybrd != global_mybrd)`` to inform kernel that I/O processing is failed.
There is a time gap between add_disk() and set global_mybrd.
Some I/Os generated between that time gap will be failed because global_mybrd is not set.

Nevertheless we confirm that driver is loaded and starts I/O processing.
Let's check the device file and sysfs entris in /sys/block/mybrd.

```
/ # ls -l /dev/mybrd
brw-rw----    1 0        0         253, 111 Nov  3 14:07 /dev/mybrd
/ # ls /sys/block/mybrd
alignment_offset   ext_range          range              stat
bdi                holders            removable          subsystem
capability         inflight           ro                 trace
dev                power              size               uevent
discard_alignment  queue              slaves
```

You can check the contents of each file with cat command.
The stat file shows how many IO are generated.
The size file shows the size of file.
Please check other files with cat command.

There is a queue directory that has the information of request_queue.

```
/ # ls /sys/block/mybrd/queue/
add_random              logical_block_size      nr_requests
discard_granularity     max_hw_sectors_kb       optimal_io_size
discard_max_bytes       max_integrity_segments  physical_block_size
discard_max_hw_bytes    max_sectors_kb          read_ahead_kb
discard_zeroes_data     max_segment_size        rotational
hw_sector_size          max_segments            rq_affinity
io_poll                 minimum_io_size         scheduler
iostats                 nomerges                write_same_max_bytes
/ # cat /sys/block/mybrd/queue/logical_block_size 
4096
/ # cat /sys/block/mybrd/queue/max_hw_sectors_kb 
512
/ # cat /sys/block/mybrd/queue/max_sectors_kb 
127
/ # cat /sys/block/mybrd/queue/max_segment_size 
65536
```

There are many files that show the properties of the queue.
You can find the detail of each file in the kernel document directory.
The kernel document is a collection of text documents that describe many kernel code and design.
Documentation/block directory has files for the block device and driver and queue-sysfs.txt has description for request_queue.
You can download kernel source and read documenent files.
Or visit link: https://www.kernel.org/doc/Documentation/block/queue-sysfs.txt

For example, logical_block_size has 4096 value that value is what we set with blk_queue_logical_block_size().
Some files have values we set in mybrd driver and others have the default values.

Next we try to write data at the disk via dd tool.
Following is writing 4096-byte of zero twice.

```
/ # dd if=/dev/zero of=/dev/mybrd bs=4096 count=2
[   19.989549] mybrd: start mybrd_make_request: block_device=ffff8800061b0000 mybrd=ffff880006821920
[   19.990228] mybrd: bio-info: sector=0 end_sector=8 rw=WRITE
[   19.990630] mybrd: bio-info: end-io=ffffffff8119de20
[   19.990993] mybrd: segment-info: len=4096 p=ffffea00001db380 offset=0
[   19.991396] mybrd: end mybrd_make_request
[   19.991668] mybrd: start mybrd_make_request: block_device=ffff8800061b0000 mybrd=ffff880006821920
[   19.992190] mybrd: bio-info: sector=8 end_sector=16 rw=WRITE
[   19.992524] mybrd: bio-info: end-io=ffffffff8119de20
[   19.992824] mybrd: segment-info: len=4096 p=ffffea00001a2dc0 offset=0
[   19.993205] mybrd: end mybrd_make_request
2+0 records in
2+0 records out
8192 bytes (8.0KB) copied, 0.003940 seconds, 2.0MB/s
/ # cat /sys/block/mybrd/stat 
       0        0        0        0        2        0       16        3        0        0        0
```

The first bio is for writing data at 8 sectors from 0-th sector.
Each bio has one segment that has 4096 bytes data.
Driver prints the pointer of the page object.
And 4096-byte segment means that the segment occupies entire page, so offset is zero.

Next is reading 4096 bytes twice.

```
/ # dd if=/dev/mybrd of=/dev/null bs=4096 count=2
[   81.453448] mybrd: start mybrd_make_request: block_device=ffff8800061b0000 mybrd=ffff880006821920
[   81.454585] mybrd: bio-info: sector=0 end_sector=32 rw=READ
[   81.455275] mybrd: bio-info: end-io=ffffffff811a7dd0
[   81.455928] mybrd: segment-info: len=4096 p=ffffea00001db280 offset=0
[   81.456681] mybrd: segment-info: len=4096 p=ffffea00001a83c0 offset=0
[   81.457419] mybrd: segment-info: len=4096 p=ffffea00001db740 offset=0
[   81.458153] mybrd: segment-info: len=4096 p=ffffea00001d9ac0 offset=0
[   81.458888] mybrd: end mybrd_make_request
[   81.459362] mybrd: start mybrd_make_request: block_device=ffff8800061b0000 mybrd=ffff880006821920
[   81.460390] mybrd: bio-info: sector=32 end_sector=96 rw=READ
[   81.461043] mybrd: bio-info: end-io=ffffffff811a7dd0
[   81.461635] mybrd: segment-info: len=4096 p=ffffea00001a8f40 offset=0
[   81.462360] mybrd: segment-info: len=4096 p=ffffea0000199900 offset=0
[   81.463091] mybrd: segment-info: len=4096 p=ffffea00001db2c0 offset=0
[   81.463867] mybrd: segment-info: len=4096 p=ffffea00001db680 offset=0
[   81.464888] mybrd: segment-info: len=4096 p=ffffea00001a8d00 offset=0
[   81.465790] mybrd: segment-info: len=4096 p=ffffea00001a8d40 offset=0
[   81.466629] mybrd: segment-info: len=4096 p=ffffea00001a5e00 offset=0
[   81.467451] mybrd: segment-info: len=4096 p=ffffea00001a5e40 offset=0
[   81.468311] mybrd: end mybrd_make_request
2+0 records in
2+0 records out
8192 bytes (8.0KB) copied, 0.015475 seconds, 517.0KB/s
```

Something weird happens.
Two bio is generated.
The first bio has 4 segments and reading 32 sectors.
The second one has 8 segments and reading 64 sectors.
Why? We commands reading 8192 bytes and dd prints that it's done reading 8192 bytes.
But driver is doing something more.

This is what Read-Ahead mechanism does.
A disk has platters. When disk read a sector-0, it spins platter to find the sector-0.
It takes longer time to find a specific sector than reading data in the sector.
So kernel always try to read more data around the specified sector and store the data into memory.
If user want to read more, kernel doesn't need to read disk but memory.
If you want to know more detail, please refer to data locality: https://en.wikipedia.org/wiki/Locality_of_reference
Nevertheless kernel usually read more data than user requests.

Do you remember that driver updates statistics?
Let's check stat file and run iostat tool that read stat file and print the contents

```
/ # cat /sys/block/mybrd/stat 
       2        0       96       15        2        0       16        3        0        9        9
/ # iostat
Linux 4.4.0+ ((none))     11/03/16     _x86_64_    (4 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.00    0.00    0.04    0.00    0.00   99.95

Device:            tps   Blk_read/s   Blk_wrtn/s   Blk_read   Blk_wrtn
mybrd             0.01         0.35         0.06         96         16
```

Please check what each value means in stat file: https://www.kernel.org/doc/Documentation/block/stat.txt
For example, first four-values are for reading: 2 I/O generated and 96 sectors were processed for 15 milli-second.
iostat tool also shows 96 sectors were processed in mybrd disk.
And next four-values mean that 2 writing I/O generated and 16 sectors were processed for 3 milli-second.

