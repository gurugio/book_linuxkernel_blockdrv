# wait-queue: waiting for resource

There is a nr_requests field in struct request_queue that is the maximum number of requests the queue can has.
If more requests are generated than nr_requests, kernel makes the thread sleep for a while.
That is when wait-queue is used.

The wait-queue is a kind of queue that has sleeping threads waiting for resource released.
Let's take a look at some kernel code to understand wait-queue implementation.

reference
* http://www.makelinux.net/ldd3/chp-6-sect-2

## wait-queue of null_blk driver

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
Then it check if there is sleeping(waiting) threads for nullb_cmd with waitqueue_active(), and wakes up threads with wake_up().

Then the sleeping thread called io_schedule() in alloc_cmd() wakes up and calls prepare_to_wait() and `__alloc_cmd()`.
If it succeeds to allocate the nullb_cmd object, it exits do-while loop and calls finish_wait().
If not, it sleeps again.

In short, if it fails to allocate the resource
* prepare_to_wait -> io_schedule (or schedule) -> finish_wait

If it frees resource and wakes up sleeping threads
* waitqueue_active -> wake_up

## wait-queue in the block layer

Actually we already saw code using wait-queue().
generic_make_request() calls blk_queue_enter() before calling make_request_fn callback of mybrd driver.

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
In blk_queue_enter(), there is following code to make thread sleep.

```
    	ret = wait_event_interruptible(q->mq_freeze_wq,
				!atomic_read(&q->mq_freeze_depth) ||
				blk_queue_dying(q));
```

## implementation of wait-queue

Let's read some functions of wait-queue.

### `__wait_event`

Following is a definition of wait_event_interruptible macro function that calls `__wait_event()`.

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
```___wait_event``` has arguments as followings:
* wq: mq_freeze_wq field of struct request_queue object
  * wait_queue_head_t object
* condition: !atomic_read(&q->mq_freeze_depth) || blk_queue_dying(q)
  * mq_freeze_depth is the number of threads using the request-queue
  * blk_queue_dying: true if request-queue is being deleting
  * if condition is true, thread will be waken
* state: TASK_INTERRUPTIBLE
  * thread status when it sleeps
* exclusive: 0
  * if 0, all threads will be waken and compete for the resource
  * if 1, it will wake up only one thread
* ret: 0
  * return value
* cmd: schedule()
  * a function called to sleep
  * null_blk calls io_schedule

`___wait_event()` code is the same to null_blk driver.
It calls prepare_to_wait() and repeats to check if it needs to sleep.

When is the mq_freeze_wq field of the request-queue initialized?
blk_alloc_queue_node(), we used to initialize the request-queue in mybrd driver, initializes mq_free_wq field when it initializes the request-queue.

```
struct request_queue *blk_alloc_queue_node(gfp_t gfp_mask, int node_id)
{
    struct request_queue *q;
	int err;
    ......
    init_waitqueue_head(&q->mq_freeze_wq);
```

init_waitqueue_head() is a wrapper of `__init_waitqueue_head()`.

```
void __init_waitqueue_head(wait_queue_head_t *q, const char *name, struct lock_class_key *key)
{
    spin_lock_init(&q->lock);
	lockdep_set_class_and_name(&q->lock, key, name);
	INIT_LIST_HEAD(&q->task_list);
}
```

It's simple.
It initializes spinlock and a list for sleeping threads.
Let me skip lockdep_set_class_and_name() because it's not related to block device driver.

### prepare_to_wait_event

Function parameters are
* wait_queue_head_t *q: data struct to represent wait-queue
* wait_queue_t *wait: data struct to represent thread to be slept
* int state: thread status after sleep

What it does:
* wait->private = current: stores task_struct object of the current thread
* wait->func = autoremove_wake_function: set a function called when sleeping thread wakes up
* lock wait_queue_head_t object
* `__add_wait_queue(q, wait)`: same to list_add(). add task_list node of wait_queue_t object to the list of wait_queue_head_t object.
* set_current_state: change task status
* unlock wait_queue_head_t object

All it does is adding thread into wait_queue_head_t object and change thread status from TASK_RUNNING to TASK_INTERRUPTIBLE.
So scheduler() will make thread sleep later because its status is not TASK_RUNNING.

### finish_wait

Contrary to prepare_to_wait_event(), it set thread status to TASK_RUNNING and extract the thread from the list of wait_queue_head_t object.

### wake_up

wake_up() is wrapper of `__wake_up_common()`.
`__wake_up_common()` calls func callback of all wait_queue_t objects in wait_queue_head_t list.
prepare_to_wait_event() registers autoremove_wake_function() as callback of wait_queue_t object.
autoremove_wake_function() calls try_to_wake_up() function of scheduler.
Then scheduler wakes up all threads in the wait-queue and each thread checks valid resource.
Some threads will be able to occupy resource and go ahead.
But some will fail to occupy resource and sleep until other threads wakes them up.
