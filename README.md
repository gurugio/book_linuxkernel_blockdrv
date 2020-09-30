_First, thank you for taking the time to read my book! I want to point out that English is not my first language and you're welcome to give me any feedback for my poor English._

# Multi-queue block device in Linux kernel v4.4

Several years ago, a new concept was merged into the block layer of Linux kernel. Before that every single block device has one queue for IO handling. Every processes inserted an IO request into the queue and block device driver extract a request from the queue. Yes, one queue was shared for many processes and for many processors.

When we used HDD mainly, a single queue did not matter. But these days SSD is so popular that a single queue design has been bottle-neck of performance. Therefore kernel developers implemented the multi-queue design.

That is not just an adding more queues. The architecture of block layer must've re-designed. You can get the theoritical background from this paper: 

> Bjørling, Matias, et al. "Linux block IO: Introducing multi-queue SSD access on multi-core systems." Proceedings of the 6th International Systems and Storage Conference. ACM, 2013. - http://kernel.dk/systor13-final18.pdf

This document shows the step-by-step process of making mybrd driver that is mimic of brd and null_blk drivers in Linux v4.4. We begin with dummy skeleton driver. And we will make a single queue and see how it works. And also we will see how kernel pass the IO request to driver via the single queue. Finally we will change mybrd driver to have multi-queue and see how it works between kernel block layer and driver.

The final source code is already implemented at https://github.com/gurugio/mybrd/blob/master/mybrd.c. So you can see what this document aim to do now. I'll also describe a few more features of kernel because they are necessary to understand and implement the block device driver.

I hope this document can guide you to the deep inside of Linux kernel.


PS.

This document is not for very beginner of Linux kernel. A small document cannot describe details beginner should know to start Linux kernel. If you already started Linux kernel and read one or two books, but did not know what to do next, this document can be good for you.

PS.

If you are interested in memory management, you'd better start reading Mel Gorman's book.

https://www.kernel.org/doc/gorman/pdf/understand.pdf


# INDEX

* [prepare](environment.md)
* [skeleton of mybrd](mybrd_skeleton.md)
* [disk](create_disk.md)
* [ramdisk](create_ramdisk.md)
* [request-mode](request-mode.md)
* [multiqueue-mode](multiqueue-mode.md)
* [pagecache and blockdriver](pagecacheand_blockdriver.md)
* [experiement for the page cache](pagecache_ex.md)
* [page-flags](page-flags.md)
* [system calls and flushing block device](systemcall_flushblock.md)
* [per-cpu variable and statistics (v2.6.11)](per-cpu_statistics.md)
* [wait-queue](wait-queue.md)
* [spinlock](spinlock.md)
* [work-queue](work-queue.md)
* [RCU](read-copy-update.md)


# references

* Documentation/block/
* https://www.thomas-krenn.com/en/wiki/Linux_Storage_Stack_Diagram#Diagram_for_Linux_Kernel_4.10

