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

There are other values BLK_MQ_RQ_QUEUE_BUSY and ERROR.
What happens if driver returns BLK_MQ_RQ_QUEUE_BUSY?
Yes, as the comment shows, corresponding request will be delayed and send to driver later.

The work-queue is a kind of queue for the work.
The work is a job that you want to do later.
So block layer would create a work-queue for works that have delayed IO handling.

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

## implementation of work-queue

We checked how block layer uses the work-queue, let's check the implementation.

### struct work_struct

We have seen some code for adding run_work field of struct blk_mq_hw_ctx into the work-queue kblockd_workqueue.
The run_work field is an object of struct delayed_work.
struct delayed_work consists of struct work_struct and timer.

We look into struct work_struct and `__INIT_WORK` macro to initialize struct work_struct.

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

They are simple.
struct work_struct consists of a list-node entry and function pointer func.
And `__init_WORK` just adds the list-node into the list.

### queue_work

queue_work() adds new work into the work-queue with following parameters:
* wq: pointer to work-queue
* work: pointer to work

queue_work() is a wrapper of `__queue_work()`.

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

Let's check some code of `__queue_work()`.

```
static void __queue_work(int cpu, struct workqueue_struct *wq,
			 struct work_struct *work)
... skip...
retry:
	if (req_cpu == WORK_CPU_UNBOUND)
		cpu = raw_smp_processor_id();
```
It gets the CPU number of current thread.

```
	struct pool_workqueue *pwq;
...
	/* pwq which will be used unless @work is executing elsewhere */
	if (!(wq->flags & WQ_UNBOUND))
		pwq = per_cpu_ptr(wq->cpu_pwqs, cpu);
	else
		pwq = unbound_pwq_by_node(wq, cpu_to_node(cpu));
```

What this function does is just adding a list-node into a list.
But the implementation is not simple.
That is because it uses struct pool_workqueue.

queue_work() didn't pass WQ_UNBOUND flag, so it gets an object of struct pool_workqueue from per-cpu variable wr->cpu_pwqs.
Let's check what struct pool_workqueue is later.
But please rembember that work-queue has per-cpu lists and work is added into a list of the same CPU.
And later the work will be executed on the same CPU.

Just go ahead now.

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

Finally work is added to a list of pwq->pool->worklist.
And pwq->nr_active has the number of work in the list.

### struct workqueue_struct

Now let's check the definition of workqueue_struct.
We should look at cpu_pwqs field carefully.

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

cpu_pwqs is per-cpu variable of struct pool_workqueue as following.
And struct woker_pool is core of struct pool_workqueue.

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

### init_workqueues

To understand work-queue, let's start with init_workqueues().
It prepares environment to use the work-queue at early booting step.

First it initializes worker_pool for each CPU.
```
	cpu_notifier(workqueue_cpu_up_callback, CPU_PRI_WORKQUEUE_UP);
... skip ...
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
cpu_worker_pools is a per-cpu variable as following.
cpu_worker_pools of each CPU has two objects of struct worker_pool.
for_each_cpu_worker_pool macro generates a loop for each worker_pool of current CPU.

```
	NR_STD_WORKER_POOLS	= 2,		/* # standard pools per cpu */
...skip...
/* the per-cpu worker pools */
static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS],
				     cpu_worker_pools);
                     
#define for_each_cpu_worker_pool(pool, cpu)				\
	for ((pool) = &per_cpu(cpu_worker_pools, cpu)[0];		\
	     (pool) < &per_cpu(cpu_worker_pools, cpu)[NR_STD_WORKER_POOLS]; \
	     (pool)++)
```

So init_workqueues() initializes worker_pool for the first CPU.
Before for_each_cpu_worker_pool() loop, there is a code calling workqueue_cpu_up_callback with cpu_notifier().
cpu_notifier() calls a function when new CPU is activated.
Each CPU will call workqueue_cpu_up_callback() when it is powered on.

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
workqueue_cpu_up_callback is the same to init_workqueues().
It initializes work_pool objects for its own CPU.

And both of workqueue_cpu_up_callback() and init_workqueues() calls create_worker().

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

Finally there is a familiar structure: struct worker.
worker is allocated by alloc_worker() that is wrapper of kzalloc_node().
And worker has a thread with name of "kworker/<cpu>:<id>" or kworker/<cpu>:<id>H" created by kthread_create_on_node().

Following is the result of my notebook.
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

Yes, my notebook has 4 CPUs.
Let's check what they are doing.

```
$ sudo cat /proc/3129/stack
[<ffffffff810960bb>] worker_thread+0xcb/0x4c0
[<ffffffff8109c3d8>] kthread+0xd8/0xf0
[<ffffffff817fa49f>] ret_from_fork+0x3f/0x70
[<ffffffffffffffff>] 0xffffffffffffffff
```

All kworker threads are sleeping in worker_thread().
Let's see what it is.

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

The pool is worker->pool that is an object of worker_pool that we saw in init_workqueues().
It extracts a work from pool->worklist and run what work should do.
queue_work() was a function to add a work into pwq->pool->worklist.
And worker_thread() is a thread to extract works from pool->worklist.

### alloc_workqueue

kblockd_workqueue is created by alloc_workqueue as following.
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
Let's check what it is.

alloc_workqueue() calls `__alloc_workqueue_key()`.
```
struct workqueue_struct *__alloc_workqueue_key(const char *fmt,
					       unsigned int flags,
					       int max_active,
					       struct lock_class_key *key,
					       const char *lock_name, ...)
{
	struct workqueue_struct *wq;
	struct pool_workqueue *pwq;
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
New object of struct workqueue_struct is created by kzalloc().
And each field of the object is initialized.
Maximum number of the work-queue is 256.

```
	if (alloc_and_link_pwqs(wq) < 0)
		goto err_free_wq;
```
alloc_and_link_pwqs() initializes cpu_pwqs field of struct workqueue_struct.
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

cpu_pwqs field is per-cpu variable for struct pool_workqueue.
If system had single core, it would be like following.
```
wq->cpu_pwqs->pool = cpu_worker_pools;
```

# summary

In short, when we add a new work into a workqueue, the work is not actually added into the workqueue.
cpu_worker_pool is a kind of pool of workers of one CPU.
And all work is added into the cpu_worker_pool of its CPU without caring what workqueue the work is belong to.

Actually old work-queue implementation is not like that.
Each work-queue has its own thread on every CPUs.
For example, if block device driver adds request-handling work into kblockd_workqueue, kblockd_workqueue has per-cpu list of work and the work is added to per-cpu list of kblockd_workqueue.
One problem is kblockd_workqueue should has threads for all CPUs.
The block layer can send work on any CPU.
So kblockd_workqueue should create many threads for each CPU, for instance [kworker/kblockd/0:0], [kworker/kblockd/1:0], [kworker/kblockd/2:0], [kworker/kblockd/3:0], if system has four CPUs.

If system has 10 work-queues and 4 CPUSs, there would be 40 threads.
If system has 20 work-queues and 64 CPUs, there would be 1280 threads.
Can you guess how much memory or resources 1280 idle threads wastes?
The old work-queue implementation is not good for modern many-core system.

So kernel developers designed the cpu_work_pool concepts and provided the common pool of works for all work-queues.
