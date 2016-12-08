# 드라이버의 자원을 기다리게하는 wait-queue

C로 어느정도 실무를 하다보면 블럭 장치나 네트워크 장치를 읽고 쓰는 프로세스는 언제든지 잠들 수 있다는걸 알게됩니다. 그래서 조금만 복잡한 소프트웨어가되도 쓰레드를 나눠서 동기 IO로할지, 같은 쓰레드내에서 비동기 IO로 할지 결정해야합니다. 블럭 장치의 경우는 epoll등을 이용해서 비동기 IO로 처리하는 경우가 많은것 같습니다. 네트워크 통신의 경우는 동접이 많아질 경우 비동기 IO라고해도 잠들어있는 쓰레드 갯수가 많아질 수 있으므로 자원낭비가 됩니다. 그래서 아예 연결을 끊어버리고 동기 IO로 구현하는 방법도 있습니다.

어쨌든 블럭 장치는 IO 요청이 많아지고, request-queue가 처리할 수 있는 한도 이상으로 IO 요청이 많아지면 wait-queue라는걸 이용해서 프로세스를 잠들게만듭니다. 어디에서 어떻게 프로세스가 잠들지를 판단하고 어떻게 프로세스를 잠들게하는지, 또 언제 어떻게 깨어날 수 있는지를 알아보겠습니다.

제 글들이 다 그렇듯이 그다지 깊게 들어가지않고 전체적인 디자인만 보겠습니다. 더 자세하고 정확한 내용은 다른 참고자료들을 활용하시기 바랍니다.

참고자료

http://www.makelinux.net/ldd3/chp-6-sect-2

##null_blk 드라이버에서 wait-queue 사용 예제
wait-queue 사용법은 간단한 것이니까 내부부터 설명하기보다는 예제를 보면서 생각해보는게 좋을것 같습니다.

mybrd를 만들때 소개한대로 mybrd에서 참고한 드라이버 소스가 있습니다. 램디스크 드라이버 brd와 null_blk라는 커널의 블럭 레이어 테스트 드라이버입니다. 이 두 드라이버를 합쳐서 램디스크가 멀티큐를 처리할 수 있도록 만든게 mybrd 드라이버입니다. 사실 램디스크가 멀티큐를 지원할 필요는 없지만 멀티큐의 구현을 알아보려는 생각으로 시도해본 것이지요.

그래서 null_blk.c 파일을 보면 mybrd 코드와 거의 동일합니다. 그런데 한가지만 다른게 있습니다. null_blk 드라이버는 struct nullb_cmd라는 구조체를 만들어서 bio-mode일때와 request-queue-mode일때 모두 동일하게 IO를 처리합니다.

null_queue_bio함수는 mybrd의 mybrd_make_request_fn 함수와 같은 일을 하는 함수입니다. bio-mode일때 bio를 처리하는 함수입니다. null_add_dev에서 blk_queue_make_request함수의 인자로 전달됩니다. mybrd에서 mybrd_make_request_fn 함수도 마찬가지로 blk_queue_make_request함수의 인자로 전달되서 bio를 처리할 때 호출되었습니다.

null_rq_prep_fn함수는 mybrd_request_fn에 해당하는 함수입니다. blk_queue_prep_rq 함수가 호출될때 인자로 전달되서 request를 처리할 때 호출됩니다.

null_blk 드라이버 소스에서는 null_queue_bio와 null_rq_prep_fn함수에서 alloc_cmd함수를 통해 nullb_cmd 객체를 만듭니다. 그리고 bio-mode일때는 nullb_cmd의 bio필드에 bio 포인터를 복사하고, request-mode일때는 rq필드에 request의 포인터를 복사합니다. 그래서 하위 처리 함수에서는 nullb_cmd의 객체만 전달받아서 처리하는 것이지요.

null_blk 드라이버의 구현과 별도로 제가 말씀드리고싶은 것은 바로 alloc_cmd에서 nullb_cmd 객체를 사용할때 바로 wait-queue를 사용한다는 것입니다.

가장 먼저 봐야할 코드는 nullb_queue 객체의 cmds 필드와 tag_map 필드입니다.
```
static int setup_commands(struct nullb_queue *nq)
{
    struct nullb_cmd *cmd;
	int i, tag_size;

	nq->cmds = kzalloc(nq->queue_depth * sizeof(*cmd), GFP_KERNEL);
	if (!nq->cmds)
		return -ENOMEM;

	tag_size = ALIGN(nq->queue_depth, BITS_PER_LONG) / BITS_PER_LONG;
	nq->tag_map = kzalloc(tag_size * sizeof(unsigned long), GFP_KERNEL);
	if (!nq->tag_map) {
		kfree(nq->cmds);
		return -ENOMEM;
	}

	for (i = 0; i < nq->queue_depth; i++) {
		cmd = &nq->cmds[i];
		INIT_LIST_HEAD(&cmd->list);
		cmd->ll_list.next = NULL;
		cmd->tag = -1U;
	}

	return 0;
}
```
cmds필드에는 nullb_cmd 객체를 미리 request-queue의 depth만큼 할당해놓습니다. 그리고 tag_map필드에 비트맵을 만듭니다. 이 비트맵은 각 비트가 하나의 nullb_cmd 객체의 사용중인지 가용한지 상태를 보여주는 것입니다. 비트의 값이 0이면 가용한 것이고, 1이면 이미 사용중인 것입니다.

```
static unsigned int get_tag(struct nullb_queue *nq)
{
    unsigned int tag;

	do {
		tag = find_first_zero_bit(nq->tag_map, nq->queue_depth);
		if (tag >= nq->queue_depth)
			return -1U;
	} while (test_and_set_bit_lock(tag, nq->tag_map));

	return tag;
}

static struct nullb_cmd *__alloc_cmd(struct nullb_queue *nq)
{
    struct nullb_cmd *cmd;
	unsigned int tag;

	tag = get_tag(nq);
	if (tag != -1U) {
		cmd = &nq->cmds[tag];
		cmd->tag = tag;
		cmd->nq = nq;
		if (irqmode == NULL_IRQ_TIMER) {
			hrtimer_init(&cmd->timer, CLOCK_MONOTONIC,
				     HRTIMER_MODE_REL);
			cmd->timer.function = null_cmd_timer_expired;
		}
		return cmd;
	}

	return NULL;
}
```
get_tag함수는 find_first_zero_bit함수를 이용해서 비트맵에서 0인 비트를 찾은 후, test_and_set_bit_lock함수를 이용해서 해당 비트를 1로 바꿉니다. test_and_set 계열의 함수들이 다 그렇듯이 값을 바꾸기 이전 값을 반환합니다. 만약 1을 반환하면 값을 1로 바꾸기 전에 다른 쓰레드에서 1로 바꾼 것이므로, 다시 0인 비트를 찾습니다.

__alloc_cmd에서는 만약 가용한 nullb_cmd를 찾지못할경우 NULL을 반환합니다.

```
static struct nullb_cmd *alloc_cmd(struct nullb_queue *nq, int can_wait)
{
    struct nullb_cmd *cmd;
	DEFINE_WAIT(wait);

	cmd = __alloc_cmd(nq);
	if (cmd || !can_wait)
		return cmd;

	do {
		prepare_to_wait(&nq->wait, &wait, TASK_UNINTERRUPTIBLE);
		cmd = __alloc_cmd(nq);
		if (cmd)
			break;

		io_schedule();
	} while (1);

	finish_wait(&nq->wait, &wait);
	return cmd;
}
```
alloc_cmd 함수는 __alloc_cmd를 호출해서 nullb_cmd 객체를 찾습니다. 이상없이 객체를 찾으면 당연히 IO처리를 계속하면 됩니다만 만약 가용한 nullb_cmd객체가 없다면 어떻게 해야할지를 alloc_cmd에서 결정하는 것입니다.

가용한 nullb_cmd 객체가 없다면 가장 먼저 prepare_to_wait을 호출합니다. 함수 인자나 내부 구현등은 나중에 prepare_to_wait의 코드를 볼때 생각하기로하고, 일단 지금은 어떻게 사용하는지만 생각해보겠습니다. 이름만봐도 지금은 나중에 잠들경우를 준비한다는걸 알 수 있습니다. 그리고 __alloc_cmd를 다시한번 호출해봅니다. 만약 또다시 실패한다면 이젠 정말 잠들어야합니다. 그래서 결국 io_schedule함수를 호출해서 프로세스를 강제로 잠재웁니다.

잠든 프로세스가 언제 깨어날까요. 그건 찾기를 실패한 자원이 다시 가용해질때겠지요. 그러므로 nullb_cmd 객체를 반환하는 함수를 찾아야합니다. alloc_cmd가 있으니 free_cmd가 있겠지요. 그리고 free_cmd에서는 put_tag를 호출합니다.
```
static void put_tag(struct nullb_queue *nq, unsigned int tag)
{
    clear_bit_unlock(tag, nq->tag_map);

	if (waitqueue_active(&nq->wait))
		wake_up(&nq->wait);
}

static void free_cmd(struct nullb_cmd *cmd)
{
    put_tag(cmd->nq, cmd->tag);
}
```
put_tag는 waitqueue_active를 호출해서 현재 wait-queue에서 잠든 프로세스가 있는지 확인하고 wake_up함수로 프로세스를 깨웁니다.

그러면 alloc_cmd의 io_schedule에서 잠든 프로세스는 깨어나고 다시 prepare_to_wait함수와 __alloc_cmd함수를 호출합니다. 이 루프를 nullb_cmd 객체를 찾을 때까지 반복합니다. 사용자 어플은 커널 레벨에서 순간순간 깨어나지만, 사용자 레벨로는 되돌아오지 않습니다. 그리고 nullb_cmd객체를 찾게되면 루프를 빠져나와서 finish_wait을 호출하고 종료합니다.

다시한번 정리하면 필요한 자원을 못찾았을때
* prepare_to_wait -> io_schedule (or schedule) -> finish_wait

자원을 해지하고, 자원을 기다리며 잠든 프로세스를 깨울때
* waitqueue_active -> wake_up

## 블럭레이어에서 wait-queue 사용
사실 우리는 wait-queue가 사용되는 코드를 이미 봤었습니다. generic_make_request에서 mybrd 드라이버의 make_request_fn콜백함수를 호출하기전에 blk_queue_enter함수가 있습니다.

```
blk_qc_t generic_make_request(struct bio *bio)
{
    struct bio_list bio_list_on_stack;
	blk_qc_t ret = BLK_QC_T_NONE;

......
	do {
		struct request_queue *q = bdev_get_queue(bio->bi_bdev);

		if (likely(blk_queue_enter(q, __GFP_DIRECT_RECLAIM) == 0)) {

			ret = q->make_request_fn(q, bio);

			blk_queue_exit(q);
......
```
blk_queue_enter함수를 보면 다음과 같이 프로세스를 잠재우는 코드가 있습니다.

```
    	ret = wait_event_interruptible(q->mq_freeze_wq,
				!atomic_read(&q->mq_freeze_depth) ||
				blk_queue_dying(q));
```
wait_event_interruptible는 매크로함수인데 최종적으로 ___wait_event라는 매크로함수를 호출하게됩니다.
```
#define wait_event_interruptible(wq, condition)    			\
({									\
	int __ret = 0;							\
	might_sleep();							\
	if (!(condition))						\
		__ret = __wait_event_interruptible(wq, condition);	\
	__ret;								\
})

#define __wait_event_interruptible(wq, condition)    		\
	___wait_event(wq, condition, TASK_INTERRUPTIBLE, 0, 0,		\
		      schedule())

#define ___wait_event(wq, condition, state, exclusive, ret, cmd)    \
({									\
	__label__ __out;						\
	wait_queue_t __wait;						\
	long __ret = ret;	/* explicit shadow */			\
									\
	INIT_LIST_HEAD(&__wait.task_list);				\
	if (exclusive)							\
		__wait.flags = WQ_FLAG_EXCLUSIVE;			\
	else								\
		__wait.flags = 0;					\
									\
	for (;;) {							\
		long __int = prepare_to_wait_event(&wq, &__wait, state);\
									\
		if (condition)						\
			break;						\
									\
		if (___wait_is_interruptible(state) && __int) {		\
			__ret = __int;					\
			if (exclusive) {				\
				abort_exclusive_wait(&wq, &__wait,	\
						     state, NULL);	\
				goto __out;				\
			}						\
			break;						\
		}							\
									\
		cmd;							\
	}								\
	finish_wait(&wq, &__wait);					\
__out:	__ret;								\
})
```
```___wait_event``` 매크로 함수의 인자는 다음과 같습니다.
* wq: struct request_queue 구조체의 mq_freeze_wq필드
 * wait_queue_head_t 타입의 객체
* condition: !atomic_read(&q->mq_freeze_depth) || blk_queue_dying(q)
 * mq_freeze_depth는 request-queue를 사용중인 프로세스의 갯수
 * blk_queue_dying: request-queue를 제거할때 참이됨
 * request-queue가 사용가능할 때 프로세스를 깨움
* state: TASK_INTERRUPTIBLE
 * 프로세스가 잠들때의 상태
* exclusive: 0
 * wait-queue의 기본 동작은 자원이 가용해지면 잠들어있는 모든 프로세스를 깨우는 것이지만, exclusive로 셋팅하면 하나의 프로세스만 깨움
* ret: 0
 * 프로세스가 깨워나면서 반환하는 값
* cmd: schedule()
 * 프로세스가 잠들때 호출할 함수
 * null_blk은 io_schedule함수를 호출했음

```___wait_event```코드는 null_blk드라이버에서 이미 본 코드입니다. prepare_to_wait을 호출하고 잠들어야할지 말아야할지 확인한다음 조건에 맞지 않으면 잠들기를 반복하는 것입니다.

request-queue에 mq_freeze_wq 필드는 언제 초기화된걸까요? mq_freeze_wq필드의 초기화도 이미 우리가 알아본 코드에 있습니다. blk_alloc_queue_node는 mybrd에서 request-queue를 초기화할 때 호출한 함수입니다. mq_freeze_wq필드도 당연히 request-queue가 초기화될때 같이 초기화되었을겁니다. blk_alloc_queue_node를 볼까요.

```
struct request_queue *blk_alloc_queue_node(gfp_t gfp_mask, int node_id)
{
    struct request_queue *q;
	int err;
    ......
    init_waitqueue_head(&q->mq_freeze_wq);
```
init_waitqueue_head가 호출되는걸 확인할 수 있습니다. init_wait_queue_head는 __init_waitqueue_head의 wrapper입니다.
```
void __init_waitqueue_head(wait_queue_head_t *q, const char *name, struct lock_class_key *key)
{
    spin_lock_init(&q->lock);
	lockdep_set_class_and_name(&q->lock, key, name);
	INIT_LIST_HEAD(&q->task_list);
}
```
구현은 간단합니다. 스핀락을 초기화하고 대기할 프로세스의 리스트를 초기화합니다. lockdep_set_calss_and_name은 블럭장치와 거리가 머니까 생략하겠습니다.

그럼 마지막으로 잠든 프로세스를 깨우는 코드를 찾으면 되겠네요. wake_up등의 함수에서 mq_freeze_wq를 인자로 받는 코드를 추적하면 됩니다. blk_mq_wake_waiters라는 함수와 blk_mq_unfreeze_queue라는 함수 등에서 wake_up_all을 호출합니다.

## wait-queue의 내부 구현
null_blk드라이버도 그렇고 블럭레이어에서도 마찬가지로 prepare_to_wait_event와 schedule, finish_wait함수가 프로세스를 잠들게하는 코드입니다. 그리고 wake_up함수가 프로세스를 깨웁니다. schedule은 간단하게 분석할 수 없으니 제외하고 나머지 함수들만 간단하게 읽어보겠습니다.

###prepare_to_wait_event

함수의 인자는
* wait_queue_head_t *q: wait-queue를 표현하는 구조체
* wait_queue_t *wait: 현재 잠들 프로세스를 표현하는 구조체
* int state: 잠들때 어떤 상태로 잠들 것인지

코드를 보면 다음과 같은 순서로 처리합니다.
* wait->private = current 현재 프로세스의 task_struct 객체를 저장
* wait->func = autoremove_wake_function 나중에 프로세스가 깨어날때 호출된 함수
* wait_queue_head_t 객체의 스핀락 잡기
* __add_wait_queue(q, wait): list_add함수와 동일합니다. wait_queue_t 객체의 task_list 노드를 wait_queue_head_t 객체의 리스트에 추가하는 것입니다.
* set_current_state: 프로세스의 상태 수정
* 스핀락 해지

wait_queue_head_t 객체에 프로세스를 추가하는게 코드의 전부입니다. 프로세스의 상태가 TASK_RUNNING이 아닌 TASK_INTERRUPTIBLE등의 잠든 상태로 바꿨으니 schedule함수가 호출되면 스케줄러가 알아서 프로세스를 잠재웁니다.

###finish_wait

prepare_to_wait_event의 반대로 처리하겠지요. 프로세스의 상태를 TASK_RUNNING으로 바꾸고, wait_queue_head_t 객체의 리스트에서 프로세스를 제거합니다.

###wake_up

wake_up은 ```__wake_up_common```의 wrapper입니다. __wake_up_common은 wait_queue_head_t 객체의 리스트를 돌면서 wait_queue_t객체의 func 필드에 등록한 함수를 호출합니다. prepare_to_wait_event에서는 autoremove_wake_function 함수를 등록했습니다. autoremove_wake_function은 스케줄러의 try_to_wake_up함수를 호출합니다. 이때부터는 스케줄러의 영역이므로 더는 분석하지 않겠습니다만 wait_queue_t 필드의 private필드에 프로세스의 task_struct 객체가 저장되어있으므로, 스케줄러가 프로세스를 깨울 수 있다는 것만 알면 될것같습니다.

결국 전체적인 디자인을 보면 리스트를 만들어서 잠든 프로세스의 task_struct 객체를 보관하고, 자원을 해지하는 함수가 리스트를 돌면서 리스트에 있는 task_struct 객체를 스케줄러에게 깨워달라고 의뢰하는 방식입니다.