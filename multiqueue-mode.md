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
* queue_ctx: sw-queue에 대한 정보
* queue_hctxs: hw-queue에 대한 정보
* mq_map: sw-queue와 hw-queue가 어떻게 매칭될지에 필요한 정보. 예를 들어 2개의 sw-queue와 1개의 hw-queue가 만들어졌으면 모든 sw-queue의 request들이 하나의 hw-queue로 전달되야합니다. 2:2로 매칭될수도 있고, 4:1, 4:2 등등 드라이버가 결정하기 나름입니다. mq_map이 이런 매칭을 결정하는게 아니고 드라이버가 제공한 함수에서 결정하는데, mq_map은 그런 결정에 필요한 정보들을 가지고 있습니다. (디록트로 커널이 제공하는 함수도 있고 우리는 커널 함수를 쓰겠습니다.)
* make_request_fn: blk_queue_make_request() 함수를 써서 hw-queue의 갯수에 따라 bio 처리 함수를 다르게 지정합니다.  

그리고 마지막으로 blk_mq_init_hw_queues() 함수를 호출하는데, 여기에서 mybrd_mq_ops.init_hctx 콜백 함수가 호출됩니다. 각 hw-queue 를 초기화하는 함수입니다.

blk_mq_init_queue() 함수로 sw-queue와 hw-queue를 초기화했으니 disk를 만듭니다. disk 생성은 큐와 상관없이 항상 동일합니다.

####blk_sq_make_request()

request-mode에서도 마찬가지로 커널이 request-queue의 bio 처리 함수를 지정했습니다. request-mode에서는 bio 처리 함수에서 스케줄러를 호출한 뒤에 드라이버가 제공한 request 처리 함수를 호출했습니다. mq-mode에서는 어떻게 bio가 처리되는지 blk_sq_make_request()를 통해 간단히 보겠습니다.

큐와 bio를 전달받는 것은 익숙합니다. 그 다음은 blk_mq_map_request()로 request를 만들고, blk_mq_run_hw_queue()로 hw_queue에서 드라이버로 request를 전달합니다.

####blk_mq_map_request()

현재 bio는 sw-queue에 있습니다. 그리고 blk_mq_map_request()함수에서 어떤 hw-queue로 넘겨질지를 결정합니다.

blk_mq_map_request()함수는 q->mq_ops->map_queue() 함수를 호출합니다. 여기서 mq_ops는 드라이버가 초기화한 콜백 함수들을 가지고 있습니다. 드라이버 소스를 보면 mybrd_mq_ops.map_queue = blk_mq_map_queue 로 설정하고 있습니다. 드라이버가 직접 sw-queue와 hw-queue의 매칭을 결정하지않고 커널 함수에 위임했습니다.

blk_mq_map_queue를 보겠습니다. 딱 한줄이네요.
````
/*
 * Default mapping to a software queue, since we use one per CPU.
 */
struct blk_mq_hw_ctx *blk_mq_map_queue(struct request_queue *q, const int cpu)
{
    return q->queue_hw_ctx[q->mq_map[cpu]];
}
EXPORT_SYMBOL(blk_mq_map_queue);
````

cpu는 현재 코드가 실행중인 cpu입니다. queue_hw_ctx는 드라이버가 요청한 hw-queue 갯수만큼 blk_mq_hw_ctx 객체를 만들 것입니다. 따라서 q->mq_map 배열이 cpu 번호와 hw-queue의 번호를 연결해주는 역할을 하고 있네요. 물론 커널 함수를 안쓰고 드라이버가 직접 만든 함수를 써도 됩니다. 특정 CPU와 특정 hw-queue를 묶고 싶을때는 직접 매칭 함수를 만들면 되겠지요. 커널 함수는 단지 공평한 분산만을 생각합니다.

blk_mq_map_request()를 계속 보면 __blk_mq_alloc_request()를 호출해서 request 객체를 만듭니다.

####blk_mq_run_hw_queue

blk_mq_map_request() 함수는 결국 bio를 포함하는 request를 만들고 request가 어떤 hw-queue로 가야할지를 결정하는 함수였습니다. 그럼 이제 request를 hw-queue로 보내야겠지요. blk_mq_run_hw_queue()함수가 sw-queue에 있는 request를 hw-queue로 전달합니다.

blk_mq_run_hw_queue()는 결국 __blk_mq_run_hw_queue()를 호출합니다. hw-queue는 사실상 하는 일이 없습니다. 그냥 드라이버의 mybrd_mq_ops.queue_rq 포인터를 읽어서 함수를 호출하는 것이 대부분입니다. 당연히 mybrd_mq_ops.queue_rq 함수는 request를 받아서 장치를 읽고 쓰는 일을 해야겠지요. 그 일은 request-mode에서와 동일할 것입니다.

##hw-queue를 구현하는 mybrd_mq_ops의 각 함수들

드라이버가 큐를 만들때 등록한 함수들이 mybrd_mq_ops에 있습니다. 하나씩 보겠습니다.

###mybrd_init_hctx

이름처럼 hw-queue를 초기화할때 호출됩니다. 인자로 받는 blk_mq_hw_ctx 포인터는 각 hw-queue마다 생성되는 blk_mq_hw_ctx 객체의 주소입니다.

인자값 data는 tag_set.driver_data로 전달한 mybrd 객체의 주소입니다. 그리고 index는 hw-queue의 번호입니다. 우리는 hw-queue를 하나만 만들었으니 0일 것입니다.

드라이버가 각 hw-queue에 전달할 정보는 struct mybrd_hw_queue_private객체입니다. 별다른건 없고 큐의 번호와 mybrd객체의 주소를 전달합니다.

hctx->driver_data필드에 mybrd_hw_queue_private객체의 주소를 저장하고 종료합니다.

###mybrd_queue_rq
***아직 완성되지 않은 내용***

함수 인자로 blk_mq_hw_ctx를 받는 것은 이해가 되실 것입니다. blk_mq_queue_data는 사실 아직 뭔지 잘 모르겠습니다. 단지 blk_mq_queue_data->rq 필드에 request의 포인터가 저장돼있다는 것만 알고있습니다.

그리고 blk_mq_rq_to_pdu() 함수를 써서 request마다 고유한 드라이버 정보를 저장하고 있습니다. 이 정보의 크기는 tag_set.cmd_size에 설정한 값입니다. 아직 pdu가 무엇의 약자인지 드라이버가 이 영역을 맘대로 덮어써도 되는지는 파악이 안됐습니다.

그 외에는 request를 처리하는 것 뿐입니다. request-mode와 동일한 방식으로 처리합니다.



###mybrd_softirq_done_fn

softirq 모드로도 request처리를 구현할 수 있습니다. 방식은 request-mode와 동일하므로 따로 구현하지 않았습니다.

##커널 부팅 실험
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
부팅을 해보겠습니다. mybrd_init_hctx() 가 호출되었네요. hw-queue 번호가 0번인게 확인됐습니다.

그리고 디스크가 추가되니 커널이 어떤 디스크인가하고 읽어봤네요. request가 2개 발생했습니다. 각각 priv가 init_hctx()에서 설정한 값인지 확인해보니 맞네요. init_hctx()에서 제대로 hw-queue를 설정했음을 알 수 있습니다. request처리 자체는 request-mode와 동일하니 자세히 볼 필요는 없습니다.

mkfs.vfat이나 dd 툴을 써서 테스트해보는 것은 별로 새로운게 없으니 각자 해보시기 바랍니다.

PS. 한가지 아쉬운건 request가 여러 sw-queue중 어느 큐에서 온건지를 알아보고싶은데, request에서 어느 필드를 봐야할지 잘 모르겠습니다. 이제 막 읽기 시작했으니 좀더 봐야지요.


