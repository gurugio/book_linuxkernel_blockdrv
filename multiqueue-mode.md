# multiqueue-mode

Yes, we finally arrived at the goal.
As the title shows, the goal of this document is making multiqueue-based block device driver.
Making the bio-mode and the request-mode are just basic steps for multiqueue-mode.
If you want to understand the latest block layer code, you should understand multiqueue-based IO processing.

Actually I don't understand every detail about the multiqueue-mode.
So I'll just describe briefly.
I hope this document could help you start other professional books and documents, such like "Understanding the Linux kernel".

Please refer following references to understand the background of the multiqueue-mode.
They inform you What kind of limitation the request-mode has, and what benefits the multiqueue-mode has.
You might not understand the documents at first.
But if you read the references again after implementing a simple example of the multiqueue-mode with this document, you would understand them better.

source: https://github.com/gurugio/mybrd/blob/master/mybrd.c

references

* https://www.thomas-krenn.com/en/wiki/Linux_Multi-Queue_Block_IO_Queueing_Mechanism_(blk-mq)
* https://lwn.net/Articles/552904/
* http://kernel.dk/blk-mq.pdf
* http://ari-ava.blogspot.de/2014/07/opw-linux-block-io-layer-part-4-multi.html 
* https://www.kernel.org/doc/Documentation/block/null_blk.txt
  * The example source mybrd.c is a mimic of null_blk driver because null_blk driver is the simplest but best real-world driver based on multiqueue block layer.

## create multiqueue-mode request-queue

(Let's call multiqueue-mode as mq-mode.)

As the name show, multiple software-queue (aka. sw-queue or submission queue) and hardware-queue (aka. hw-queue) are created for mq-mode IO processing.

Kernel creates sw-queues as many as the CPUs or nodes.
The sw-queue receives IO from the user application.
Each sw-queue works on its own CPU, so there is no lock racing and cache sync.
And the sw-queue sends IO to hw-queues.
Driver creates one or more hw-queues.
If the driver is for old hard-disk, it would create only one hw-queue.
But if the driver is for the state-of-the-art SSD that supports multi-channel, it would create several hw-queue.
Finally driver generates multi-thread to extract the requests from multi hw-queue.

It's hard to describe.
The best way to understand a mechanism is implementing it.

As you can see the paper, Linux Block IO: Introducing Multi-queue SSD Access on Multi-core Systems (http://kernel.dk/blk-mq.pdf), they developed null_blk driver to prove performance improvement of mq-mode.
The null_blk driver works without real device so that it can shows performance improvement only of block layer.
The mybrd driver applied core code of null_blk driver to make multi-queue based ramdisk.

### add mq-mode to mybrd_alloc()

As we added MYBRD_Q_RQ value for the request-mode, MYBRD_Q_MQ value is added for the mq-mode.
blk_mq_tag_set data struct is for kernel to manage the request-queue.
Driver decides how many hw-queues will be created because driver know the HW.
If HW supported two channel IO processing, driver would create two hw-queues.
And driver stores the number of hw-queue and other information into blk_mq_tag_set object.

Following is field of the blk_mq_tag_set that driver should initialize.
* ops: operation function of the multiqueue-based request-queue
* nr_hw_queues: the number of the hw-queues
* queue_depth: maximum number of the request each hw-queue can have
* numa_node: NUMA node number from which kernel allocate memory for hw-queue and etc
* cmd_size: size of driver specific information that will be passed along with the request
* driver_data: driver specific information for hw-queue

#### blk_mq_init_allocated_queue()

The multiqueue-based queue is created by blk_mq_init_queue() function.
Let's read the code of blk_mq_init_queue() briefly.

First it creates a queue with blk_alloc_queue_node() and initializes the queue with blk_mq_init_allocated_queue().
When blk_mq_init_allocated_queue() initializes the queue, both of sw-queues and hw-queues are initialized.
There are two data-structures, blk_mq_hw_ctx and blk_mq_ctx.
Objects of blk_mq_hw_ctx are created as many as specified by driver variable hw_queue_depth.
And objects of blk_mq_ctx are created as many as cores in the system, by alloc_percpu() function.
If there are many sw-queues and many hw-queues, blk_mq_init_allocated_queue() decides matching between sw-queues and hw-queues.
If there is only one hw-queue, all sw-queue should be matched to one hw-queue.
The matching policy is the map_queue field of struct blk_mq_ops.
Driver can make its own policy but mybrd uses default policy, blk_mq_map_queue, provided by kernel.

Following is some fields of the request-queue.
* queue_ctx: pointer to blk_mq_ctx representing sw-queue
* queue_hctxs: pointer to blk_mq_hw_ctx representing hw-queue
* mq_map: matching policy between sw-queue and hw-queue
* make_request_fn: pointer to the request handling function, initialized to blk_sq/mq_make_request by blk_queue_make_request()

blk_mq_init_cpu_queues() initializes sw-queue and matches sw-queue and hw-queue by map_queue function of blk_mq_ops.
blk_mq_init_hw_queues() initializes hw-queue and calls init_hctx function of blk_mq_ops.
Callback functions of blk_mq_ops is initialized by mybrd, so mybrd can initialize hw-queue and sw-queue for itself.

#### blk_sq_make_request() and blk_mq_make_request()

Kernel initialized a bio processing function for the request-mode.
For the request-mode, kernel calls the bio processing function to extract bios and merge them into a request.
And then kernel calls the request handling function of driver.

The bio processing function the the mq-mode is blk_sq_make_request and blk_mq_make_request for single hw-queue and multi hw-queues respectively.
Kernel extracts bio from the queue by blk_sq/mq_make_request() and make a request by blk_mq_map_request.
Then kernel passes the request to hw_queue of driver with blk_mq_run_hw_queue().

#### blk_mq_map_request()

When bio is in sw-queue, blk_mq_map_request() function allocates a request for the bio and decides what hw-queue will receive the request via q->mq_ops->map_queue().
mp_ops has callback functions provided by driver object mybrd_mq_ops.
mybrd driver sets mybrd_mq_ops.map_queue to blk_mq_map_queue.
So kernel decides how sw-queue and hw-queue are matched.

Let's see the code of blk_mq_map_queue().
```
/*
 * Default mapping to a software queue, since we use one per CPU.
 */
struct blk_mq_hw_ctx *blk_mq_map_queue(struct request_queue *q, const int cpu)
{
    return q->queue_hw_ctx[q->mq_map[cpu]];
}
EXPORT_SYMBOL(blk_mq_map_queue);
```

cpu value has the core number on which the current thread is executed.
queue_hw_ctx is an array of blk_mq_hw_ctx objects.
The size of array is specified by driver.
q->mq_map is an array of the number of hw_queue.
If q->mq_map array is {0,0,0,0}, there are 4 sw-queues and 1 hw-queue. 
All are matched to hw-queue 0.
If q->mq_map array is {0,0,1,1}, there are 4 sw-queues and 2 hw-queues.
{0,1} sw-queue are matched to hw-queue {0,0} and 2,3 sw-queues are matched to {1,1}.

Of course, you can make your own policy.

#### blk_mq_run_hw_queue

blk_mq_map_request() creates a request and decides which hw-queue will take the request.
blk_mq_run_hw_queue() transfers the request from sw-queue to hw-queue.

Actually blk_mq_run_hw_queue only calls callback function of driver, for instance mybrd_mq_ops.queue_rq of mybrd driver.
Request processing is done by driver that is the same for request-mode and bio-mode.

## callback functions in mybrd_mq_ops implementating hw-queue

Let's check what callback functions in mybrd_mq_ops.

### mybrd_init_hctx

As the name shows, it initializes hw-queue.
An argument for blk_mq_hw_ctx is a ponter to blk_mq_hw_ctx object of each hw-queues.
An argument for data is a pointer to mybrd object stored in tag_set.driver_data.
And index is the hw-queue number.
mybrd driver creates only one hw-queue, so index should be 0.

mybrd driver pass a object of struct mybrd_hw_queue_private to the hw-queue.
It contains a queue number and address of mybrd object.

### mybrd_queue_rq

A pointer to the request is stored at blk_mq_queue_data->rq.
And blk_mq_rq_to_pdu() returns a specific data of each requests.
The size of the request specific data is specified by tag_set.cmd_size value.

Processing a request is the same to the request-mode.

### mybrd_softirq_done_fn

Request handling of softirq mode also is the same to softirq mode of the request-mode.

## Kernel booting

```
[    0.298833] mybrd: start init_hctx: hctx=ffff88000643c400 mybrd=ffff88000771c600 priv[0]=ffff880006871eb0
[    0.299741] mybrd: info hctx: numa_node=0 queue_num=0 queue->ffff880007750000
[    0.300318] mybrd: end init_hctx
[    0.300668] mybrd: start queue_rq: request-ffff880007740000 priv-ffff880006871eb0 request->special=          (null)
[    0.301506] CPU: 0 PID: 1 Comm: swapper/0 Not tainted 4.4.0+ #74
[    0.301961] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS Ubuntu-1.8.2-1ubuntu1 04/01/2014
[    0.302707]  ffff880006871eb0 ffff88000750b848 ffffffff8130dadf ffff880007740000
[    0.303310]  ffff88000750b868 ffffffff814faa83 ffff88000643c400 ffff88000750b890
[    0.303927]  ffff88000750b8f0 ffffffff812f95b9 ffff880007750000 ffff88000643c408
[    0.304544] Call Trace:
[    0.304744]  [<ffffffff8130dadf>] dump_stack+0x44/0x55
[    0.305138]  [<ffffffff814faa83>] mybrd_queue_rq+0x43/0xb0
[    0.305582]  [<ffffffff812f95b9>] __blk_mq_run_hw_queue+0x1b9/0x340
[    0.306079]  [<ffffffff812f93e9>] blk_mq_run_hw_queue+0x89/0xa0
[    0.306552]  [<ffffffff812faa2f>] blk_sq_make_request+0x1bf/0x2b0
[    0.307027]  [<ffffffff812efc8e>] generic_make_request+0xce/0x1a0
[    0.307520]  [<ffffffff812efdc2>] submit_bio+0x62/0x140
[    0.307924]  [<ffffffff8119fa78>] submit_bh_wbc.isra.38+0xf8/0x130
[    0.308569]  [<ffffffff8119fdef>] block_read_full_page+0x24f/0x2f0
[    0.309050]  [<ffffffff811a1e50>] ? I_BDEV+0x10/0x10
[    0.309442]  [<ffffffff8111a96b>] ? __add_to_page_cache_locked+0x11b/0x1b0
[    0.309996]  [<ffffffff811a2820>] ? blkdev_readpages+0x20/0x20
[    0.310452]  [<ffffffff811a2833>] blkdev_readpage+0x13/0x20
[    0.310905]  [<ffffffff8111ad08>] do_read_cache_page+0x78/0x1a0
[    0.311377]  [<ffffffff8111ae44>] read_cache_page+0x14/0x20
[    0.311834]  [<ffffffff81301358>] read_dev_sector+0x28/0x90
[    0.312248]  [<ffffffff813043f6>] read_lba+0x126/0x1e0
[    0.312660]  [<ffffffff81304ab7>] efi_partition+0xe7/0x720
[    0.313063]  [<ffffffff8131763b>] ? string.isra.4+0x3b/0xd0
[    0.313508]  [<ffffffff81318fec>] ? vsnprintf+0x24c/0x510
[    0.313904]  [<ffffffff81319339>] ? snprintf+0x39/0x40
[    0.314281]  [<ffffffff813049d0>] ? compare_gpts+0x260/0x260
[    0.314740]  [<ffffffff813025e9>] check_partition+0x139/0x220
[    0.315157]  [<ffffffff81301b63>] rescan_partitions+0xb3/0x2a0
[    0.315627]  [<ffffffff811a2e22>] __blkdev_get+0x282/0x3b0
[    0.316031]  [<ffffffff811a3ae2>] blkdev_get+0x112/0x300
[    0.316418]  [<ffffffff81185fae>] ? unlock_new_inode+0x3e/0x70
[    0.316909]  [<ffffffff811a26fc>] ? bdget+0x10c/0x120
[    0.317288]  [<ffffffff814dbc32>] ? put_device+0x12/0x20
[    0.317728]  [<ffffffff812ff921>] add_disk+0x3e1/0x470
[    0.318225]  [<ffffffff812ffbc2>] ? alloc_disk_node+0x102/0x130
[    0.318738]  [<ffffffff81f7d865>] ? brd_init+0x153/0x153
[    0.319132]  [<ffffffff81f7da76>] mybrd_init+0x211/0x277
[    0.319562]  [<ffffffff810003b1>] do_one_initcall+0x81/0x1b0
[    0.319983]  [<ffffffff81f3b08e>] kernel_init_freeable+0x158/0x1e3
[    0.320443]  [<ffffffff81889b00>] ? rest_init+0x80/0x80
[    0.320876]  [<ffffffff81889b09>] kernel_init+0x9/0xe0
[    0.321260]  [<ffffffff8188f50f>] ret_from_fork+0x3f/0x70
[    0.321736]  [<ffffffff81889b00>] ? rest_init+0x80/0x80
[    0.322128] mybrd: queue-rq: req=ffff880007740000 len=4096 rw=READ
[    0.322615] mybrd:     sector=0 bio-info: len=4096 p=ffffea0000000940 offset=0
[    0.323091] mybrd: lookup: page-          (null) index--1 sector-0
[    0.323501] mybrd: copy: ffff880000025000 <- 0 (4096-bytes)
[    0.323903] mybrd: 0 0 0 0 0 0 0 0
[    0.324114] mybrd: end queue_rq
[    0.324487] mybrd: start queue_rq: request-ffff880007740000 priv-ffff880006871eb0 request->special=          (null)
[    0.325273] CPU: 0 PID: 1 Comm: swapper/0 Not tainted 4.4.0+ #74
[    0.325840] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS Ubuntu-1.8.2-1ubuntu1 04/01/2014
[    0.326619]  ffff880006871eb0 ffff88000750b8e8 ffffffff8130dadf ffff880007740000
[    0.327199]  ffff88000750b908 ffffffff814faa83 ffff88000643c400 ffff88000750b930
[    0.327923]  ffff88000750b990 ffffffff812f95b9 ffff880007750000 ffff88000643c408
[    0.328538] Call Trace:
[    0.328723]  [<ffffffff8130dadf>] dump_stack+0x44/0x55
[    0.329106]  [<ffffffff814faa83>] mybrd_queue_rq+0x43/0xb0
[    0.329570]  [<ffffffff812f95b9>] __blk_mq_run_hw_queue+0x1b9/0x340
[    0.330038]  [<ffffffff812f93e9>] blk_mq_run_hw_queue+0x89/0xa0
[    0.330517]  [<ffffffff812faa2f>] blk_sq_make_request+0x1bf/0x2b0
[    0.330974]  [<ffffffff812efc8e>] generic_make_request+0xce/0x1a0
[    0.331430]  [<ffffffff812efdc2>] submit_bio+0x62/0x140
[    0.331857]  [<ffffffff8119fa78>] submit_bh_wbc.isra.38+0xf8/0x130
[    0.332321]  [<ffffffff8119fdef>] block_read_full_page+0x24f/0x2f0
[    0.332830]  [<ffffffff811a1e50>] ? I_BDEV+0x10/0x10
[    0.333227]  [<ffffffff8111a96b>] ? __add_to_page_cache_locked+0x11b/0x1b0
[    0.333795]  [<ffffffff811a2820>] ? blkdev_readpages+0x20/0x20
[    0.334229]  [<ffffffff811a2833>] blkdev_readpage+0x13/0x20
[    0.334679]  [<ffffffff8111ad08>] do_read_cache_page+0x78/0x1a0
[    0.335118]  [<ffffffff8111ae44>] read_cache_page+0x14/0x20
[    0.335575]  [<ffffffff81301358>] read_dev_sector+0x28/0x90
[    0.335985]  [<ffffffff81302783>] amiga_partition+0x53/0x410
[    0.336405]  [<ffffffff81303fe0>] ? sgi_partition+0x190/0x190
[    0.336863]  [<ffffffff81304298>] ? sun_partition+0x2b8/0x2f0
[    0.337288]  [<ffffffff81303e3a>] ? osf_partition+0x15a/0x170
[    0.337937]  [<ffffffff81302730>] ? put_partition+0x60/0x60
[    0.338394]  [<ffffffff813025e9>] check_partition+0x139/0x220
[    0.338863]  [<ffffffff81301b63>] rescan_partitions+0xb3/0x2a0
[    0.339300]  [<ffffffff811a2e22>] __blkdev_get+0x282/0x3b0
[    0.339805]  [<ffffffff811a3ae2>] blkdev_get+0x112/0x300
[    0.340224]  [<ffffffff81185fae>] ? unlock_new_inode+0x3e/0x70
[    0.340696]  [<ffffffff811a26fc>] ? bdget+0x10c/0x120
[    0.341093]  [<ffffffff814dbc32>] ? put_device+0x12/0x20
[    0.341518]  [<ffffffff812ff921>] add_disk+0x3e1/0x470
[    0.341924]  [<ffffffff812ffbc2>] ? alloc_disk_node+0x102/0x130
[    0.342387]  [<ffffffff81f7d865>] ? brd_init+0x153/0x153
[    0.342814]  [<ffffffff81f7da76>] mybrd_init+0x211/0x277
[    0.343230]  [<ffffffff810003b1>] do_one_initcall+0x81/0x1b0
[    0.343688]  [<ffffffff81f3b08e>] kernel_init_freeable+0x158/0x1e3
[    0.344174]  [<ffffffff81889b00>] ? rest_init+0x80/0x80
[    0.344590]  [<ffffffff81889b09>] kernel_init+0x9/0xe0
[    0.344996]  [<ffffffff8188f50f>] ret_from_fork+0x3f/0x70
[    0.345417]  [<ffffffff81889b00>] ? rest_init+0x80/0x80
[    0.345839] mybrd: queue-rq: req=ffff880007740000 len=4096 rw=READ
[    0.346326] mybrd:     sector=8 bio-info: len=4096 p=ffffea0000000980 offset=0
[    0.346971] mybrd: lookup: page-          (null) index--1 sector-8
[    0.347462] mybrd: copy: ffff880000026000 <- 0 (4096-bytes)
[    0.347905] mybrd: 0 0 0 0 0 0 0 0
[    0.348176] mybrd: end queue_rq
```

In the booting log, we can see mybrd_init_hctx() was called with hw-queue number 0.
When a disk was added, kernel generates two requests to check the disk.
init_hctx() set the priv value as ffff880006871eb0 and each request has priv value ffff880006871eb0.
So we can confirm that the requests are handled by the hw-queue that is initialized by init_hctx().

Please run other tools, dd and mkfs.ext4, to test mybrd disk.
