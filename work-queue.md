# work-queue

There is mybrd_queue_rq() function in mybrd driver.
It always returns BLK_MQ_RQ_QUEUE_OK value.
Following is definition of BLK_MQ_RQ_QUEUE_OK.

```
enum {
    BLK_MQ_RQ_QUEUE_OK	= 0,	/* queued fine */
	BLK_MQ_RQ_QUEUE_BUSY	= 1,	/* requeue IO for later */
	BLK_MQ_RQ_QUEUE_ERROR	= 2,	/* end IO with error */
```
BLK_MQ_RQ_QUEUE_OK외에도 BUSY와 ERROR 가 있네요. 드라이버가 만약 BLK_MQ_RQ_QUEUE_BUSY가 반환되었을때 블럭 레이어가 어떻게 처리할까요? BLK_MQ_RQ_QUEUE_BUSY의 주석에서 보듯이 나중에 다시 해당 request를 처리하게됩니다. 바로 이렇게 어떤 동작을 나중에 다시 처리하기 위해 만들어진게 work-queue입니다.

There are other values BLK_MQ_RQ_QUEUE_BUSY and ERROR.
What happens if driver returns BLK_MQ_RQ_QUEUE_BUSY?
Yes, as the comment shows, corresponding request will be delayed and send to driver later.
The work-queue is used to do something later.

reference
* https://www.kernel.org/doc/Documentation/workqueue.txt
* http://www.makelinux.net/ldd3/chp-7-sect-6

## blk_mq_run_hw_queue

Let's take a look at BLK_MQ_RQ_QUEUE_BUSY handling.

We already checked the callstack of mybrd_queue_rq() as following.

```
blk_sq_make_request -> blk_mq_run_hw_queue -> __blk_mq_run_hw_queue -> mybrd_queue_rq
```

We can check `__blk_mq_run_hw_queue` calls callback of q->mq_ops->queue_rq that is mybrd_queue_rq as following.

```
    while (!list_empty(&rq_list)) {
		struct blk_mq_queue_data bd;
		int ret;

		rq = list_first_entry(&rq_list, struct request, queuelist);
		list_del_init(&rq->queuelist);

		bd.rq = rq;
		bd.list = dptr;
		bd.last = list_empty(&rq_list);

		ret = q->mq_ops->queue_rq(hctx, &bd);
		switch (ret) {
		case BLK_MQ_RQ_QUEUE_OK:
			queued++;
			continue;
		case BLK_MQ_RQ_QUEUE_BUSY:
			list_add(&rq->queuelist, &rq_list);
			__blk_mq_requeue_request(rq);
			break;
		default:
			pr_err("blk-mq: bad return on queue: %d\n", ret);
		case BLK_MQ_RQ_QUEUE_ERROR:
			rq->errors = -EIO;
			blk_mq_end_request(rq, rq->errors);
			break;
		}
```

If the return valu is BLK_MQ_RQ_QUEUE_OK, it will handle other request in rq_list list.
Of if the return value is BLK_MQ_RQ_QUEUE_BUSY, it put the request back to rq_list list and exit the loop.

```
    /*
	 * Any items that need requeuing? Stuff them into hctx->dispatch,
	 * that is where we will continue on next queue run.
	 */
	if (!list_empty(&rq_list)) {
		spin_lock(&hctx->lock);
		list_splice(&rq_list, &hctx->dispatch);
		spin_unlock(&hctx->lock);
		/*
		 * the queue is expected stopped with BLK_MQ_RQ_QUEUE_BUSY, but
		 * it's possible the queue is stopped and restarted again
		 * before this. Queue restart will dispatch requests. And since
		 * requests in rq_list aren't added into hctx->dispatch yet,
		 * the requests in rq_list might get lost.
		 *
		 * blk_mq_run_hw_queue() already checks the STOPPED bit
		 **/
		blk_mq_run_hw_queue(hctx, true);
	}
```

Then if rq_list is not empty, driver is busy and is not able to handle IO at the moment.
If so, it moves requests to hctx->dispatch list and calls blk_mq_run_hw_queue with trun as the second argument.

Following is blk_mq_run_hw_queue() code.
```
void blk_mq_run_hw_queue(struct blk_mq_hw_ctx *hctx, bool async)
{
    if (unlikely(test_bit(BLK_MQ_S_STOPPED, &hctx->state) ||
	    !blk_mq_hw_queue_mapped(hctx)))
		return;

	if (!async) {
		int cpu = get_cpu();
		if (cpumask_test_cpu(cpu, hctx->cpumask)) {
			__blk_mq_run_hw_queue(hctx);
			put_cpu();
			return;
		}

		put_cpu();
	}

	kblockd_schedule_delayed_work_on(blk_mq_hctx_next_cpu(hctx),
			&hctx->run_work, 0);
}
```

If async argument is false, it calls `__blk_mq_run_hw_queue()` and sends requests to driver.
Of if async is true, it calls kblockd_schedule_delayed_work_on().

```
int kblockd_schedule_delayed_work_on(int cpu, struct delayed_work *dwork,
    			     unsigned long delay)
{
	return queue_delayed_work_on(cpu, kblockd_workqueue, dwork, delay);
}
EXPORT_SYMBOL(kblockd_schedule_delayed_work_on);
```
kblockd_schedule_delayed_work_on() passes a work-queue, kblockd_workqueue, to queue_delayed_work_on().
```
/**
 * queue_delayed_work_on - queue work on specific CPU after delay
 * @cpu: CPU number to execute work on
 * @wq: workqueue to use
 * @dwork: work to queue
 * @delay: number of jiffies to wait before queueing
 *
 * Return: %false if @work was already on a queue, %true otherwise.  If
 * @delay is zero and @dwork is idle, it will be scheduled for immediate
 * execution.
 */
bool queue_delayed_work_on(int cpu, struct workqueue_struct *wq,
    		   struct delayed_work *dwork, unsigned long delay)
{
	struct work_struct *work = &dwork->work;
	bool ret = false;
	unsigned long flags;

	/* read the comment in __queue_work() */
	local_irq_save(flags);

	if (!test_and_set_bit(WORK_STRUCT_PENDING_BIT, work_data_bits(work))) {
		__queue_delayed_work(cpu, wq, dwork, delay);
		ret = true;
	}

	local_irq_restore(flags);
	return ret;
}
EXPORT_SYMBOL(queue_delayed_work_on);
```

Following is parameters of queue_delayed_work_on()
* cpu: cpu number to run the work (
* wq: a object of struct workqueue_struct that represent the work-queue (=kblockd_workqueue)
* dwork: a object struct delayed_work that represent the work (=hctx->run_work)
* delay: time when the work should be done (=0)

queue_delayed_work_on이 호출될때 전달된 wq 객체는 kblockd_workqueue라는 전역변수입니다. 그리고 dwork 객체는 hctx->run_work입니다. 그럼 kblockd_workqueue가 뭔지, hctx->run_work가 뭔지를 알아보겠습니다.

### kblockd_workqueue와 hctx->run_work

Following is definition of kblockd_workqueue and how it is initialized.

```
/*
 * Controlling structure to kblockd
 */
static struct workqueue_struct *kblockd_workqueue;

int __init blk_dev_init(void)
{
    BUILD_BUG_ON(__REQ_NR_BITS > 8 *
			FIELD_SIZEOF(struct request, cmd_flags));

	/* used for unplugging and affects IO latency/throughput - HIGHPRI */
	kblockd_workqueue = alloc_workqueue("kblockd",
					    WQ_MEM_RECLAIM | WQ_HIGHPRI, 0);
	if (!kblockd_workqueue)
		panic("Failed to create kblockd\n");
```

blk_dev_init() creates a work-queue with name "kblockd".
Work-queue should work with its own thread.
Let's check if there is a thread with name "kblockd"

```
$ ps aux | grep kblock
root        73  0.0  0.0      0     0 ?        S<   Dez05   0:00 [kblockd]

gohkim   24240  0.0  0.0  16408  2524 pts/25   S+   16:15   0:00 grep --color=auto kblock
$ 
```
Yes, there is kernel thread with name of kblockd.
Let's check what it is doing.
```
$ sudo cat /proc/66/stack
[sudo] password for gohkim: 
[<ffffffff82aa2c89>] rescuer_thread+0x339/0x3c0
[<ffffffff82aa8d89>] kthread+0x109/0x140
[<ffffffff832dbf3c>] ret_from_fork+0x2c/0x40
[<ffffffffffffffff>] 0xffffffffffffffff
```

If you investigate alloc_workqueue() function, you woll find following code.

```
struct workqueue_struct *__alloc_workqueue_key(const char *fmt,
					       unsigned int flags,
					       int max_active,
					       struct lock_class_key *key,
					       const char *lock_name, ...)
... skip ...
		rescuer->task = kthread_create(rescuer_thread, rescuer, "%s",
					       wq->name);
```

kthread_create() function creates a thread that runs rescuer_thread().
The rescuer_thread() has infinite loop, so kblockd thread was sleeping in rescuer_thread().

We confirm that work-queue has its own thread.
And we find out there is kblockd_workqueue work-queue for block layer.
Let's investigate the work-queue later in detail.

First we need to check how block layer uses the work-queue kblockd_workqueue.
We already found out blk_mq_run_hw_queue() adds a work hctx->run_work into kblockd_workqueue.
So let's check what hctx->run_work is.

```
static int blk_mq_init_hctx(struct request_queue *q,
    	struct blk_mq_tag_set *set,
		struct blk_mq_hw_ctx *hctx, unsigned hctx_idx)
{
	int node;
	unsigned flush_start_tag = set->queue_depth;

	node = hctx->numa_node;
	if (node == NUMA_NO_NODE)
		node = hctx->numa_node = set->numa_node;

	INIT_DELAYED_WORK(&hctx->run_work, blk_mq_run_work_fn);
```

blk_mq_init_hctx() initializes hctx->run_work with INIT_DELAYED_WORK macro.
blk_mq_run_work_fn is a callback function registerred in hctx->run_work.

Let's check what blk_mq_run_work_fn() does.
```
static void blk_mq_run_work_fn(struct work_struct *work)
{
    struct blk_mq_hw_ctx *hctx;

	hctx = container_of(work, struct blk_mq_hw_ctx, run_work.work);

	__blk_mq_run_hw_queue(hctx);
}
```
It calls `__blk_mq_run_hw_queue()` again.

```
static void __blk_mq_run_hw_queue(struct blk_mq_hw_ctx *hctx)
{
    struct request_queue *q = hctx->queue;
	struct request *rq;
	LIST_HEAD(rq_list);
	LIST_HEAD(driver_list);
	struct list_head *dptr;
	int queued;

	WARN_ON(!cpumask_test_cpu(raw_smp_processor_id(), hctx->cpumask));

	if (unlikely(test_bit(BLK_MQ_S_STOPPED, &hctx->state)))
		return;

	hctx->run++;

	/*
	 * Touch any software queue that has pending entries.
	 */
	flush_busy_ctxs(hctx, &rq_list);

	/*
	 * If we have previous entries on our dispatch list, grab them
	 * and stuff them at the front for more fair dispatch.
	 */
	if (!list_empty_careful(&hctx->dispatch)) {
		spin_lock(&hctx->lock);
		if (!list_empty(&hctx->dispatch))
			list_splice_init(&hctx->dispatch, &rq_list);
		spin_unlock(&hctx->lock);
	}
```

Delayed requests were added into hctx->dispatch list.
Now hctx->displatch is not empty.
So requests in hctx->dispatch are moved to rq_list and sent to driver.

Therefore what kblockd_workqueue() does it just calling `__blk_mq_run_hw_queue` after some delay.

## work-queue의 내부 구현
이제 대강 work-queue의 사용법을 알았으니, 내부 구현을 한번 보겠습니다.

###struct work_struct
이전에 struct blk_mq_hw_ctx 구조체에서 run_work 필드가 work-queue에 추가되는 코드를 봤습니다. run_work필드는 struct delayed_work 구조체이고, struct delayed_work는 struct work_struct구조체에 타이머를 더해서 만들어진 것입니다.

가장 핵심이 되는 struct work_struct 구조체와 struct work_struct 타입의 객체를 초기화하는 ```__INIT_WORK```매크로를 보겠습니다.

```
struct work_struct {
	atomic_long_t data;
	struct list_head entry;
	work_func_t func;
};

#define __INIT_WORK(_work, _func, _onstack)				\
	do {								\
		__init_work((_work), _onstack);				\
		(_work)->data = (atomic_long_t) WORK_DATA_INIT();	\
		INIT_LIST_HEAD(&(_work)->entry);			\
		(_work)->func = (_func);				\
	} while (0)
```
아주 간단합니다. work-queue의 리스트 헤드에 추가될 리스트 노드와 실행할 함수로 이루어져있습니다.
* entry: work-queue내부의 리스트에 연결될 리스트 노드
* func: work-queue에서 해당 작업이 선택되면 func에 저장된 함수

###queue_work

work-queue에 새로운 작업을 추가하는 함수입니다. 함수인자를 보면
* wq: work-queue를 표현하는 struct workququq_struct 객체의 포인터
* work: work-queue에 추가될 작업을 표현하는 struct work_struct 객체의 포인터

결국 work-queue에 work를 추가하는 것 뿐입니다.
다음과 같이 함수 코드를 보면 최종적으로 ```__queue_work```라는 함수를 호출합니다.

```
static inline bool queue_work(struct workqueue_struct *wq,
			      struct work_struct *work)
{
	return queue_work_on(WORK_CPU_UNBOUND, wq, work);
}

/**
 * queue_work_on - queue work on specific cpu
 * @cpu: CPU number to execute work on
 * @wq: workqueue to use
 * @work: work to queue
 *
 * We queue the work to a specific CPU, the caller must ensure it
 * can't go away.
 *
 * Return: %false if @work was already on a queue, %true otherwise.
 */
bool queue_work_on(int cpu, struct workqueue_struct *wq,
		   struct work_struct *work)
{
	bool ret = false;
	unsigned long flags;

	local_irq_save(flags);

	if (!test_and_set_bit(WORK_STRUCT_PENDING_BIT, work_data_bits(work))) {
		__queue_work(cpu, wq, work);
		ret = true;
	}

	local_irq_restore(flags);
	return ret;
}
EXPORT_SYMBOL(queue_work_on);
```
여기에서 locak_irq_save를 호출한 것을 보면 작업을 관리할 때 CPU별로 따로 관리한다는걸 알 수 있습니다. 즉 0번 CPU에서 queue_work함수가 호출되었으면 이때 추가된 작업은 추후에 0번 CPU에서 실행될 가능성이 높다는 것입니다.

```__queue_work```함수인자는 다음과 같습니다.
* int cpu: work가 실행될 cpu 번호
* wq: work가 추가될 work-queue
* work: struct work_struct 객체 포인터

함수 코드를 읽어보겠습니다.
```
retry:
	if (req_cpu == WORK_CPU_UNBOUND)
		cpu = raw_smp_processor_id();
```
가장 먼저 현재 실행되고 있는 CPU가 뭔지를 알아냅니다.

```
	struct pool_workqueue *pwq;
...
	/* pwq which will be used unless @work is executing elsewhere */
	if (!(wq->flags & WQ_UNBOUND))
		pwq = per_cpu_ptr(wq->cpu_pwqs, cpu);
	else
		pwq = unbound_pwq_by_node(wq, cpu_to_node(cpu));
```

함수가 하는 일은 사실상 work-queue의 리스트 헤드에 새로운 노드를 추가하는 것이지만, 구현을 보면 간단하지 않습니다. 왜냐면 struct pool_workqueue 라는게 있기 때문입니다.

참고문서
* http://studyfoss.egloos.com/5626173

일단 queue_work에서 WQ_UNBOUND 플래그를 사용하지않았으므로, work-queue에서 cpu_pwqs라는 per-cpu변수에서 pool_workqueue 객체를 가져온다는걸 알 수 있습니다. 이게 뭔지는 나중에 확인하겠습니다. 여기에서 생각해야할 것은 work-queue에 per-cpu변수가 있고, 결국 각 CPU마다 작업이 연결될 것입니다.

```
	struct list_head *worklist;
...

	if (likely(pwq->nr_active < pwq->max_active)) {
		trace_workqueue_activate_work(work);
		pwq->nr_active++;
		worklist = &pwq->pool->worklist;
	} else {
		work_flags |= WORK_STRUCT_DELAYED;
		worklist = &pwq->delayed_works;
	}

	insert_work(pwq, work, worklist, work_flags);
```
pwq에 현재 실행중인 작업이 몇개인지 세는 pwq->nr_active 카운트를 증가시키고, pwq->pool->worklist 리스트헤드에 work를 추가합니다.


###struct workqueue_struct

workqueue_struct 구조체에서 가장 주의해서 봐야할 것은 우리가 queue_work에서 봤듯이 실제 작업이 연결될 cpu_pwqs 필드입니다.

```
/*
 * The externally visible workqueue.  It relays the issued work items to
 * the appropriate worker_pool through its pool_workqueues.
 */
struct workqueue_struct {
	struct list_head	pwqs;		/* WR: all pwqs of this wq */
	struct list_head	list;		/* PR: list of all workqueues */

	struct mutex		mutex;		/* protects this wq */
	int			work_color;	/* WQ: current work color */
	int			flush_color;	/* WQ: current flush color */
	atomic_t		nr_pwqs_to_flush; /* flush in progress */
	struct wq_flusher	*first_flusher;	/* WQ: first flusher */
	struct list_head	flusher_queue;	/* WQ: flush waiters */
	struct list_head	flusher_overflow; /* WQ: flush overflow list */

	struct list_head	maydays;	/* MD: pwqs requesting rescue */
	struct worker		*rescuer;	/* I: rescue worker */

	int			nr_drainers;	/* WQ: drain in progress */
	int			saved_max_active; /* WQ: saved pwq max_active */

	struct workqueue_attrs	*unbound_attrs;	/* PW: only for unbound wqs */
	struct pool_workqueue	*dfl_pwq;	/* PW: only for unbound wqs */

	char			name[WQ_NAME_LEN]; /* I: workqueue name */

	/*
	 * Destruction of workqueue_struct is sched-RCU protected to allow
	 * walking the workqueues list without grabbing wq_pool_mutex.
	 * This is used to dump all workqueues from sysrq.
	 */
	struct rcu_head		rcu;

	/* hot fields used during command issue, aligned to cacheline */
	unsigned int		flags ____cacheline_aligned; /* WQ: WQ_* flags */
	struct pool_workqueue __percpu *cpu_pwqs; /* I: per-cpu pwqs */
	struct pool_workqueue __rcu *numa_pwq_tbl[]; /* PWR: unbound pwqs indexed by node */
};
```
cpu_pwqs 필드는 per-cpu 변수이고 struct pool_workqueue라는 구조체의 객체입니다. 그리고 pool_workqueue 구조체는 struct worker_pool 구조체를 포함합니다.

```
struct pool_workqueue {
	struct worker_pool	*pool;		/* I: the associated pool */
	struct workqueue_struct *wq;		/* I: the owning workqueue */
	int			work_color;	/* L: current color */
	int			flush_color;	/* L: flushing color */
	int			refcnt;		/* L: reference count */
	int			nr_in_flight[WORK_NR_COLORS];
						/* L: nr of in_flight works */
	int			nr_active;	/* L: nr of active works */
	int			max_active;	/* L: max active works */
	struct list_head	delayed_works;	/* L: delayed works */
	struct list_head	pwqs_node;	/* WR: node on wq->pwqs */
	struct list_head	mayday_node;	/* MD: node on wq->maydays */

	/*
	 * Release of unbound pwq is punted to system_wq.  See put_pwq()
	 * and pwq_unbound_release_workfn() for details.  pool_workqueue
	 * itself is also sched-RCU protected so that the first pwq can be
	 * determined without grabbing wq->mutex.
	 */
	struct work_struct	unbound_release_work;
	struct rcu_head		rcu;
} __aligned(1 << WORK_STRUCT_FLAG_BITS);

struct worker_pool {
	spinlock_t		lock;		/* the pool lock */
	int			cpu;		/* I: the associated cpu */
	int			node;		/* I: the associated node ID */
	int			id;		/* I: pool ID */
	unsigned int		flags;		/* X: flags */

	struct list_head	worklist;	/* L: list of pending works */
	int			nr_workers;	/* L: total number of workers */

	/* nr_idle includes the ones off idle_list for rebinding */
	int			nr_idle;	/* L: currently idle ones */

	struct list_head	idle_list;	/* X: list of idle workers */
	struct timer_list	idle_timer;	/* L: worker idle timeout */
	struct timer_list	mayday_timer;	/* L: SOS timer for workers */

	/* a workers is either on busy_hash or idle_list, or the manager */
	DECLARE_HASHTABLE(busy_hash, BUSY_WORKER_HASH_ORDER);
						/* L: hash of busy workers */

	/* see manage_workers() for details on the two manager mutexes */
	struct mutex		manager_arb;	/* manager arbitration */
	struct worker		*manager;	/* L: purely informational */
	struct mutex		attach_mutex;	/* attach/detach exclusion */
	struct list_head	workers;	/* A: attached workers */
	struct completion	*detach_completion; /* all workers detached */

	struct ida		worker_ida;	/* worker IDs for task name */

	struct workqueue_attrs	*attrs;		/* I: worker attributes */
	struct hlist_node	hash_node;	/* PL: unbound_pool_hash node */
	int			refcnt;		/* PL: refcnt for unbound pools */

	/*
	 * The current concurrency level.  As it's likely to be accessed
	 * from other CPUs during try_to_wake_up(), put it in a separate
	 * cacheline.
	 */
	atomic_t		nr_running ____cacheline_aligned_in_smp;

	/*
	 * Destruction of pool is sched-RCU protected to allow dereferences
	 * from get_work_pool().
	 */
	struct rcu_head		rcu;
} ____cacheline_aligned_in_smp;
```
커널에 이렇게 주석이 많은 구조체도 많지 않을 것입니다. 그만큼 복잡하고, 성능에 큰 영향을 미치는 구조체들일 것입니다.

###init_workqueues

우선 work-queue를 사용할 수 있도록 초기화를 하는 init_workqueues 함수를 보겠습니다.

```
	cpu_notifier(workqueue_cpu_up_callback, CPU_PRI_WORKQUEUE_UP);
```

cpu_notifier는 각 CPU코어가 동작을 시작하면 호출될 함수들을 등록하는 함수입니다. 따라서 각 CPU 코어마다 workqueue_cpu_up_callback함수를 호출하도록 만들어놓은 것입니다.

그리고 각 CPU 코어를 위한 worker_pool라는걸 만듭니다.
```
	for_each_possible_cpu(cpu) {
		struct worker_pool *pool;

		i = 0;
		for_each_cpu_worker_pool(pool, cpu) {
			BUG_ON(init_worker_pool(pool));
			pool->cpu = cpu;
			cpumask_copy(pool->attrs->cpumask, cpumask_of(cpu));
			pool->attrs->nice = std_nice[i++];
			pool->node = cpu_to_node(cpu);

			/* alloc pool ID */
			mutex_lock(&wq_pool_mutex);
			BUG_ON(worker_pool_assign_id(pool));
			mutex_unlock(&wq_pool_mutex);
		}
	}
```
for_each_cpu_worker_pool 매크로로 감춰져있는데, 실제로는 cpu_worker_pools라는 per-cpu변수를 만드는 코드입니다.

workqueue.c파일의 첫부분을 보면 다음과같이 cpu_worker_pools변수가 정적으로 선언되어있습니다.
```
/* the per-cpu worker pools */
static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS],
				     cpu_worker_pools);
                     
#define for_each_cpu_worker_pool(pool, cpu)				\
	for ((pool) = &per_cpu(cpu_worker_pools, cpu)[0];		\
	     (pool) < &per_cpu(cpu_worker_pools, cpu)[NR_STD_WORKER_POOLS]; \
	     (pool)++)
```

cpu_worker_pools는 per-cpu변수인데, 각각이 struct worker_pool 타입 객체의 배열로 이루어져있습니다. 쉽게 생각하면 결국 CPU마다 여러개의 worker_pool 객체를 가지고 있게 됩니다.

그리고 바로 이렇게 초기화된 worker_pool을 처음으로 사용하는 함수가 workqueue_cpu_up_callback입니다. init_workqueues는 정적으로 선언된 cpu_worker_pool 객체들의 초기값만을 셋팅해서 준비만 하는 것이고, CPU가 실제로 활성화가 되었을 때 workqueue_cpu_up_callback함수에서 cpu_worker_pool에 연결될 첫번째 쓰레드를 하나씩 만듭니다.

```
static int workqueue_cpu_up_callback(struct notifier_block *nfb,
					       unsigned long action,
					       void *hcpu)
{
	int cpu = (unsigned long)hcpu;
	struct worker_pool *pool;
	struct workqueue_struct *wq;
	int pi;

	switch (action & ~CPU_TASKS_FROZEN) {
	case CPU_UP_PREPARE:
		for_each_cpu_worker_pool(pool, cpu) {
			if (pool->nr_workers)
				continue;
			if (!create_worker(pool))
				return NOTIFY_BAD;
		}
		break;
```
for_each_cpu_worker_pool매크로를 사용해서 각 CPU에 각 worker_pool마다 한번씩 create_worker 함수를 호출합니다.

```
static struct worker *create_worker(struct worker_pool *pool)
{
	struct worker *worker = NULL;
	int id = -1;
	char id_buf[16];

	/* ID is needed to determine kthread name */
	id = ida_simple_get(&pool->worker_ida, 0, 0, GFP_KERNEL);
	if (id < 0)
		goto fail;

	worker = alloc_worker(pool->node);
	if (!worker)
		goto fail;

	worker->pool = pool;
	worker->id = id;

	if (pool->cpu >= 0)
		snprintf(id_buf, sizeof(id_buf), "%d:%d%s", pool->cpu, id,
			 pool->attrs->nice < 0  ? "H" : "");
	else
		snprintf(id_buf, sizeof(id_buf), "u%d:%d", pool->id, id);

	worker->task = kthread_create_on_node(worker_thread, worker, pool->node,
					      "kworker/%s", id_buf);
```
드디어 struct woker라는 구조체가 나오는데, 바로 실제 실행될 쓰레드를 관리하는 구조체입니다. alloc_worker는 그냥 kzalloc_node를 호출하는 함수인데, 쓰레드가 실행될 CPU로부터 가까운 노드의 메모리를 할당하는 것입니다. 그리고 worker의 구조체를 초기화하고 kthread_create_on_node함수로 쓰레드를 하나 만듭니다. 
쓰레드의 이름을 보니 "kworker/숫자:숫자"로 나오네요. 진짜 그런 커널 쓰레드가
있는지 확인해보겠습니다. 2개의 숫자중 앞의 숫자는 pool의 번호나 pool의
CPU번호가 되고, 두번째 숫자는 ida_simple_get으로 할당받은 임의의 숫자입니다.
```
$ ps aux | grep kworker
root         4  0.0  0.0      0     0 ?        S    15:59   0:00 [kworker/0:0]
root         5  0.0  0.0      0     0 ?        S<   15:59   0:00 [kworker/0:0H]
root        17  0.0  0.0      0     0 ?        S<   15:59   0:00 [kworker/1:0H]
root        23  0.0  0.0      0     0 ?        S    15:59   0:00 [kworker/2:0]
root        24  0.0  0.0      0     0 ?        S<   15:59   0:00 [kworker/2:0H]
root        30  0.0  0.0      0     0 ?        S    15:59   0:00 [kworker/3:0]
...
```
여러개의 쓰레드가 있네요. 만약 kworker라는 이름의 쓰레드가 너무 많거나 이런
이름의 쓰레드중에 D 상태인 쓰레드가 있다면, 어디선가 잘못된 작업을 생성한 것일
수 있습니다. 아니면 작업이 종료되지못하고 계속 대기를 하고있을 수도 있습니다.
kworker쓰레드중에 하나를 골라서 무엇을 실행하고있는지 볼까요.
```
$ sudo cat /proc/3129/stack
[<ffffffff810960bb>] worker_thread+0xcb/0x4c0
[<ffffffff8109c3d8>] kthread+0xd8/0xf0
[<ffffffff817fa49f>] ret_from_fork+0x3f/0x70
[<ffffffffffffffff>] 0xffffffffffffffff
```
현재 worker_thread함수에 머물러 있습니다. 커널을 gdb로 확인해봐도 알겠지만,
worker_thread의 코드만 봐도, 현재 worker_thread함수안에서 잠든 상태인걸로
생각됩니다. 만약 잘못된 작업을 실행하고있거나 락에 걸린 작업을 실행중이라면
다른 함수를 실행하고있다고 나오겠지요. 그렇게 work-queue에 대한 디버깅을 할 수
있습니다.

이 쓰레드가 하는 일은 worker_thread 함수를 보면 알 수 있습니다.

```
static int worker_thread(void *__worker)
{
	struct worker *worker = __worker;
	struct worker_pool *pool = worker->pool;
...
	do {
		struct work_struct *work =
			list_first_entry(&pool->worklist,
					 struct work_struct, entry);

		if (likely(!(*work_data_bits(work) & WORK_STRUCT_LINKED))) {
			/* optimization path, not strictly necessary */
			process_one_work(worker, work);
			if (unlikely(!list_empty(&worker->scheduled)))
				process_scheduled_works(worker);
		} else {
			move_linked_works(work, &worker->scheduled, NULL);
			process_scheduled_works(worker);
		}
	} while (keep_working(pool));
```
최초의 쓰레드가 하는 일은 이 while 루프에 있습니다. 바로 worker_pool에 있는 worklist 리스트를 읽어서 이 리스트에 있는 각 작업들을 실행하는 것입니다. queue_work 함수가 하는 일이 바로 이 worklist 리스트에 새로운 작업을 연결하는 것이었는데, queue_work로 worklist 리스트에 추가된 작업을 실행하는 쓰레드가 만들어지는 것입니다.

###alloc_workqueue

kblockd_workqueue라는 work-queue가 생성될때 alloc_workqueue라는 함수를 사용했었습니다. alloc_workqueue함수를 알아보겠습니다.
```
int __init blk_dev_init(void)
{
	BUILD_BUG_ON(__REQ_NR_BITS > 8 *
			FIELD_SIZEOF(struct request, cmd_flags));

	/* used for unplugging and affects IO latency/throughput - HIGHPRI */
	kblockd_workqueue = alloc_workqueue("kblockd",
					    WQ_MEM_RECLAIM | WQ_HIGHPRI, 0);
	if (!kblockd_workqueue)
		panic("Failed to create kblockd\n");
```
blk_dev_init함수에서 kblockd_workqueue가 생성됩니다.

alloc_workqueue함수는 ```__alloc_workqueue_key```함수를 호출합니다.
```
struct workqueue_struct *__alloc_workqueue_key(const char *fmt,
					       unsigned int flags,
					       int max_active,
					       struct lock_class_key *key,
					       const char *lock_name, ...)
{
...
    wq = kzalloc(sizeof(*wq) + tbl_size, GFP_KERNEL);

...

	max_active = max_active ?: WQ_DFL_ACTIVE;
	max_active = wq_clamp_max_active(max_active, flags, wq->name);
    
    /* init wq */
	wq->flags = flags;
	wq->saved_max_active = max_active;
	mutex_init(&wq->mutex);
	atomic_set(&wq->nr_pwqs_to_flush, 0);
	INIT_LIST_HEAD(&wq->pwqs);
	INIT_LIST_HEAD(&wq->flusher_queue);
	INIT_LIST_HEAD(&wq->flusher_overflow);
	INIT_LIST_HEAD(&wq->maydays);
```
새로운 workqueue_struct 타입의 객체를 만들어야하므로 kzalloc으로 메모리를 할당받습니다. 그리고 각 필드를 초기화합니다. max_active가 0이면 256으로 바꿉니다. 총 256개의 작업이 연결될 수 있다는 것입니다.

```
	if (alloc_and_link_pwqs(wq) < 0)
		goto err_free_wq;
```
alloc_and_link_pwqs는 workqueue_struct에서 cpu_pwqs필드를 초기화하는 함수입니다.
```
static int alloc_and_link_pwqs(struct workqueue_struct *wq)
{
	bool highpri = wq->flags & WQ_HIGHPRI;
	int cpu, ret;

	if (!(wq->flags & WQ_UNBOUND)) {
		wq->cpu_pwqs = alloc_percpu(struct pool_workqueue);
		if (!wq->cpu_pwqs)
			return -ENOMEM;

		for_each_possible_cpu(cpu) {
			struct pool_workqueue *pwq =
				per_cpu_ptr(wq->cpu_pwqs, cpu);
			struct worker_pool *cpu_pools =
				per_cpu(cpu_worker_pools, cpu);

			init_pwq(pwq, wq, &cpu_pools[highpri]);

			mutex_lock(&wq->mutex);
			link_pwq(pwq);
			mutex_unlock(&wq->mutex);
		}
		return 0;
```
alloc_percpu는 매크로함수입니다. 인자로 전달한 구조체의 타입을 확인해서 해당 타입의 per-cpu변수를 만듭니다.

```
#define alloc_percpu(type)						\
	(typeof(type) __percpu *)__alloc_percpu(sizeof(type),		\
						__alignof__(type))
```
이렇게 cpu_pwqs객체를 만들었으면 각 CPU 코어별 cpu_worker_pools 객체와 struct pool_workqueue 객체를 연결합니다.
결국 per-cpu변수라는걸 무시하고 보면 최종적으로 이렇게 된다고 보면 됩니다.
```
wq->cpu_pwqs->pool = cpu_worker_pools;
```

간단하게 생각하면 모든 work-queue가 공유할 cpu_worker_pools를 만든 다음 cpu_worker_pools에 작업들을 연결하는 것입니다. 만약 각 work-queue가 자기만의 per-cpu 변수를 만들어서 쓰레드를 관리한다면, 프로세서가 256개가 있는 대규모 서버에서는 하나의 work-queue가 256개의 쓰레드를 생성하게 될 것이고, 시스템에 10개의 work-queue만 있어도 2560개의 쓰레드가 됩니다. 당연히 이렇게 모든 work-queue가 공유할 리스트를 만들고, 거기에 각 work-queue는 작업만 추가하게되면 불필요한 쓰레드를 만들 필요가 없겠지요.

다시한번 참고문서를 보시면서 이전 버전에서는 어떻게 구현되었었고, 어떤 문제가 생겨서 왜 이런 디자인이 되었는지를 보신다면 전체적인 설계에 대해 감이 오실것입니다.
참고: http://studyfoss.egloos.com/5626173
