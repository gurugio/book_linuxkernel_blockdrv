# read-copy-update (RCU)

RCU is getting more popular in recent kernel version because it improve performance significantly if you use it correctly.
Let us check what it is and how it is implemented.

Basically RCU makes one write-thread and many read-thread run concurrently.
rwlock is similar to run many read-thread run at the same time but rwlock cannot run write-thread and read-thread at the same time.
Of course, it would be best to have something being enable to run many write-thread and many read-thread but it does not exist at the moment.
The best solution we have now is RCU to run one write-thread and many read-thread.

Unfortunately there are some constrains:
1. read-thread cannot read the latest data that the write-thread just has written.
1. write-thread waits for all read-lock to be unlocked.
1. Only one write-thread can run. You must use spinlock to synchronize data for many write-thread.

References

* http://www2.rdrop.com/users/paulmck/RCU/
* http://www.cs.nyu.edu/~lerner/spring11/proj_rcu.pdf
* http://www.slideshare.net/vh21/yet-another-introduction-of-linux-rcu


## How mybrd uses RCU

For example, let us review how mybrd uses RCU.

### mybrd_lookup_page uses rcu_read_lock

Let us start with the simplest example.
mybrd_lookup_page uses rcu_read_lock when it finds a page including the target sector.

```
static struct page *mybrd_lookup_page(struct mybrd_device *mybrd,
    			      sector_t sector)
{
	pgoff_t idx;
	struct page *p;

	rcu_read_lock(); // why rcu-read-lock?

	// 9 = SECTOR_SHIFT
	idx = sector >> (PAGE_SHIFT - 9);
	p = radix_tree_lookup(&mybrd->mybrd_pages, idx);

	rcu_read_unlock();

	pr_warn("lookup: page-%p index-%d sector-%d\n",
		p, p ? (int)p->index : -1, (int)sector);
	return p;
}
```

As you can see with the name of the function, rcu_read_lock, you can call rcu_read_lock before reading a resource.
It looks a little bit weird.
So far locking is usually for preventing other threads from accessing the resource and makeing only one thread access the resource.
How does the rcu_read_lock make many read-thread access the resource at the same time and?

Let us check the source code in kernel version 2.6.11.
It would be the simplest implementation of RCU because RCU was just merged into the kernel.

```
/**
 * rcu_read_lock - mark the beginning of an RCU read-side critical section.
 *
 * When synchronize_kernel() is invoked on one CPU while other CPUs
 * are within RCU read-side critical sections, then the
 * synchronize_kernel() is guaranteed to block until after all the other
 * CPUs exit their critical sections.  Similarly, if call_rcu() is invoked
 * on one CPU while other CPUs are within RCU read-side critical
 * sections, invocation of the corresponding RCU callback is deferred
 * until after the all the other CPUs exit their critical sections.
 *
 * Note, however, that RCU callbacks are permitted to run concurrently
 * with RCU read-side critical sections.  One way that this can happen
 * is via the following sequence of events: (1) CPU 0 enters an RCU
 * read-side critical section, (2) CPU 1 invokes call_rcu() to register
 * an RCU callback, (3) CPU 0 exits the RCU read-side critical section,
 * (4) CPU 2 enters a RCU read-side critical section, (5) the RCU
 * callback is invoked.  This is legal, because the RCU read-side critical
 * section that was running concurrently with the call_rcu() (and which
 * therefore might be referencing something that the corresponding RCU
 * callback would free up) has completed before the corresponding
 * RCU callback is invoked.
 *
 * RCU read-side critical sections may be nested.  Any deferred actions
 * will be deferred until the outermost RCU read-side critical section
 * completes.
 *
 * It is illegal to block while in an RCU read-side critical section.
 */
#define rcu_read_lock()    	preempt_disable()

/**
 * rcu_read_unlock - marks the end of an RCU read-side critical section.
 *
 * See rcu_read_lock() for more information.
 */
#define rcu_read_unlock()	preempt_enable()
```

So simple.
It only enables/disables preemption.
If one thread reads a resource, other threads on other CPUs also can read the resource.
But any threads on the same CPU cannot read the resource.

Now let us check the implementation of kernel version 4.4.x.
It becames more complicated but let us assume that we enable CONFIG_TREE_RCU and disable CONFIG_PREEMPT_RCU as arch/x86/configs/x86_64_defconfig sets.
If we remove code for debugging and disabled options, it would be like following code.

```
static inline void rcu_read_lock(void)
{
    __rcu_read_lock();
	__acquire(RCU);
	rcu_lock_acquire(&rcu_lock_map);
	RCU_LOCKDEP_WARN(!rcu_is_watching(),
			 "rcu_read_lock() used illegally while idle");
}

static inline void rcu_read_unlock(void)
{
    RCU_LOCKDEP_WARN(!rcu_is_watching(),
			 "rcu_read_unlock() used illegally while idle");
	__release(RCU);
	__rcu_read_unlock();
	rcu_lock_release(&rcu_lock_map); /* Keep acq info for rls diags. */
}

static inline void __rcu_read_lock(void)
{
    if (IS_ENABLED(CONFIG_PREEMPT_COUNT))
		preempt_disable();
}

static inline void __rcu_read_unlock(void)
{
    if (IS_ENABLED(CONFIG_PREEMPT_COUNT))
		preempt_enable();
}

# define __acquire(x) (void)0
# define __release(x) (void)0


# define rcu_lock_acquire(a)    	do { } while (0)
# define rcu_lock_release(a)		do { } while (0)
```

After removing extra code, only preempt_disable/enable remain.

### How mybrd_insert_page uses spin_lock/unlock

Let us see how mybrd_insert_page function inserts a page into the radix tree with radix_tree_insert function.
Please note which lock it uses.

```
static struct page *mybrd_insert_page(struct mybrd_device *mybrd,
    			      sector_t sector)
{
	pgoff_t idx;
	struct page *p;
	gfp_t gfp_flags;

	p = mybrd_lookup_page(mybrd, sector);
	if (p)
		return p;

	// must use _NOIO
	gfp_flags = GFP_NOIO | __GFP_ZERO;
	p = alloc_page(gfp_flags);
	if (!p)
		return NULL;

	if (radix_tree_preload(GFP_NOIO)) {
		__free_page(p);
		return NULL;
	}

	// According to radix tree API document,
	// radix_tree_lookup() requires rcu_read_lock(),
	// but user must ensure the sync of calls to radix_tree_insert().
	spin_lock(&mybrd->mybrd_lock);

	// #sector -> #page
	// one page can store 8-sectors
	idx = sector >> (PAGE_SHIFT - 9);
	p->index = idx;

	if (radix_tree_insert(&mybrd->mybrd_pages, idx, p)) {
		__free_page(p);
		p = radix_tree_lookup(&mybrd->mybrd_pages, idx);
		pr_warn("failed to insert page: duplicated=%d\n",
			(int)idx);
	} else {
		pr_warn("insert: page-%p index=%d sector-%d\n",
			p, (int)idx, (int)sector);
	}

	spin_unlock(&mybrd->mybrd_lock);

	radix_tree_preload_end();
	
	return p;
}
```

It uses spinlock to add a page into the radix tree.
You must use spinlock to write data protected by RCU but there is no lock in code reading data.
Therefore many threads can read data concurrently but only one thread can write data.

For your information, radix_tree_preload prepares some per-cpu data which will be uses when adding new node into the radix tree.
preempt_disable/enable are enought to protect per-cpu data.
That is why radix_tree_preload/end are necessary along with spinlock.


## How RCU uses for the linked list


커널에서 rcu를 가장 먼저 적용한 코드가 리스트입니다. 커널에서 많이 사용되는 리스트에 rcu가 적용된다면 커널 전체적으로 큰 성능 향상이 있겠지요. 리스트 코드 자체가 간단하니 rcu가 어떻게 사용되는지를 알아보기에도 좋은 코드이므로 한번 읽어보겠습니다.

어느순간 헷갈릴 수도 있는데 rcu는 여러개의 읽기 쓰레드만 동시에 실행될 수 있다는걸 주의하셔야합니다. list_add_rcu나 list_del_rcu등은 리스트를 보호하는 스핀락 등을 잠근 후에 호출되야합니다. 그리고 list_add_rcu나 list_del_rcu가 호출되는 순간에도 리스트를 읽는 쓰레드는 rcu_read_lock/unlock만 호출하면서 여러개의 쓰레드가 동시에 리스트에 접근하고 있다는걸 주의해야합니다.

리스트를 읽는 대표적인 방법이 list_for_each_entry 매크로를 쓰는 것입니다. 이 매크로도 rcu버전이 따로 있으므로, 리스트를 읽는 쓰레드는 list_for_each_entry_rcu를 써야합니다. 어쨌든 바로 list_for_each_entry_rcu를 사용하는 쓰레드가 여러개가 동시에 실행될 수 있다는 것입니다.

다음은 include/linux/rculist.h파일에서 주요 함수들을 복사해온 것입니다.
```
static inline void INIT_LIST_HEAD_RCU(struct list_head *list)
{
    WRITE_ONCE(list->next, list);
	WRITE_ONCE(list->prev, list);
}

#define list_next_rcu(list)    (*((struct list_head __rcu **)(&(list)->next)))

static inline void __list_add_rcu(struct list_head *new,
    	struct list_head *prev, struct list_head *next)
{
	new->next = next;
	new->prev = prev;
	rcu_assign_pointer(list_next_rcu(prev), new);
	next->prev = new;
}

static inline void __list_del(struct list_head * prev, struct list_head * next)
{
    next->prev = prev;
	WRITE_ONCE(prev->next, next);
}

#define list_entry_rcu(ptr, type, member) \
    container_of(lockless_dereference(ptr), type, member)

#define lockless_dereference(p) \
({ \
    typeof(p) _________p1 = READ_ONCE(p); \
	smp_read_barrier_depends(); /* Dependency order vs. p above. */ \
	(_________p1); \
})

#define list_for_each_entry_rcu(pos, head, member) \
    for (pos = list_entry_rcu((head)->next, typeof(*pos), member); \
		&pos->member != (head); \
		pos = list_entry_rcu(pos->member.next, typeof(*pos), member))
```
리스트를 처리하는 add,del 등의 함수들마다 rcu가 적용된 함수들이 있는데 리스트의 헤드를 초기화하는 INIT_LIST_HEAD도 rcu버전이 따로 있습니다. 사실 리스트의 헤드나 노드 하나를 동시에 접근하는 경우가 드물지만, 다른 데이터구조에 리스트가 포함된 경우가 있을 수 있으므로 필요할 때도 있을 것입니다. 보통은 INIT_LIST_HEAD로 초기화해도 됩니다.

list_next_rcu도 따로 구현은 됐지만 ```__rcu```라는 컴파일러 attribute 만 빼면 따로 구현할 필요는 없어보입니다. ```__rcu```는 sparse라는 커널 코드의 정적 분석을 위한 툴의 분석을 돕기 위해 정의된 것이므로 지금은 신경쓸 필요가 없습니다.

list_add_rcu를 보기전에 list_add와 무슨 차이가 있는지 알아보기위해 list_add부터 보겠습니다. list_add는 ```__list_add```로 구현되므로 ```__list_add```를 보겠습니다.
```
static inline void __list_add(struct list_head *new,
    		      struct list_head *prev,
			      struct list_head *next)
{
	next->prev = new;
	new->next = next;
	new->prev = prev;
	prev->next = new;
}
```
new노드를 리스트에 추가하기전에 먼저 next->prev포인터를 수정합니다. 만약 락이 없다면 리스트를 역방향으로 순회하던 쓰레드가 next->prev를 읽고 new 노드로 넘어오는데, new->prev 포인터는 아직 초기화가 안되었으므로 커널 패닉이 발생할 것입니다.

list_add_rcu는 ```__list_add_rcu```로 구현됩니다. ```__list_add_rcu```는 순서대로 보면
* 새로 추가될 new 노드 초기화
* smp_mb: new노드 초기화코드와 prev/next 노드 초기화 코드의 순서가 섞이지 않게 mb를 추가함
* WRITE_ONCE(prev->next, new)
 * 만약 어떤 쓰레드가 리스트를 정방향으로 순회하던중 prev를 읽고있었다면, 이제부터 new 노드에 접근이 가능함
 * 만약 smp_mb 코드가 없었다면 이 쓰레드가 new 노드에 접근이 되었다해도, new->next 포인터가 초기화되지 않았을 수도 있음
 * WRITE_ONCE매크로를 써서 컴파일러가 포인터를 레지스터에 저장하지못하도록해서 atomic하게 메모리에 있는 포인터 값을 수정하도록 했음
* next->prev = new
 * 만약 어떤 쓰레드가 리스트를 역방향으로 순회하던중 next를 읽고있었다면, 이제부터 new노드에 접근이 가능함
 * prev와 next 사이에는 smp_mb가 없음. 즉 prev와 next의 초기화는 순서가 바뀌어도 상관없음

smp_mb와 WRITE_ONCE를 합친게 smp_store_release입니다. 멀티코어환경에서 확실하게 메모리 값을 바꾸는 매크로입니다.

list_del_rcu는 ```__list_del```을 호출합니다. ```__list_del```은 list_del에서도 호출하는 함수입니다. 즉 list_del_rcu는 사실상 list_del과 같지만 함수의 짝을 맞추기위해 만든 것입니다. 그리고 특별히 주의해야할게 더 있습니다. 아래에 list_del_rcu의 주석을 보겠습니다.
```
 * Note that the caller is not permitted to immediately free
 * the newly deleted entry.  Instead, either synchronize_rcu()
 * or call_rcu() must be used to defer freeing until an RCU
 * grace period has elapsed.
 */
static inline void list_del_rcu(struct list_head *entry)
{
    __list_del_entry(entry);
	entry->prev = LIST_POISON2;
}
```
list_del_rcu로 하나의 노드를 리스트에서 제거한 후 곧바로 kfree등으로 노드의 메모리를 해지하면 안된다는 것입니다. unlock을 하는것과 별개로 synchronize_rcu나 call_rcu를 호출 한 후에 노드의 메모리를 해지하라는 것입니다. 

다음은 thin_dtr이라는 함수에서 list_del_rcu를 사용하는 코드입니다.
```
static void thin_dtr(struct dm_target *ti)
{
    struct thin_c *tc = ti->private;
	unsigned long flags;

	spin_lock_irqsave(&tc->pool->lock, flags);
	list_del_rcu(&tc->list);
	spin_unlock_irqrestore(&tc->pool->lock, flags);
	synchronize_rcu();

	thin_put(tc);
	wait_for_completion(&tc->can_destroy);
```
thin_put함수가 tc라는 객체의 메모리를 해지하는 함수입니다. 이와같이 락을 잡고 리스트를 수정한 후에 리스트 노드의 메모리가 해지되기 전에 synchronize_rcu를 호출해야합니다. critical section은 작을 수록 좋으므로 당연히 synchronize_rcu이전에 락을 풀어야겠지요.

리스트 노드의 포인터에서 리스트를 포함한 객체의 포인터를 얻어내는 list_entry 매크로도 rcu 버전이 있습니다. rcu버전이므로 메모리 베리어와 READ_ONCE등을 써서 포인터를 읽어야겠지요. 그런 처리를 lockless_dereference매크로에서 처리합니다.

마지막으로 list_for_each_entry매크로도 rcu버전에서는 list_entry_rcu를 사용하도록 바뀐것을 알 수 있습니다.

## synchronize_rcu (synchronize_kernel in v2.6)

list_del_rcu의 주석에서 "grace period"라는게 나옵니다. 바로 아래 그림처럼 rcu로 보호되는 객체를 해지하고싶지만, 이미 객체에 접근하고 있는 read 쓰레드들이 모두 종료될때까지 기다리는 시간을 grace period라고 부릅니다. 이미 객체가 변경된 다음에 read하는 쓰레드들은 이때 이미 바뀐 객체를 보고있을 것이므로 상관이 없습니다.

예를 들어 리스트에서 list_del_rcu를 써서 하나의 노드를 제거했을 때 rcu_read_lock을 호출하고 이미 리스트를 순회중인 쓰레드들이 있을건데 이 쓰레드들이 모두 rcu_read_unlock을 호출하면 grace period가 끝난 것입니다. list_del_rcu가 호출된 뒤에 rcu_read_lock을 호출하고 리스트에 접근한 쓰레드들은 이미 바뀐 리스트를 보고있을 것이므로 삭제된 노드가 메모리에서 해지되도 상관이 없겠지요. 하지만 list_del_rcu가 호출되기전에 리스트에 접근하던 쓰레드들은 삭제된 노드에 접근할 수 있으므로, 모든 쓰레드가 rcu_read_unlodk을 호출한 뒤에 노드의 메모리가 해지할 수 있습니다.

![](https://static.lwn.net/images/ns/kernel/rcu/GracePeriodBad.png)
grace period


참고자료
* https://lwn.net/Articles/253651/

그럼 이 grace period를 어떻게 확인할 수 있을까요? 즉 다른 코어에서 실행되는 쓰레드들이 rcu_read_unlock을 호출했는지 안했는지를 어떻게 알아낼 수 있을까요? grace period를 기다리는 함수가 바로 synchronize_rcu이므로 이 함수를 읽어보겠습니다.

먼저 v2.6.11에서 좀더 간단하게 구현된 코드를 읽어보겠습니다. 이전 버전에서는 synchronize_kernel이라는 이름으로 구현되었습니다.

```
struct rcu_synchronize {
    struct rcu_head head;
	struct completion completion;
};

/* Because of FASTCALL declaration of complete, we use this wrapper */
static void wakeme_after_rcu(struct rcu_head  *head)
{
	struct rcu_synchronize *rcu;

	rcu = container_of(head, struct rcu_synchronize, head);
	complete(&rcu->completion);
}

/**
 * synchronize_kernel - wait until a grace period has elapsed.
 *
 * Control will return to the caller some time after a full grace
 * period has elapsed, in other words after all currently executing RCU
 * read-side critical sections have completed.  RCU read-side critical
 * sections are delimited by rcu_read_lock() and rcu_read_unlock(),
 * and may be nested.
 */
void synchronize_kernel(void)
{
	struct rcu_synchronize rcu;

	init_completion(&rcu.completion);
	/* Will wake me after RCU finished */
	call_rcu(&rcu.head, wakeme_after_rcu);

	/* Wait for it */
	wait_for_completion(&rcu.completion);
}
```
complete라는 자료구조를 사용했는데 wait-queue와 같은거라고 생각하면 됩니다. call_rcu함수에서 per-cpu변수인 rcu_data객체에 wakeme_after_rcu함수를 등록해놓으면, 커널에서 grace-period가 종료되었을 때 wakeme_after_rcu함수를 호출하고, wakeme_after_rcu에서 wait_for_completion에서 잠든 쓰레드를 깨워주는 것입니다. 결론은 콜백함수를 등록하고 잠들었다가 커널이 grace-period가 끝나면 깨워주는 코드입니다.

여기있는 rcu_data라는것은 rcupdate.c파일에 정적으로 정의된 per-cpu변수입니다. 즉 각 코어에서 어떤 쓰레드가 grace-period를 대기중인지를 관리하는 일을 합니다.

grace-period는 다른 모든 코어에서 rcu_read_unlock이 호출되는 것을 기다리는 시간을 말하는데, 사실 rcu_read_unlock은 kernel preemption을 가능하게 해주는 함수이므로, 결국은 커널의 실행이 끝나고 유저 모드로 돌아가거나 다른 프로세스로 바뀌게되면 rcu_read_unlock이 실행된거라고도 볼 수 있습니다. 따라서 grace-period는 태스크릿을 기반으로 구현됩니다. 다음은 각 cpu마다 rcu와 관련된 rcu_process_callbacks함수를 태스크릿에 등록하는 rcu_online_cpu함수입니다.
```
static void __devinit rcu_online_cpu(int cpu)
{
    struct rcu_data *rdp = &per_cpu(rcu_data, cpu);
	struct rcu_data *bh_rdp = &per_cpu(rcu_bh_data, cpu);

	rcu_init_percpu_data(cpu, &rcu_ctrlblk, rdp);
	rcu_init_percpu_data(cpu, &rcu_bh_ctrlblk, bh_rdp);
	tasklet_init(&per_cpu(rcu_tasklet, cpu), rcu_process_callbacks, 0UL);
}
```
타이머 인터럽트가 실행되고, softirq와 tasklet이 실행되면 결국 rcu_process_callback함수가 실행됩니다. 그럼 rcu_process_callback을 볼까요.
```
/*
 * This does the RCU processing work from tasklet context. 
 */
static void __rcu_process_callbacks(struct rcu_ctrlblk *rcp,
    		struct rcu_state *rsp, struct rcu_data *rdp)
{
	if (rdp->curlist && !rcu_batch_before(rcp->completed, rdp->batch)) {
		*rdp->donetail = rdp->curlist;
		rdp->donetail = rdp->curtail;
		rdp->curlist = NULL;
		rdp->curtail = &rdp->curlist;
	}

	local_irq_disable();
	if (rdp->nxtlist && !rdp->curlist) {
		rdp->curlist = rdp->nxtlist;
		rdp->curtail = rdp->nxttail;
		rdp->nxtlist = NULL;
		rdp->nxttail = &rdp->nxtlist;
		local_irq_enable();

		/*
		 * start the next batch of callbacks
		 */

		/* determine batch number */
		rdp->batch = rcp->cur + 1;
		/* see the comment and corresponding wmb() in
		 * the rcu_start_batch()
		 */
		smp_rmb();

		if (!rcp->next_pending) {
			/* and start it/schedule start if it's a new batch */
			spin_lock(&rsp->lock);
			rcu_start_batch(rcp, rsp, 1);
			spin_unlock(&rsp->lock);
		}
	} else {
		local_irq_enable();
	}
	rcu_check_quiescent_state(rcp, rsp, rdp);
	if (rdp->donelist)
		rcu_do_batch(rdp);
}
```
이전에 call_rcu에서 rdp->nxtlist에 새로운 rcu_head객체를 추가했습니다. 그래서 이번에는 rdp->nxtlist에 있는 rcu_head 객체를 rdp->curlist로 옮기고, rdp->nxtlist는 NULL로 초기화합니다. 이제 이 다음부터 추가된 rcu_head객체는 현재 grace-period에 포함되지 않게됩니다.

새로 rdp->batch에 현재 grace-period의 번호를 저장하고, rcu_start_batch를 호출합니다. rcu_start_batch에서는 rsp->cpumask 값을 초기화합니다. 만약 시스템에 0,1,2,3번 총 4개의 cpu가 있으면 4개의 비트를 1로 써서 0xf값이 됩니다. 

call_rcu를 호출한 코어에서 cpumask 값을 초기화했으면, 다른 코어에서는 rcu_check_quiescent_state을 호출해서 비트를 0으로 지웁니다. 그래서 결국 모든 비트가 0이되면 해당 grace-period는 종료된 것이므로 rcu_do_batch함수에서 rdp에 있는 모든 콜백함수들을 호출하고 이때 처음에 synchronize_kernel에서 등록했던 wakeme_after_rcu가 호출됩니다.

코드가 복잡하고 커널 버전마다 계속 코드가 갱신되고 있어서, 코드에 대한 설명자료도 부족합니다. 저도 세부적인 코드를 정확하게 이해하고있는건지 잘 모르겠습니다. 대략 이렇게 동작한다는걸 알고 나중에 계속 봐야될것 같습니다.

참고자료

http://www2.rdrop.com/users/paulmck/RCU/
http://blog.naver.com/like8099/90030483093
