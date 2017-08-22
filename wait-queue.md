# wait-queue: waiting for resource

There is a nr_requests field in struct request_queue that is the maximum number of requests the queue can has.
If more requests are generated than nr_requests, kernel makes the thread sleep for a while.
That is when wait-queue is used.

The wait-queue is a kind of queue that has sleeping threads waiting for resource released.
Let's take a look at some kernel code to understand wait-queue implementation.

reference
* http://www.makelinux.net/ldd3/chp-6-sect-2

## wait-queue usage of null_blk driver

Following function allocates two resources: struct nullb_cmd and bit map.
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

cmds filed is and array of struct nullb_cmd.
And tag_map field is a bit map that represent status of nullb_cmd object.
If a bit in tag_map is 1, corresponding tag_map is busy.


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

`__alloc_amd` returns NULL if there is no free nullb_cmd object.

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

alloc_cmd() calls `__alloc_cmd()` to find valid nullb_cmd object.
If there is no valid nullb_cmd object, it should wait unitl current IOs will be finishes and free nullb_cmd object.

First it calls prepare_to_wait() to initialize wait field of nullb_queue object.
Next it tries to find valid nullb_cmd object.
If it fails again, there is no choice.
It calls io_schedule() to run other threads.

If io_schedule() returns, it means another threads wakes up sleeping thread.
So it tried to find valid nullb_cmd again.
Of course, another thread can occupy nullb_cmd already.
So that do-while loop has current thread wait until it really occupies a nullb_cmd object.
Then it finalizes wait data.

put_tag() function shows how another thread can wake up sleeping threads.

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

put_tag() is called if one thread finishes IO and free a nullb_cmd object.
It clears corresponding bit to show there is free nullb_cmd boject.
Then it check if there is sleeping(waiting) threads for nullb_cmd with waitqueue_active() and wait up threads with wake_up().


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
