# experiement

I made a application program to test the page cache with mybrd device.
And I added some code to mybrd_ioctl() of mybrd driver.

source: https://github.com/gurugio/mybrd/blob/ex-pagecache/mybrd.c

## ioctl of mybrd

Following is mybrd_ioctl() that is added for this chapter.

```
static int mybrd_ioctl(struct block_device *bdev, fmode_t mode,
                        unsigned int cmd, unsigned long arg)
{
        int error = 0;
        struct inode *in = bdev->bd_inode;
        struct address_space *asp = in->i_mapping;
        struct radix_tree_root *root = &asp->page_tree;
        struct page *p;
        unsigned long index = 0;

        pr_warn("asp: nrpages=%d host=%p\n",
                (int)asp->nrpages, asp->host);
        pr_warn("radix-tree: height=%d rnode=%p rnode-count=%d\n",
                root->height, root->rnode, root->rnode ? root->rnode->count:-1);

        for (index = 0; index < 10; index++) {
                p = radix_tree_lookup(root, index);
                if (p)
                        pr_warn("index=%d page=%p pindex=%d\n",
                                (int)index, p, (int)p->index);
                else
                        pr_warn("no page-cache index=%d\n", (int)index);
        }

        for (index = 0; index < 10; index++) {
                struct buffer_head *bh[10];
                bh[index] = __bread(bdev, index, PAGE_SIZE);
                pr_warn("bh: index=%d state=%x page=%p blocknr=%d size=%d\n",
                        (int)index, (int)bh[index]->b_state,
                        bh[index]->b_page, bh[index]->b_blocknr, bh[index]->b_size);
        }

        return error;
}
```

We don't use cmd value.
Whenever user application calls ioctl, mybrd driver prints information of pages in radix-tree.
And mybrd driver prints information of buffer heads for mybrd device.
Mybrd driver uses ``__bread()`` to find buffer heads of mybrd device.
``__bread()`` function gets block device, number of block and size of block, and returns buffer head including the index.
If there is no such block in the page cache, it adds new page into the page cache and creates new buffer head and returns.

## application

Let's look into the application.
It reads 10K data from mybrd disk.
And it calls ioctl with command value 0x1234.
Actually the command value is meaningless.
Mybrd driver does not care the comman value.
Next the application writes 10K data and calls ioctl.
Finally it closes the device file.

What we should care is how the page cache works if application read/write a block device.
Let's build this file with ``gcc a.c -static`` command and run on qemu VM.
Please refer following document for setup qemu VM.
* https://github.com/gurugio/book_linuxkernel_blockdrv/blob/master/environment.md

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

## result

Following is the result of the program.

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

mybrd_ioctl function prints pages in the page cache and buffer heads.
Yes, the same pages are included in the page cache and linked to buffer heads.

After writing data, the state of buffer head was changed from 0x21 to 0x23.
Following is the flags of buffer head.
```
enum bh_state_bits {
	BH_Uptodate,	/* Contains valid data */
	BH_Dirty,	/* Is dirty */
	BH_Lock,	/* Is locked */
	BH_Req,		/* Has been submitted for I/O */
	BH_Uptodate_Lock,/* Used by the first bh in a page, to serialise
			  * IO completion of other buffers in the page
			  */

	BH_Mapped,	/* Has a disk mapping */
	BH_New,		/* Disk mapping was newly created by get_block */
	BH_Async_Read,	/* Is under end_buffer_async_read I/O */
	BH_Async_Write,	/* Is under end_buffer_async_write I/O */
	BH_Delay,	/* Buffer is not yet allocated on disk */
	BH_Boundary,	/* Block is followed by a discontiguity */
	BH_Write_EIO,	/* I/O error on write */
	BH_Unwritten,	/* Buffer is allocated on disk but not written */
	BH_Quiet,	/* Buffer Error Prinks to be quiet */
	BH_Meta,	/* Buffer contains metadata */
	BH_Prio,	/* Buffer should be submitted with REQ_PRIO */
	BH_Defer_Completion, /* Defer AIO completion to workqueue */

	BH_PrivateStart,/* not a state bit, but the first bit available
			 * for private allocation by other entities
			 */
};
```

0x21 is BH_Uptodate and BH_Mapped that means data of the buffer head is copied from disk and the valid data.
After writing, BH_Dirty flag is added because page data is changed and should be flushed into the disk.

Please remember that after application terminated, mybrd block device has no page cache because closing the block device file flushes all page caches.
free command shows the same status before/after the application.
We cannot check how much page cache mybrd generates with free command.
That is why I made an application and ioctl() handler in mybrd driver.
