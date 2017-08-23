# spinlock

Old spinlock implementation was based on cmpxchg assembly instruction.
If cmpxchg instruction suceeds to change a counter from 0 to 1, it acquires the lock.
If the counter is already 1, it fails to lock.
But that implementation is very poor for multicore system because one global counter is shared widely.

And there is one more critical problem of cmpxchg based spinlock.
There is no order in waiting threads.
So even-if thread A has been waiting the lock first, it could get the lock last.

Therefore kernel v3 introduced ticket-based spinlock.
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

왜 큐 개념을 적용했는지 이해가 되시리라 믿습니다. 이렇게 각 큐에 넣어주고 각 쓰레드에게 대기번호를 준다는 개념이 티켓과 같다고 해서 ticket spinlock이라고 이름이 지어졌습니다.

그리고 중요한 사실이 하나 더 있는데요. 뒤에 함수 코드를 보면 눈으로 확인할 수 있는데 바로 캐시 미스가 줄어든다는 것입니다. ticket spinlock을 사용하는 각 쓰레드는 자기 고유한 티켓 번호를 받으니까 큐의 head값을 읽기만 합니다. 락을 풀어주는 쓰레드만 head값에 새로운 값을 써줍니다. 

Now you understand where this name came from.

There is one important but invisible benefit of this algorithm.
This can minimize cache bouncing

### cmpxchg

참고자료
* http://x86.renejeschke.de/html/file_module_x86_id_41.html

참고로 비트값 기반 스핀락에서 사용하는 cmpxchg 명령에 대해서 생각해보겠습니다.
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
참고자료에 보면 의사코드로 cmpxchg가 어떤 일을 하는지 보여줍니다. 메모리에 있는 값이 0이면 1로 바꾸려고 할때 cmpxchg(&spinlock, 0, 1)과 같이 호출합니다. 그러면 cmpxchg 명령은 메모리 값이 0인지 확인해서 0일때만 메모리에 1을 씁니다. 메모리 값이 1이면 메모리를 읽어오기만 하지요.

결국 스핀락을 기다리는 쓰레드들이 여러개의 코어에서 실행되고 있다면, 각 코어들은 메모리 읽기만을 계속 하고 있을것입니다. 매번 메모리로부터 프로세서 레지스터로 데이터가 계속 올라오는게 아니라 캐시에서 읽어올 것입니다. 그러다가 하나의 코어에서 락을 풀면서 메모리에 0을 쓰는 순간, 다른 코어들로 cache bouncing 신호 (인텔은 IPI라고 부르는)들이 전파되고, 모든 코어가 일제히 캐시 라인 크기 (인텔은 64바이트)씩 메모리를 읽게됩니다. 게중에 먼저 실행되서 메모리 버스의 락을 잡은 코어는 메모리에 1을 쓸것이고, 다른 코어들은 늦었으므로 메모리 값이 1인것만 보겠지요. 그렇게 또 다른 코어들은 계속 캐시를 읽을 것입니다.

만약 아주 짧은 critical section을 가지고있으면서 자주 실행되는 코드라면 cache bouncing이 자주 일어나고, 많은 코어들이 캐시가 아닌 메모리를 읽는 상황이 자주 발생되니 결국 시스템 전체 성능이 떨어질 것입니다.

###arch_spin_lock

이제 막 초기화된 spinlock은 (head,tail)=(0,0) 값을 가질 것입니다. 락을 잡는 쓰레드는 tail값이 자기 티켓 번호가 되므로 tail값을 읽어서 보관하고, tail값을 증가시키고 종료합니다.

락을 기다리는 쓰레드는 tail값을 읽어서 보관하고 tail 값을 증가시킵니다. 그리고 head값이 자기 티켓 번호와 같아질때까지 루프를 돕니다.
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
xadd는 atomic하게 값을 더해주고, 더하기 이전의 값을 반환해줍니다. 즉 tail값에 TICKET_LOCK_INC만큼 더해주는데 이 값은 1입니다.

지역변수 inc에는 더하기 이전의 값이 저장될 것이고, 더하기 이전에 head와 tail이 같다면 비어있는 큐에 처음 쓰레드가 들어온 것입니다. 결국 락을 기다리는 쓰레드가 없다는 것이므로 바로 락을 잡은 것입니다.

락을 잡은 쓰레드가 락을 풀어준다면 head값이 증가할 것입니다. 그래서 루프를 돌면서 head값을 읽습니다. 이때 READ_ONCE를 씁니다.
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
READ_ONCE는 __READ_ONCE_SIZE를 호출하는데 결국 포인터를 volatile로 타입을 바꿔서 값을 읽을때마다 반드시 메모리를 읽도록 해주는 것입니다. 컴파일러는 같은 값을 계속 읽는 코드를 보면 값을 레지스터에 저장하려고 할 것입니다. 메모리를 읽는 것보다는 레지스터를 읽는게 더 빠르니까요. 하지만 위와 같은 경우는 반드시 매번 메모리를 읽어야합니다. 레지스터만 읽어서는 다른 프로세서에서 값을 바꾼걸 알아차릴 수 없습니다. 그래서 volatile로 타입을 바꿔서 읽는 코드를 추가해서 컴파일러가 최적화를 못하도록 한 것입니다.

__tickets_equal은 2개의 정수값을 비교하는데 xor 연산자를 씁니다. 그 이유는 어셈블리 명령에서 cmp보다 xor이 빠르기때문으로 생각됩니다. 어쨌든 단순히보면 값을 비교하는 것뿐입니다.

```__ticket_lock_spinning```이나 __ticket_check_and_clear_slowpath 함수는 일반적으로는 사용되지않으니 무시해도 될것 같습니다.

함수 마지막 꼭 barrier를 넣어야한다는 것을 기억하면 좋을것 같습니다.

###arch_spin_unlock

개념적으로 큐에서 쓰레드를 빼는 것이니 head값을 증가시켜주면 끝입니다.
```
    	__add(&lock->tickets.head, TICKET_LOCK_INC, UNLOCK_LOCK_PREFIX);
```

###arch_spin_trylock

trylock은 계속 시도하는게 아니므로 메모리에 쓰기가 발생해도 괸찮겠지요. 그래서 cmpxchg를 써서 값을 갱신해보는걸로 락을 시험합니다.

큐와 티켓의 개념만 알면 간단한 코드입니다만 그런 배경지식이 없이 본다면 막막할 수 있는 코드라고 생각됩니다. 다행히 좋은 참고자료가 있어서 저도 쉽게 이해할 수 있었습니다.



