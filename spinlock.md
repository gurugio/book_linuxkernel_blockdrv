# spinlock

Old spinlock implementation was based on cmpxchg assembly instruction.
If cmpxchg instruction suceeds to change a counter from 0 to 1, it acquires the lock.
If the counter is already 1, it fails to lock.
But that implementation is very poor for multicore system because one global counter is shared widely.

And there is one more critical problem of cmpxchg based spinlock.
There is no order in waiting threads.
So even-if thread A has been waiting the lock first, it could get the lock last.

Therefore kernel v2.6.25 introduced ticket-based spinlock.
Let's read core code of the ticket-based spinlock.

reference
* https://lwn.net/Articles/267968/
* http://arighi.blogspot.de/2008/12/cacheline-bouncing.html

## struct arch_spinlock_t

If you check the definition of struct spinlock_t, you would know it is wrapper of arch_spinlock_t.

```
#if (CONFIG_NR_CPUS < (256 / __TICKET_LOCK_INC))
typedef u8  __ticket_t;
typedef u16 __ticketpair_t;
#else
typedef u16 __ticket_t;
typedef u32 __ticketpair_t;
#endif
... skip ...
typedef struct arch_spinlock {
    union {
		__ticketpair_t head_tail;
		struct __raw_tickets {
			__ticket_t head, tail;
		} tickets;
	};
} arch_spinlock_t;
```

If there are many COREs more than 256, struct arch_spinlock will be bigger.
For example, if there were 256 CPUs, struct arch_spinlock would be as following.

```
#define TICKET_LOCK_INC 1
#define TICKET_SHIFT 8

typedef struct arch_spinlock
{
    union
	{
		u16 head_tail;
		struct __raw_tickets
		{
			u8 head, tail;
		} tickets;
	};
} arch_spinlock_t;
```

There are head and tail.
Yes, it is based on the queue data structure.

Let's assume that there is a queue and waiting threads are added into the queue.
If head=3 and tail=3, there is no thread in the queue, then spinlock is not locked.
If thread-1 locks the spinlock, queue[3] is set to 1 and head and tail become (head=3,tail=4).
And if thread-2 waits the spinlock, queue[4] is set to 2 and (head=3,tail=5).
More threads come, tail will be increased again and again.

If thread-1 releases the spinlock, head becomes 4.
Each thread remembers its waiting number.
The thread-2 has 4 waiting number, so thread-2 can lock the spinlock.
Other threads wait again.
There is no competetion.
First come first served.

Now you understand where this name came from.

There is one important but invisible benefit of this algorithm.
This can minimize cache bouncing.
Please check following reference for cache issue
* https://lwn.net/Articles/531254/

### cmpxchg

Let's check what cmpxchg is.

cmpxchg is described as following pseudo code.
* http://x86.renejeschke.de/html/file_module_x86_id_41.html

```
/*
accumulator = AL, AX, or EAX, depending on whether
a byte, word, or doubleword comparison is being performed
*/
if(accumulator == Destination) {
    ZF = 1;
	Destination = Source;
}
else {
	ZF = 0;
	accumulator = Destination;
}
```

cmpxchg checks memory value and write 1 only-if its value is 0.
If the memory value is 1, it only read the value.

If many threads are waiting for the lock on many CPUs, each thread is running a loop to read cache.
And if one thread unlock the lock with writing 0, its CPU sends signal to other CPUs and many CPUs refresh cache.
Cache line is 128-byte on the latest CPU.
So one cach flush or bounce generates 128-byte data transfer.
If critical section is shorter or lock contention is heavier, or there are more CPUs, scalability will be worse.

### arch_spin_lock

Let's take a look at the ticket spinlock implementation.

```
static __always_inline void arch_spin_lock(arch_spinlock_t *lock)
{
    register struct __raw_tickets inc = { .tail = TICKET_LOCK_INC };

	inc = xadd(&lock->tickets, inc);
	if (likely(inc.head == inc.tail))
		goto out;

	for (;;) {
		unsigned count = SPIN_THRESHOLD;

		do {
			inc.head = READ_ONCE(lock->tickets.head);
			if (__tickets_equal(inc.head, inc.tail))
				goto clear_slowpath;
			cpu_relax();
		} while (--count);
		__ticket_lock_spinning(lock, inc.tail);
	}
clear_slowpath:
	__ticket_check_and_clear_slowpath(lock, inc.head);
out:
	barrier();	/* make sure nothing creeps before the lock is taken */
}
```

The initial value of the spinlock is (head,tail)=(0,0).
It increases tail value with xadd() and stores old tail value into inc variable.
From now, inc.tail value is ticket number of the thread.
If old tail and head are the same, the lock was acquired.
If not, thread starts loop until head value becomes ticket value (=inc.tail).

Let's look into READ_ONCE() macro that used to read memory value.

```
#define __READ_ONCE_SIZE    					\
({									\
	switch (size) {							\
	case 1: *(__u8 *)res = *(volatile __u8 *)p; break;		\
	case 2: *(__u16 *)res = *(volatile __u16 *)p; break;		\
	case 4: *(__u32 *)res = *(volatile __u32 *)p; break;		\
	case 8: *(__u64 *)res = *(volatile __u64 *)p; break;		\
	default:							\
		barrier();						\
		__builtin_memcpy((void *)res, (const void *)p, size);	\
		barrier();						\
	}								\
})
```

READ_ONCE() is wrapper of `__READ_ONCE_SIZE` that changes pointer to volatile type and read memory.
Compiler optimization usually stores value in register because it's faster.
So if there is loop to read one variable again and again, compiler could optimiza it with read variable once.
And the variable would be stored in register.

Spinlock code must read memory value to check its change.
Changing memory is up to CPU because there is cache layer between memory and CPU.
But if compiler optimizes the spinlock code to read register, it cannot detect memory change.

So READ_ONCE use volatile type pointer to prevent compiler from optimizing memory reading.
WRITE_ONCE also uses the same way.

### arch_spin_unlock

Unlocking is simple.
It just add the head value as following.
```
    	__add(&lock->tickets.head, TICKET_LOCK_INC, UNLOCK_LOCK_PREFIX);
```

### arch_spin_trylock

trylock doesn't have loop.
So it uses cmpxchg to test the lock as old style spinlock.
