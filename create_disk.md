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
Can you guess where the producer and consumer of request_queue are?
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

It's a little bit confusing. But the code is simple. We need only three code to handle sectors:

* Start number of the sectors: bio->bi_iter.bi_sector
* how many sectors: bio_sectors(bio)
* read or write: bio_rw(bio)

당연히 몇개의 섹터가 될지 모르지만 하나의 페이지 4096바이트보다 클 수 있겠지요? 그러니 bio_for_each_segment 매크로를 써서 각 bio_vec을 하나씩 꺼내오면 됩니다. 그러면 데이터를 읽고 써야할 페이지와 페이지 내부의 offset, 길이 정보를 알 수 있습니다. 설명도 깊고 많은 개념들이 나타나지만 코드로 보면 정작 중요한건 몇개 안된다는걸 알 수 있으실겁니다. 데이터 방향, 크기를 어떻게 표현하느냐입니다.

우리는 그냥 bio_vec의 정보만 출력했지만 사실은 뭐가 필요할까요? 바로 여기가 DMA 동작을 실행해야하는 부분입니다. DMA는 페이지 단위로 실행됩니다. 그래서 bio 구조체가 페이지 정보들을 가지고 있는 것입니다. 하나의 bio가 하나의 scatter-gatter DMA가 됩니다.

어쨌든 bio를 분석하면 어떤 페이지에 얼마만큼 읽기/쓰기를 해야하는지를 알 수 있습니다. 그리고 IO가 끝나면 해당 bio를 폐기처분해야합니다. bio객체도 어딘가에서 할당했을거니 당연히 해지가 필요하겠지요. 그런 일을 하는 함수가 bio_endio()입니다.

그리고 이 장치가 얼마만큼 읽기/쓰기를 했는지 등의 통계 정보를 갱신하는게 generic_start/end_io_acct()입니다. 통계 정보는 커널을 부팅해서 확인해보겠습니다.

마지막으로 BLK_QC_T_NONE을 반환합니다. 그냥 아무 문제 없었다는걸 알려주는 것입니다. 커널이 mybrd_make_request_fn을 호출했을테니 커널에게 문제 없음을 알려주는 것이지요.

더 자세한 설명은 https://www.kernel.org/doc/Documentation/block/biodoc.txt 를 참고하세요.

##bio 발생 실험

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
커널로부터 bio 정보가 잘 전달되고있는지 실험을 해보겠습니다.

커널을 부팅했더니 부팅 메세지에 위와같이 메시지가 나타납니다. mybrd_init()이 제대로 호출된걸 확인할 수 있고, 그 다음에 mybrd_alloc()에서 출력한 메세지들도 보이네요. 그리고 mybrd_make_request_fn()가 호출된 것도 보입니다. mybrd_make_request_fn()에서는 분명 bio의 정보들을 출력해야되는데 이상하게 IO 에러를 나타내는 메세지가 출력됩니다. 이게 뭘까요?

이전에 말씀드렸는데 add_disk()가 호출되서 커널이 gendisk를 인식한 즉시 해당 디스크에 IO가 발생할 수 있습니다. 우리 드라이버는 add_disk()를 호출한 다음 global_mybrd에 새로 할당된 mybrd_device 객체의 포인터를 저장합니다. 그말은 global_mybrd = mybrd_allod() 코드가 실행되기전에 디스크로 IO가 발생했다는 것입니다. 그리고 mybrd_make_request_fn()에서는 if(mybrd != global_mybrd) 코드로 분기해서 bio_io_error()를 호출합니다. 커널에게 해당 bio를 처리하다가 에러가 발생했다고 알려주는 것입니다. 그러니 커널은 IO가 실패했다고 메세지를 출력하게됩니다.

어쨌든 부팅이 완료됐으니 몇가지 실험을 해보겠습니다.

드라이버가 제대로 등록됐는지를 알아보기 위해 장치 파일을 확인하고, /sys/block/mybrd 디렉토리도 열어보겠습니다.
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
여기서 가장 중요한 파일은 stat파일입니다. 이 장치에 얼마만큼의 IO가 발생했는지를 확인하는 것입니다. size 파일도 있는데요 디스크의 크기가 저장된 파일입니다. cat /sys/block/mybrd/size 명령으로 출력해보세요. 그 외에 다른 파일들도 한번 출력해보세요.

그리고 또 열어볼게 queue 디렉토리입니다. 바로 request-queue의 정보를 가지고 있습니다.
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
다양한 파일들이 있습니다. 이 파일들의 자세한 정보는 커널 소스 내부에 있는 문서에서 확인할 수 있습니다. 커널 소스에 있는 Documentation 디렉토리가 커널 자체를 설명하는 다양한 문서들이 저장된 디렉토리입니다. 이중에서 Documentation/block 을 보면 블럭 장치에 대한 설명들이 있고, queue에 대한 설명은 queue-sysfs.txt파일에 있습니다. 다음 링크를 열어보면 웹으로도 볼수있습니다.

https://www.kernel.org/doc/Documentation/block/queue-sysfs.txt

예를 들어 logical_block_size파일에 4096이라는 값이 있는데, 우리가 드라이버 코드에서 blk_queue_logical_block_size() 함수로 설정한 값입니다. 이렇게 우리가 직접 설정한 queue에 대한 설정값들과 커널이 디폴트로 설정한 값들이 모여있습니다.

그럼 다음으로는 dd 툴을 이용해서 디스크에 데이터를 써보겠습니다. 다음은 mybrd 장치에 4096바이트씩 두번 0을 쓰는 실험입니다. 
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
첫번째 처리하는 bio는 0번째 섹터부터 8개의 섹터를 쓰는 bio입니다. 그리고 bio에 2개의 세그먼트가 존재하는데 각 세그먼트는 4096바이트입니다. 각 세그먼트가 저장된 페이지의 주소도 확인가능합니다. 또 세그먼트의 크기가 4096바이트라는 것은 곧 페이지 전체를 사용한다는 의미이므로 offset은 0입니다.

그리고 반대로 디스크에서 4096바이트씩 2번 읽어들이는 실험도 해봅니다.
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
뭔가 이상합니다. 두개의 bio가 처리되는데 첫번째는 4개의 세그먼트를 가지고 있고 총 32개의 섹터를 읽습니다. 두번째는 세그먼트가 더 많고, 총 64개의 섹터를 읽습니다. 분명 어플에서는 8192바이트를 읽습니다. dd값의 결과 메세지가 분명 8192바이트를 복사했다고 나오는데 커널에서는 더 많은 데이터를 읽습니다.

사실 이건 커널에서 데이터를 미리 읽어놓은 알고리즘이 있기 때문입니다. 디스크의 특성상 플래터를 돌려서 특정한 섹터를 찾는 시간이 오래걸리고, 한번 특정 섹터를 찾으면 데이터를 읽는 시간은 상대적으로 짧습니다. 그러니 한번 섹터를 찾았을 때 좀더 많이 읽어놓으면 다음에 비슷한 위치를 읽었을 때 디스크 IO없이 미리 읽어놓은 데이터를 바로 쓸 수 있습니다. 그리고 많은 연구결과 프로그램들이 특정한 위치의 데이터를 읽고나면 계속 그 다음 위치의 데이터를 읽을 확률이 높다는게 밝혀졌습니다. 이런 프로그램의 특성을 data locality라고 부릅니다. 어쨌든 읽기는 요청한 것보다 좀더 많은 IO가 발생한다는걸 알아두겠습니다. 

그리고 우리가 드라이버 코드에 통계 정보를 업데이트하는 함수를 호출한걸 기억하시나요? 과연 통계 정보가 갱신되고 있는지 stat파일을 출력해보겠습니다. iostat프로그램을 쓰면 자동으로 stat파일을 읽어줍니다.
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
https://www.kernel.org/doc/Documentation/block/stat.txt 이 문서를 보면 파일의 각 값들이 뭔지를 알수있습니다.

앞의 4개 숫자는 각각 2개의 IO가 발생했다는것과 0번의 IO 병합이 발생했고, 96개의 섹터가 15ms동안 처리되었다는 것을 말해줍니다. iostat 프로그램도 총 96섹터의 read가 발생했다고 알려줍니다.

뒤의 4개의 숫자는 쓰기에 대한 정보입니다. 총 2번의 IO가 발생했고 16개 섹터가 3ms동안 처리되었다는 것입니다.

더 자세한 설명을 해당 문서를 참고하세요.

