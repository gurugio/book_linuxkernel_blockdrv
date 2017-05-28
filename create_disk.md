# create a disk

Let's start developing a block device.
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

request-queue라는걸 만듭니다. 이것이 뭐냐면 request가 저장되는 queue입니다. 큐는 특정한 데이터 객체들이 한쪽으로 저장되고 한쪽으로 빠져나가는 걸 말하지요. 결국 request라는게 저장되고 빠져나가는 큐입니다. request는 다음 mybrd_make_request_fn() 함수에서 처리하게됩니다.

일단 여기에서는 request-queue라는 객체를 만들고, 그 객체의 각 필드를 초기화한다는걸 생각하면 됩니다. 각 세부 필드를 하나씩 설명하려면 한꺼번에 외울게 너무 많아지니 생략합니다.

Let me introduce a request_queue object of struct request_queue.
Queue is a data structure to store data on one side and extract data on another side.
So request_queue is a queue to store requests.
Requests in the queue is extracted and consumed by mybrd_make_request_fn() function as I will introduce later.

At this point, you should know there is a queue so-called request-queue and it should be initialized as source code does.
There are many fields in struct request_queue.
You don't need to understand all of the fields right now.
Let's skip the detail and just use the code as it is.

하나만 알고 넘어가면 됩니다. blk_queue_make_request()가 request-queue라는 객체와 mybrd_make_request_fn()함수를 연결한다는 것만 생각하면 됩니다. 큐가 있으면 큐에 데이터를 넣는 코드가 있고 빼는 코드가 있고, 그리고 큐에서 빼낸 데이터를 처리하는 코드가 있겠지요. 커널은 이 request-queue에 request라는걸 넣고 빼주는걸 알아서 해줍니다. 왜냐면 모든 드라이버에 공통적인 코드이기 때문입니다. 커널은 결국 드라이버를 위한 프레임워크 역할을 합니다. 만약 모든 드라이버에 공통적으로 사용될만한 코드가 있으면 커널 개발자들은 반드시 커널 함수로 구현해놓습니다. 그리고 드라이버 개발자가 이용하도록 합니다.  드라이버 개발이 편하라는 이유도 있지만 더 큰 이유는 버그를 줄이기 위해서입니다. 드라이버 개발자들이 해야할 일이 많을 수록 드라이버는 더 불안정해지겠지요. 커널 코드는 전세계 커널 개발자들이 리뷰하고 테스트합니다. 또 전세계 수많은 스마트폰, 서버 등의 머신에서 돌아가면서 자동으로 테스트되고 버그 리포팅이 됩니다. 하지만 드라이버는 특정 회사에서 개발하게 되고, 결국 커널의 약점은 대부분 드라이버가 됩니다. 따라서 최대한 드라이버 코드가 커널 함수를 많이 쓸수록 더 안정적인 드라이버가 되고, 그게 결국 운영체제 전체의 안정성을 높여줍니다.

request-queue객체를 만들고 해지하는 커널 함수가 따로있는 것도 메모리 할당만 하면 되는게 아니라, 그 외에 request-queue 객체의 많은 필드들을 초기화해야하는데, 그런 초기화들이 드라이버마다 다른게 아니라 공통적이기 때문입니다. 많은 커널 객체들이 할당 함수를 따로 가지고 있습니다. 그럴때는 반드시 할당 함수를 사용하고 kmalloc등을 써서 직접 객체를 생성하고 초기화하지 않도록 주의해야합니다.

커널이 큐를 관리해주므로 우리가 만들건 결국 큐에서 빠져나온 request를 분석해서 뭘 할지를 결정하는 것입니다. 만약 우리가 하드디스크 드라이버의 개발자라면 request를 분석해서 메모리 어디에 있는 데이터를 읽거나 쓰면 되는지 확인해서, 메모리의 데이터를 하드디스크로 보내거나 디스크의 데이터를 메모리로 가져오기면 하면 됩니다. request 객체에 대한 설명도 자연스럽게 나왔습니다. request 객체는 결국 메모리 어디에 얼마만큼의 데이터를 읽거나 쓰라는 정보를 가지는 객체입니다. 직접 mybrd_make_request_fn()함수를 만들어보면 이해가 될 것입니다.

####gendisk object

request-queue다음에는 gendisk라는 객체를 만듭니다. gendisk는 이름 그대로 디스크를 표현하는 객체입니다. 마찬가지로 전용 할당 함수 alloc_disk()가 있습니다. alloc_disk()의 인자는 1인데 1개의 디스크를 만든다는게 아니라 디스크에 최대 1개의 파티션이 있다는 것입니다. 그냥 디스크를 파티션으로 나누는게 아닐때 1로 지정합니다.

코드를 보면 디스크의 major number, minor number를 설정합니다. 왜냐면 이 디스크를 위한 장치 파일이 생성되기 때문입니다. first_minor를 111로 설정했습니다. 나중에 실험해보면 바로 장치 파일의 minor number가 111인걸 확인할 수 있습니다. 

이 디스크에서 사용할 request-queue가 방금 우리가 생성한 rq라는 것을 설정합니다. 그림고 디스크의 이름과 크기를 지정합니다. 디스크의 크기는 섹터 단위입니다. 한 섹터는 512바이트이므로 바이트 단위 숫자를 512로 나눠서 지정합니다.

fops필드가 있습니다. 바로 디스크의 장치 파일을 열고 사용할 때 호출될 함수입니다. 커널에서 struct block_device_operations의 정의를 찾아보면 read, write, open, close, ioctl 등 익숙한 이름들이 있습니다. 바로 어플리케이션에서 장치 파일을 가지고 read(), write(), open(), close(), ioctl() 등의 시스템 콜을 호출할 때 드라이버가 등록한 함수들이 호출되도록 만든 것입니다. mybrd는 /dev/mybrd 장치 파일을 만듭니다. 우리는 ioctl() 시스템 콜에 mybrd_ioctl()함수가 호출되도록 등록했으니, 나중에 장치 파일을 열고 ioctl()을 호출하는 어플을 만들어서 실험해보시기 바랍니다.

커널의 콜 스택을 분석할때 dump_stack() 함수가 편리합니다. 현재까지의 콜 스택을 커널 메세지로 출력하는 함수입니다. mybrd_ioctl()함수에서 dump_stack()을 호출하면 어플의 시스템 콜이 어떻게 커널에서 처리되는지 파악하기 편하겠지요.

gendisk객체를 만들었으면 커널에 새로운 디스크를 생성하라고 알려줘야합니다. 그게 바로 add_disk()함수입니다. 사실 request-queue를 만들긴 했지만, 커널에 드라이버가 만든 request-queue를 알려주는 함수는 없습니다. 바로 gendisk 객체에 request-queue를 등록하면 커널은 gendisk에 접근할 때마다 이 디스크가 사용할 request-queue가 뭔지 알게되는 것이지요. 따라서 블럭 장치의 가장 핵심 객체가 바로 gendisk이고, 이 핵심 객체를 커널이 사용하도록 등록하는게 add_disk()함수입니다.

add_disk()가 호출된 후에는 /dev/mybrd 파일이 생깁니다. /sys/block/mybrd 디렉토리도 생깁니다. 그리고 커널은 새로 디스크가 연결된걸 알고, 디스크를 읽어봅니다. add_disk()가 호출된 순간부터 디스크로 IO가 발생합니다. 그러니 add_disk()를 호출하기 전에 IO를 처리할 모든 준비가 끝나야겠지요.

##bio 구조체
드라이버가 request-queue를 만들고, 커널이 IO 요청을 request로 만들어서 큐를 통해서 전달한다고 말씀드렸지요. 그런데 이번 장에서 바로 이 request 객체를 처리하는걸 만들어보지는 않을겁니다. 그것보다 더 간단한 자료구조인 bio를 소개하고, bio를 기준으로 IO를 처리하는걸 구현해보겠습니다.

bio가 뭐냐면 IO를 처리하는데 있어서 최소단위가 되는 구조체입니다. request 객체는 결국 여러개의 bio를 가지게됩니다. 왜 bio 처리를 먼저 만들어보냐면 IO의 최소단위이므로 커널이 IO를 처리하는 과정을 그대로 볼 수 있기 때문입니다. 나중에 request 단위로 처리하는걸 만들면 bio의 merge라는 등 더 복잡한 처리가 소개되니까 일단 가장 기본 단위부터 만들어보는게 이해하기 좋겠지요.

###struct bio & struct bio_vec
디스크 IO가 일어나려면 기본적으로 얼마만큼의 데이터를 어디에서 가져와야된다는 정보가 있어야합니다. 그 정보가 바로 struct bio 구조체로 표현됩니다.

그리고 섹터라는게 나옵니다. 바로 디스크 IO 최소단위입니다.  디스크에 한번에 쓰고 읽을 수 있는 단위가 바로 섹터입니다. 하드디스크를 상상해볼까요. 둥그런 플래터가 빙빙돌고 있습니다. 바로 그 플래터의 한 부분을 섹터라고 부르고, 크기는 512바이트입니다.

https://ko.wikipedia.org/wiki/%ED%95%98%EB%93%9C_%EB%94%94%EC%8A%A4%ED%81%AC_%ED%94%8C%EB%9E%98%ED%84%B0

디스크를 둥근 플래터로 생각하지말고 섹터의 배열이라고 생각하봅시다. 그럼 몇번째 섹터부터 몇개의 섹터를 읽고 쓸거냐라는게 왜 IO의 기준이 되는지 이해가 될겁니다. 물론 하드디스크냐 SSD냐 등등 섹터만 기준이 되는게 아닙니다. 헤더니 실린더니 하드디스크에서 어느 위치냐를 지정하는데는 더 복잡한 방법이 사용됩니다. 실제 장치 드라이버를 만들려면 장치의 기계적인 특성을 모두 알아야겠지만 우리는 가상의 장치만 생각하겠습니다. 이 장치는 섹터가 한줄로 길게 늘어선 모양이고 섹터의 배열로 이루어져있습니다.

사용자 어플을 생각해보세요. read()/write() 시스템 콜을 보면 파일의 어느부분부터 몇 바이트를 읽고 쓰겠다는 명령을 내립니다. 드라이버는 몇번 섹터부터 몇개의 섹터를 읽고 쓰라는 명령을 받습니다. 중간에 있는 커널의 블럭 레이어가 바로 파일의 위치, 크기를 섹터위치, 섹터 갯수로 변환해주는 일을 합니다. 드라이버는 결국 섹터만 생각하면 됩니다.

그리고 섹터 정보가 모여있는게 bio입니다. 

http://www.makelinux.net/books/lkd2/ch13lev1sec3

여기에 좋은 그림이 있네요. 하나의 bio에는 여러개의 bio_vec가 들어있고, 각 bio_vec에는 어떤 페이지의 어떤 위치에 몇 바이트라는 정보가 있습니다. 이 bio_vec이 세그먼트를 표현하는 구조체입니다.

세그먼트니 섹터니 bio니 알아야될 개념이 많아지고 어지러워집니다. 그런데 코드를 보면 간단합니다. 복잡하게 생각할 것 없이 바로 코드를 보면서 이해하시면 됩니다.

* 몇번 섹터부터 - bio->bi_iter.bi_sector
* 몇개의 섹터를 - bio_sectors(bio)
* 읽을거냐 쓸거냐 - bio_rw(bio)

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

