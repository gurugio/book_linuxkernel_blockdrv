# page-flags

잠시 쉬어가는 시간을 가질겸해서 페이지의 플래그 셋팅에 대해서 잠깐 보겠습니다. include/linux/page-flags.h 파일에 있는 내용들입니다.

이전에 코드를 볼 때 __SetPageReferenced 등의 페이지 플래그 셋팅 함수들이 있었는데 태깅 툴로 찾아봐도 잘 안보입니다. grep으로 검색해봐도 안보입니다. 그 이유는 page-flags.h 파일에 매크로로 정의되기 때문입니다.

include/linux/page-flags.h 파일을 열어보면 주석으로 PG_reserved, PG_private 등 페이지의 플래그들의 의미에 대해서 설명이 있습니다. 실제 PG_* 플래그들은 enum으로 구현되었습니다.

예를 들어 PG_locked 플래그의 처리가 어떻게 구현되었는지를 보겠습니다. PG_locked를 아무리 검색해봐도 더 나타나는 곳이 없을 것입니다. 그대신 이런 코드가 있습니다.
```
TESTPAGEFLAG(Locked, locked)
```
그럼 TESTPAGEFLAG 매크로는 무엇일까요.
```
#define TESTPAGEFLAG(uname, lname)    				\
static inline int Page##uname(const struct page *page)			\
			{ return test_bit(PG_##lname, &page->flags); }
```
uname은 Locked이고 lname은 locked입니다. 각각 변환해서 최종 생성되는 코드가 뭔지 만들어보겠습니다.
```
static inline int PageLocked(const struct page *page)
			{ return test_bit(PG_locked, &page->flags); }
```
결국 PG_locked 비트를 검사하는 PageLocked라는 함수를 만들게됩니다.

즉, 플래그 이름에서 PG_라는 접두어는 공통이니까 생략하고 locked라는 이름만 사용한 것입니다.

lru관련 플래그는 어떤가 볼까요.

```
PAGEFLAG(LRU, lru) __CLEARPAGEFLAG(LRU, lru)
```

PG_lru 플래그 이름을 그대로 쓰는게 아니라 lru라고만 썼습니다. PAGEFLAG와 __CLEARPAGEFLAG는 뭘지 볼까요.

```
#define PAGEFLAG(uname, lname) TESTPAGEFLAG(uname, lname)    	\
	SETPAGEFLAG(uname, lname) CLEARPAGEFLAG(uname, lname)

#define TESTPAGEFLAG(uname, lname)    				\
static inline int Page##uname(const struct page *page)			\
			{ return test_bit(PG_##lname, &page->flags); }

#define SETPAGEFLAG(uname, lname)    				\
static inline void SetPage##uname(struct page *page)			\
			{ set_bit(PG_##lname, &page->flags); }

#define CLEARPAGEFLAG(uname, lname)    				\
static inline void ClearPage##uname(struct page *page)			\
			{ clear_bit(PG_##lname, &page->flags); }

static inline void __ClearPage##uname(struct page *page)    	\
			{ __clear_bit(PG_##lname, &page->flags); }
```
최종 생성되는 함수들을 보면 PageLru, SetPageLru, ClearPageLru, __ClearPageLru가 생성됩니다.

여기서 하나의 규칙이 발견됩니다. 특정 플래그를 확인하는 함수는 Page<플래그이름>이고, 플래그를 셋팅하는 함수는 SetPage<이름>, 지우는 함수는 ClearPage<이름>이 됩니다.

clear_bit을 쓸때도 있고, ```__clear_bit```을 쓸때도 있는데, 커널의 함수 이름을 정할 때 일반적으로 __<이름>처럼 쓴 경우는 락이 없는 것들입니다. __가 없으면 락을 잡아서 atomic하게 처리하는 함수라는 의미입니다. 경우에 따라서 골라쓰면 됩니다.

이제 페이지 플래그에 관한 함수들이 나오면 함수 이름 끝에 있는 플래그 이름만 보고도 뭘 하려는 것인지 알 수 있습니다.