# request-mode

So far we've looked into bio-based IO.
Our mybrd driver receives a bio one by one from kernel and processes it in order.
As I explained before, bio is the least unit of IO.
Of course, there is bigger unit than bio.
It is the request.

In previous chapters, I explained that request-queue is a queue in which requests are stored.
But we didn't use the request at all.
We just exported a bio-handler function for kernel and had kernel call it with bio.
Fianlly we will use request-queue with request in this chapter.

Again, bio is a set of sequential sectors.
Sectors are sequantial but pages, which store sectors, are not sequantial.
So we extracted segments from bio and check which page includes the segment.

The request is a set of bios.
When Kernel starts IO processing, it creates a request which includes one bio and inserts the request into the request-queue.
Soon or later, driver will extract a request from the request-queue and process it.
While the request is in the request-queue, IO scheduler does something for the request.

As I told you, there is only one bio in the request at the beginning.
What IO scheduler does is adding more bios into the request.
For example, let's assume there is a write-request including 8 sectors (sector numbers are 0 ~ 7) in the request-queue.
And user application continues to write data at sector 8 ~ 15.
Kernel also creates a request for sector 8 ~ 15 and adds it into the request-queue.
Now IO scheduler does its job.
It merges two requests into one for sector 0 ~ 15.

The job of IO scheduler is watching the request-queue and merging sequential IO request.
It can increase disk throughtput because head of HDD disk moves less.

IO scheduler has severy scheduler policy such as noop, cfq, deadline and etc.
And it has more features to increase block IO performance.
Please refer other documents for the detail.

In this chapter, we will modify mybrd driver source to handle requests.

source: https://github.com/gurugio/mybrd/blob/ch04-request-mode/mybrd.c

## create the request-queue for request-mode

### create a new queue

In previous chapters, we created mybrd_device object in mybrd_alloc() and generated a request-queue with blk_alloc_queue_node().
The driver can get bio from kernel via the request-queue.

So the first thing we need to change is the request-queue.
Please see mybrd_alloc() function in the new source file, there is new variable, queue_mode, that show what is the base unit of IO processing.
If IO processing is based on the bio, queue_mode is set as MYBRD_Q_BIO, so called as bio-mode.
If you want to change IO processing to be based on the request, you need to set the queue_mode value as MYBRD_Q_RQ, so called as request-mode.

The queue_mode value is defined as MYBRD_Q_RQ, so the request-queue is created by blk_init_queue_node() in mybrd_alloc() of the source file.
There is still code to create the bio-based request-queue with blk_alloc_queue_node().
I'll explain the difference of blk_alloc_queue_node() and blk_init_queue_node() later.

blk_init_queue_node() registers a function of driver, mybrd_request_fn() as the request handler.
And driver should provide spin-lock for synchronization of the request-queue.
Finally blk_queue_prep_rq() is called to register blk_queue_prep_rq().
I'll explain blk_queue_prep_rq() later.

For now, please notice that we need to do three things:
1. create a request-queue with blk_init_queue_node()
2. provide a spin-lock
3. call blk_queue_prep_rq()

### blk_init_queue_node()

이제부터 커널 코드를 조금씩 보겠습니다. 커널 소스를 받고 태그를 생성해놓으셨을거라 생각됩니다. 그럼 blk_init_queue_node()함수를 찾아보시면 blk_alloc_queue_node()와 blk_init_allocated_queue()로 이뤄어져있다는걸 확인할 수 있습니다.

blk_alloc_queue_node()는 이미 우리가 사용했던 함수입니다. 즉 아주 단순하게 bio처리만 할 수 있는 큐를 만드는 함수입니다. 이렇게 만들어진 큐에 blk_init_allocated_queue()를 이용해서 뭔가 추가적인 설정을 하게되고 그래서 결국 request를 처리하는 큐가 만들어지는 것입니다.

blk_init_allocated_queue() 함수를 보면 눈에 띄는 것이 큐의 request_fn 필드에 우리가 전달한 함수 포인터를 설정하는 것입니다. 나중에 커널에서 큐의 request_fn 포인터를 읽어서 드라이버의 함수를 호출할 것입니다. 그리고 우리가 전달한 spin-lock도 큐에 저장됩니다.

그리고 또 익숙한 함수가 있습니다. blk_queue_make_request()함수가 있고, 함수 인자에 blk_queue_bio라는 함수가 전달되고 있습니다. 분명 드라이버가 제공하는 함수가 아닙니다. 커널에 포함된 함수입니다. 나중에 디스크가 커널에 등록되고 큐가 동작을 시작하면 커널이 blk_queue_bio() 함수를 통해 큐에서 bio를 꺼내온다는걸 알 수 있습니다. 그리고 마지막으로 elevator_init()라는 함수를 호출합니다. 이 elevator라는게 뭘까요. 

여기서 커널 분석할때 한가지 팁을 말씀드리겠습니다. 커널 개발자들은 매우 뛰어난 사람들입니다. 전세계에서 얼마나 많은 리눅스 서버들, 임베디드 리눅스 장비들이 짧게는 며칠씩 길게는 몇년씩 돌아가고 있을지 상상해보세요. 그런 안정성을 가진 거대한 소프트웨어를 만들려면 저같은 사람은 비교도 안되게 뛰어난 사람들이 디자인하고 핵심 코드를 만들었을 것입니다. 그런 사람들이 코드에 주석다는걸 좋아할까요? 매우 싫어합니다. 왜냐면 코드만 만들기도 바쁜데 주석을 쓰는 시간도 아깝다는 사람도 있습니다. 그리고 더 큰 이유는 코드만 봐도 이해가 되는데 왜 주석을 달아야되는지 필요성을 모르겠다는 것입니다. 만약 코드가 봐도 뭐하는지 모르겠다면 그건 잘못된 코드라는 것이지요. 얼마나 코드를 잘짜고 잘 읽으면 주석의 필요성을 모르겠다는건지 정말 대단합니다. 어쨌든 커널 코드는 주석을 잘 안답니다. 그런데 보세요. elevator_init()함수에는 주석이 있습니다. /* init elevator */라는 주석을 보는 순간 저는 사실 황당함을 느꼈습니다. 그냥 함수 이름을 그대로 써논거같은데 왜 주석을 달았을까요. 저는 개인적으로 그만큼 elevator라는게 중요하기 때문이라고 생각합니다. 커널 코드는 최대한 주석을 줄이려고하기때문에 주석이 있다는건 그만큼 중요하거나 그 뒷배경이 복잡한 코드라는 것입니다. 따라서 이 elevator라는 것도 블럭 레이어에 있어서 중요한 것일겁니다.

말을 길게 썼는데 사실 elevator는 IO scheduler를 말하는 것입니다. 이전에 우리가 bio 단위로 IO를 처리할 때는 elevator가 없는 큐를 썼었습니다. 그런데 지금 커널이 elevator가 있는 큐를 만들어주고 있습니다. 그래서 결론은 지금 생성되는 큐는 IO scheduler를 가지고 있는 큐라는 것입니다. 그래서 bio단위로 동작하는 큐보다 throughput이 좋을거라고 이전에 설명드렸습니다.

elevator라는 이름이 좀 이상할수도 있습니다. 왜 스케줄러를 elevator라고 부르는건지 이유가 있습니다. 그 이유를 찾는건 조금만 검색해도 알 수 있으므로 여러분께 맡기겠습니다.

커널은 많은 사람들이 의논해서 만들기때문에 이름을 지을때도 논쟁이 될때가 많습니다. IO scheduler같이 커널의 전체적인 성능에 중요한 일을 하는 코드에 갠히 elevator라는 이름을 붙이지는 않을 것입니다. 그 외에 다른 코드를 볼 때도 항상 왜 이런 이름을 붙였을까 고민해보는 것도 좋은 경험이 될것입니다.

###blk_queue_bio()

blk_init_queue_node()와 blk_alloc_queue_node()의 차이가 뭘까요. 바로 blk_queue_bio()를 호출하는 것입니다. 드라이버에서 큐를 만들때 blk_alloc_queue_node()로 생성하면 bio처리를 할 때 드라이버가 만든 함수가 호출됩니다. 하지만 blk_init_queue_node()로 큐를 만들면 커널이 제공하는 blk_queue_bio()함수가 bio 처리를 합니다.

내친김에 blk_queue_bio() 함수도 열어볼까요. 조금 복잡하고 저도 다 아는게 아니므로 개론적인 것들만 설명하겠습니다. 우리가 이미 아는 함수들이 사용되고 있습니다. 뭔가 에러가 발생했으면 bio_endio()를 호출하고 끝냅니다. 중간을 보면 elv_merge()라는 함수가 처음으로 request 객체를 사용하고 있습니다. 그리고 바로 밑에 bio_attempt_back_merge()나 bio_attempt_front_merge() 함수를 호출합니다. 감이 오실겁니다. 큐에있는 bio들을 조사해서 현재 전달된 bio와 합치는 것입니다. 

그리고 get_request()함수로 새로운 request를 생성합니다. 하나의 큐가 가질 수 있는 request는 한계가 있습니다. 그 한계를 큐의 depth라고 부릅니다. 만약 큐에 request가 너무 많다면 이 get_request()함수는 프로세스를 잠들게합니다.

그 다음 plug라는게 사용되는데 이건 글이 너무 길어지므로 설명하지 않겠습니다. __blk_run_queue()함수가 최종적으로 큐에 등록된 request_fn 함수를 호출하는 함수입니다.

아주 간략하게만 설명했습니다만 이전에 짧게만 설명했던 큐에서 bio들이 합쳐지는 과정이 어떻게 구현되고 언제 어떻게 드라이버가 등록한 request 처리 함수가 호출되는지 약간은 감이 오셨을거라 생각됩니다. 이제 Understanding the Linux kernel 등 본격적인 커널 책을 보시면 좀더 잘 이해가 되실 겁니다.


## request handling

Let's see the code for the request handling.

### mybrd_request_fn()

우리는 mybrd_alloc()에서 큐를 만들면서 mybrd_request_fn() 함수를 등록했습니다. 이제 IO가 발생하면 커널이 request-queue에 request를 저장하고 적당한 시점에 mybrd_request_fn()를 호출해줄 것입니다. mybrd_request_fn() 함수 인자를 보면 request-queue가 전달됩니다. 큐 전체가 전달된 것입니다. 그러므로 mybrd_request_fn()이 호출되기전에 IO scheduler가 큐에 대한 처리를 끝냈다는걸 알 수 있습니다. 이제 드라이버는 큐에서 request를 하나씩 꺼내서 처리하면 됩니다.

We registered mybrd_request_fn() as bio handler when we created the request-queue in mybrd_alloc().
If IO is generated, kernel stores a request in the request-queue

####blk_fetch_request()

request-queue에서 request를 뽑아오는 함수입니다. 참고로 이전에 blk_queue_bio() 함수에서 __blk_run_queue()를 호출하기전에 큐의 spin-lock을 잠그는 코드가 있었습니다. 따라서 blk_fetch_request()를 호출할 때는 큐가 잠겨있는 상태입니다. 그러므로 blk_fetch_request()를 호출하기전에 락을 잠글 필요가 없습니다. 대신 호출 후에 락을 풀어줍니다. 드라이버가 큐를 처리하는 중에 사용자 어플에서 다시 IO를 발생시키고, 커널이 큐에 접근해야할 필요가 생길 수 있습니다. 만약 락을 안풀어주면 커널은 큐에 접근할 수 없게되고 어플은 IO를 발생시키지도 못하겠지요.

참고로 blk_fetch_request() 함수의 코드를 보면 blk_peek_request()를 호출하게되고 여기에서 큐에 저장된 prep_rq_fn 함수 포인터를 읽고 호출합니다. prep_rq_fn 함수는 BLKPREP_OK나 BLKPREP_DEFER, BLKPERP_KILL 중 하나를 반환하게 돼있습니다.

이렇게 prep_rq_fn 함수가 무슨 값을 반환해야하는지 언제 호출되는지 등은 사실 커널 코드를 보지않으면 알수가 없습니다. 따라서 다른 커널 드라이버들이 blk_fetch_request()를 어떻게 호출하는지 등을 잘 보고 따라서 만들 필요가 있습니다. 물론 lkml등의 메일링 리스트를 계속 주시하면서 새롭게 생기는 코드들을 알아두면 prep_rq_fn이 생겼을때부터 적용할 수 있겠지요.

어쨌든 mybrd_alloc()에서 blk_queue_prep_rq()함수에 mybrd_prep_rq_fn()을 전달했습니다. 그래서 prep_rq_fn이 호출될때 mybrd_prep_rq_fn()이 호출될거라는걸 알 수 있습니다.

그리고 주의할 것은 blk_fetch_request()를 다시 호출하기전에 락을 잡는걸 잊지 말아야합니다. blk_fetch_request()함수의 주석을 보면 락이 잠긴 상태에서 호출되어야한다고 써있습니다. 커널의 주석은 한줄한줄 주의깊에 읽고 잘 따라야합니다.

####blk_end_request_all()

request 처리가 끝났음을 커널에 알리는 함수입니다. request 객체를 커널이 생성했으니 커널에게 해지를 맡기는게 당연하겠지요. 인자로 request의 포인터와 에러 값을 받습니다. 에러는 대표적으로 -EIO등을 쓸 수 있겠고, 에러가 없다면 0을 전달하면 됩니다.

###_mybrd_request_fn()

request 한개를 받아서 처리하는 함수입니다. blk_fetch_request()에서 반환된 request는 mybrd_prep_rq_fn()에서 한번 처리된 request입니다. mybrd_prep_rq_fn()은 request의 special 필드에 mybrd 객체의 주소를 저장합니다. 그러니 _mybrd_request_fn()이 호출된 시점에서 request의 special 필드에 mybrd 객체가 저장되어있어야 하겠지요. 사실 이건 그냥 prep_rq_fn이라는게 있다는 것을 보여주기위한 코드입니다. 드라이버나 장치의 특성에 따라서 필요한 처리를 하면 되고, 필요없으면 생략해도 됩니다.

blk_rq_pos()는 request에서 처리해야할 첫번째 섹터 번호를 알아내는 함수입니다. blk_rq_pos()가 정의된 include/linux/blkdev.h 파일을 열어보면 blk_rq_pos()외에도 request의 정보를 얻어내는 함수들이 많이 있습니다. request 구조체를 직접 접근하지말고 이런 helper 함수를 사용하는게 중요합니다. 왜냐면 request의 구조체 자체는 언제든지 바뀔 수 있기 때문입니다. request 구조체가 바뀌어도 helper함수의 결과값은 항상 동일하므로 request의 변화에 투명한 코드를 만들 수 있습니다.

mybrd에서는 rq_for_each_segment()를 써서 세그먼트를 하나씩 열어서 처리합니다. 그런데 request는 연속된 섹터를 포함하고 있습니다. 따라서 rq_for_each_segment()를 쓰는 것보다 더 좋은 방법이 있을 것입니다. 하지만 mybrd에는 이미 섹터 단위로 페이지를 관리하도록 구현된 radix-tree가 있으므로 이걸 그대로 이용하기 위해 rq_for_each_segment()를 이용합니다. 세그먼트를 얻어온 다음에 radix-tree에 데이터를 넣고 빼는 코드는 bio-mode 코드와 동일합니다.


## request handling in SOFTIRQ

소스: https://github.com/gurugio/mybrd/blob/ch04-request-mode-softirq/mybrd.c

irqmode라는걸 추가할 수 있는데 한번 만들어보겠습니다. 이전에는 커널이 호출한 즉시 request들을 처리했습니다. 예를들어서 진짜 하드디스크의 드라이버라면 디스크에서 이전 데이터의 처리가 끝났다는 인터럽트가 발생할때마다 새로운 request를 처리할 것입니다. 그런데 이렇게 인터럽트가 발생했을 때마다 request를 처리하는게 좋지만은 않습니다. request 처리를 인터럽트 핸들러에서 구현하려면 프로세스가 잠들어서도 안되고, 그러다보니 메모리 할당도 까다로워지는 등 여러가지 불편한점들이 많습니다.

따라서 이런 request 처리를 잠시 미뤘다가 하는 방법도 있습니다. 커널이 제공하는 softirq라는 지연 처리 메커니즘이 있고 이걸 활용해서 request 처리를 하는 방법입니다.

softirq 자체에 대해서는 다음 문서를 참고하시기 바랍니다.

https://lwn.net/Articles/520076/

우리는 실제 코드를 보면서 이해해보겠습니다.

###blk_complete_request()

blk_cpu_done이라는 per-cpu 리스트가 있습니다. 드라이버에서 blk_complete_request()를 호출하면 해당 request를 blk_cpu_done 리스트에 추가해놓고 request_fn 함수를 종료합니다. 

blk_cpu_done이라는게 뭔지 blk_cpu_done으로 코드를 검색해보면 blk_softirq_init() 함수에 리스트를 초기화하는게 보입니다. blk_softirq_init()함수는 blk_cpu_done 리스트를 만들고, BLOCK_SOFTIRQ를 등록합니다.

그러면 나중에 softirq가 실행될 때 BLOCK_SOFTIRQ로 등록된 blk_done_softirq() 함수가 호출되고, blk_cpu_done 리스트에서 request를 꺼내서 큐에 있는 softirq_done_fn 함수를 호출합니다. 결국 이런 과정을 거쳐서 mybrd_softirq_done_fn() 함수가 호출되는 것입니다.

###mybrd_softirq_done_fn()

어짜피 하나의 request를 처리해야하는건 mybrd_request_fn과 동일합니다. mybrd_request_fn()에서 했듯이 _mybrd_request_fn()을 호출해서 IO를 처리합니다.

차이가 있다면 request의 queuelist를 초기화해서 큐로부터 request를 분리하는 것 뿐입니다. request 처리가 완료되었을 때 blk_end_request_all()을 호출하는 것도 mybrd_request_fn()과 동일합니다.
