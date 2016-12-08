# spinlock

보통 스핀락하면 정수 카운터를 두고 cmpxchg 명령으로 1을 써본다음 이전값이 0이었으면 락을 얻은 것이고, 1이었으면 다시 시도한다고 알려져있습니다. 저도 그렇게만 생각했었는데요 이번에 코드를 읽어보니 큐 개념을 적용해서 좀더 멀티코어 환경에 맞도록 개선되었습니다. 그래서 간단하게나마 소개해보려고 합니다.

0과 1로 카운터를 두고 락을 표시하면 가장 큰 문제가 먼저 기다린 쓰레드라해도 늦게 락을 얻을 수 있다는 것입니다. 경쟁이 심하지 않다면 문제가 없겠지만, 스핀락 자체가 네트워크 패킷 처리나 메모리 할당 등 경쟁이 심할 수밖에 없는데 사용되는 것이라 고려를 안할 수가 없습니다. 그래서 ticket spinlock이라는게 나왔다고 합니다.

spin_lock함수나 spinlock_t 자료구조등의 코드를 분석해보면 디버깅코드를 빼면 결국 아키텍쳐별 코드로 구현된걸 알 수 있습니다. 성능을 최대한 뽑아내야하니 불가피했을겁니다. 우리는 x86 용 코드를 읽어보겠습니다.

참고자료
* http://studyfoss.egloos.com/5144295
* http://barriosstory.blogspot.de/2008/03/cache.html
* struct arch_spinlock_t

struct spinlock_t를 보면 struct raw_spinlock으로 구현됐고 raw_spinlock은 arch_spinlock_t으로 구현된걸 알 수 있습니다.
```
typedef struct arch_spinlock {
    union {
		__ticketpair_t head_tail;
		struct __raw_tickets {
			__ticket_t head, tail;
		} tickets;
	};
} arch_spinlock_t;
```
프로세서 코어가 256개 이상이면 대기하는 쓰레드의 갯수가 많아지므로 데이터 타입의 크기가 커집니다. 일단 256개 이하라고 생각해보면 결국 다음과 같이 됩니다.
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
8비트의 head와 tail이라는 두개의 변수를 가집니다. 이름만 봐도 큐 자료구조의 개념으로 구현되었다는 것을 알 수 있습니다.

큐에 head와 tail이 있습니다. 실제 큐가 아니라 어떤 가상의 큐가 있다고 생각하고, 락을 기다리는 쓰레드를 큐게 넣는다고 가정하는 것입니다. 예를 들어 head=3, tail=3이라고 가정하면 큐에 아무런 쓰레드도 없는 상태이므로, 현재 락은 풀려있는 것입니다. 1번 쓰레드가 락을 잡았다면, 큐의 3번에 1번 쓰레드가 저장됩니다. 그리고 큐에 새로운 쓰레드를 넣었으므로 head=3, tail=4가 됩니다. 2번 쓰레드가 락을 기다린다면 큐의 4번에 쓰레드를 넣고 head=3, tail=5가 됩니다. 계속 다른 쓰레드가 락을 기다린다면 head=3으로 유지되고 tail은 계속 늘어나겠지요.

1번 쓰레드가 락을 놓는다면 큐에서 쓰레드를 빼는 것이므로 head값이 늘어납니다. tail값과는 상관없이 head=4가 됩니다. 각 쓰레드는 락을 기다릴때 자기 자신이 저장된 번호을 기억합니다. 2번 쓰레드는 head가 4이므로 자신의 차례라는 것을 알고 락을 잡습니다. 그 이후의 쓰레드들은 계속 기다립니다. 결국 먼저 대기하기 시작한 쓰레드가 먼저 락을 잡게됩니다.

왜 큐 개념을 적용했는지 이해가 되시리라 믿습니다. 이렇게 각 큐에 넣어주고 각 쓰레드에게 대기번호를 준다는 개념이 티켓과 같다고 해서 ticket spinlock이라고 이름이 지어졌습니다.

그리고 중요한 사실이 하나 더 있는데요. 뒤에 함수 코드를 보면 눈으로 확인할 수 있는데 바로 캐시 미스가 줄어든다는 것입니다. ticket spinlock을 사용하는 각 쓰레드는 자기 고유한 티켓 번호를 받으니까 큐의 head값을 읽기만 합니다. 락을 풀어주는 쓰레드만 head값에 새로운 값을 써줍니다. 

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



