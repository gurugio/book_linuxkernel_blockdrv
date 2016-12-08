# per-cpu변수와 통계 정보(v2.6.11)

중간에 제가 예전에 처음 발견하고 신기해했던 코드가 있어서 소개를 하겠습니다.

__inc_zone_page_state(page, NR_FILE_PAGES) 함수가 있었는데요 바로 페이지가 어떤 zone에 속하는지 찾고, 해당 zone에 페이지 사용량에 대한 통계를 갱신하는 함수입니다. 우리는 페이지캐시에 대한 통계만 봤었지만, 당연히 그 외에도 많은 통계정보들이 있겠지요. NR_FILE_PAGES가 정의된 include/linux/mmzone.h 파일을 보면 그 외에 다양한 통계 항목들이 있습니다.

제가 신기했던건 통계 자체가 아니라 통계 정보를 per-cpu 변수로 관리한다는 점이었습니다. 사실 통계 정보를 별 생각없이 atomic이나 spinlock으로 동기화를 하는데, CPU가 늘어날 수록 캐시 효율을 매우 떨어뜨려서 성능에 매우 안좋습니다. 그래서 어플단에서는 통계 정보를 lazy하게 갱신하는 등의 방법을 쓰는데, 커널단에서는 역시 per-cpu 변수로 각 cpu별로 저장해놨다가 사용자가 /proc/zoneinfo 파일을 출력할때만 각 cpu별 변수를 합해서 출력합니다. 그러면 평상시 페이지캐시 동작에는 오버헤드가 아주 적겠지요. 사용자가 /proc/zoneinfo를 파일을 출력해볼 일이 거의 없으니까요.

그래서 이번장에서는 per-cpu를 어떻게 구현하는지를 간단하게 알아보겠습니다. per-cpu 구현 자체를 알고나면 __inc_zone_page_state() 함수의 분석은 그리 어렵지 않겠지요.

(per-cpu의 구현은 단순히 C코드가 아니라 링커와 gcc 컴파일러의 기능을 써서 구현되므로 코드가 좀 복잡합니다. 특히 최신 커널 버전에서 per-cpu 변수의 동적 할당에 대한 코드가 많이 바껴서 v2.6와 v4.4의 코드가 완전히 다릅니다. 저도 잘 이해안되는 코드도 많고, 그 외의 코드들도 글로 설명하려니 너무 길어질것 같습니다. 그래서 간단히 v2.6 코드 기준으로 설명하는데, 실제 최신 버전에는 좀더 최적화가 되었다는걸 생각하시고 강좌를 보시기 바랍니다. 최신 버전의 구현은 다음 참고 자료를 확인하세요.)

참고
* http://jake.dothome.co.kr/per-cpu/
* http://www.makelinux.net/ldd3/chp-8-sect-5
* http://studyfoss.egloos.com/5375570
* http://studyfoss.egloos.com/5377666


##per-cpu 변수를 정적으로 만들기

###DEFINE_PER_CPU와 per-cpu 섹션

가장 기본적으로 알아야될게 per-cpu 변수를 어떻게 저장하는지겠지요. 일단 DEFINE_PER_CPU(type, name) 매크로는 type 타입으로 name라는 이름의 per-cpu변수를 정적으로 만드는 매크로입니다. 정적으로 만든다는게 중요합니다. 즉 컴파일타임에 변수가 만들어진다는 것입니다. 소스를 한번 보겠습니다.
```
#define DEFINE_PER_CPU(type, name) \
    __attribute__((__section__(".data.percpu"))) __typeof__(type) per_cpu__##name
```
C코드가 아닌 것들이 나옵니다. ```__로 시작해서 __```로 끝나는 지시어들은 대부분 gcc가 제공하는 기능들입니다. 물론 구글로 검색하면 gcc 메뉴얼에서 해당 페이지를 찾아줍니다만 간단하게 설명해보면 이렇습니다.

* ```__attribute__```: gcc에게 전달할 명령이 있다는걸 알려줍니다.
* ```__section__```: 다음에 나타날 변수를 어떤 섹션에 저장하라고 알려줍니다.
* ```__typeof__```: 다른 변수의 타입을 가져옵니다.
우선 섹션이라는게 뭘까요? 실행파일이나 object 파일이 저장될 때 ELF라는 포맷이 있습니다. 파일의 어디부터 어디까지 코드를 저장할지, 어디에 데이터를 저장할지, 어디에 파일의 헤더나 구조에 대한 정보를 저장할지 등등 미리 정해진 포맷이 있는거지요. 이때 파일이 각 구역을 섹션이라고 부릅니다.

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[], char *envp[])
{
    int dd[3];
	__typeof__(dd) k;
	printf("%d %d\n", sizeof(k), sizeof(dd));
	return 0;
}
```

이렇게 ```__typeof__```를 쓰는 예제를 한번 빌드해서 a.out을 만들어보겠습니다. 그리고 readelf -S 명령으로 어떤 섹션들이 있는지 한번 뽑아볼까요.

```
$ readelf -S a.out
There are 30 section headers, starting at offset 0x1a08:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298
       000000000000001c  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000004002b8  000002b8
       0000000000000078  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           0000000000400330  00000330
       000000000000005a  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           000000000040038a  0000038a
       000000000000000a  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          0000000000400398  00000398
       0000000000000030  0000000000000000   A       6     1     8
  [ 9] .rela.dyn         RELA             00000000004003c8  000003c8
       0000000000000018  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             00000000004003e0  000003e0
       0000000000000060  0000000000000018  AI       5    12     8
  [11] .init             PROGBITS         0000000000400440  00000440
       000000000000001a  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         0000000000400460  00000460
       0000000000000050  0000000000000010  AX       0     0     16
  [13] .text             PROGBITS         00000000004004b0  000004b0
       00000000000001c2  0000000000000000  AX       0     0     16
  [14] .fini             PROGBITS         0000000000400674  00000674
       0000000000000009  0000000000000000  AX       0     0     4
  [15] .rodata           PROGBITS         0000000000400680  00000680
       000000000000000b  0000000000000000   A       0     0     4
  [16] .eh_frame_hdr     PROGBITS         000000000040068c  0000068c
       0000000000000034  0000000000000000   A       0     0     4
  [17] .eh_frame         PROGBITS         00000000004006c0  000006c0
       00000000000000f4  0000000000000000   A       0     0     8
  [18] .init_array       INIT_ARRAY       0000000000600e10  00000e10
       0000000000000008  0000000000000000  WA       0     0     8
  [19] .fini_array       FINI_ARRAY       0000000000600e18  00000e18
       0000000000000008  0000000000000000  WA       0     0     8
  [20] .jcr              PROGBITS         0000000000600e20  00000e20
       0000000000000008  0000000000000000  WA       0     0     8
  [21] .dynamic          DYNAMIC          0000000000600e28  00000e28
       00000000000001d0  0000000000000010  WA       6     0     8
  [22] .got              PROGBITS         0000000000600ff8  00000ff8
       0000000000000008  0000000000000008  WA       0     0     8
  [23] .got.plt          PROGBITS         0000000000601000  00001000
       0000000000000038  0000000000000008  WA       0     0     8
  [24] .data             PROGBITS         0000000000601038  00001038
       0000000000000010  0000000000000000  WA       0     0     8
  [25] .bss              NOBITS           0000000000601048  00001048
       0000000000000008  0000000000000000  WA       0     0     1
  [26] .comment          PROGBITS         0000000000000000  00001048
       000000000000002d  0000000000000001  MS       0     0     1
  [27] .shstrtab         STRTAB           0000000000000000  00001075
       0000000000000108  0000000000000000           0     0     1
  [28] .symtab           SYMTAB           0000000000000000  00001180
       0000000000000630  0000000000000018          29    45     8
  [29] .strtab           STRTAB           0000000000000000  000017b0
       0000000000000251  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```
파일안에 어떤 섹션이 어느 위치에 얼마 크기로 있는지가 출력됩니다. C 프로그래밍을 좀 오래해본 분들은 .data나 .bss, .text 섹션은 들어보셨을겁니다. 각 섹션마다 역할이 있겠지만 __attribute__ ((__section__ ("섹션이름"))) 을 쓰면 내가 만든 변수를 특정한 섹션에 강제로 저장시킬 수 있습니다.

그럼 커널에는 어떤 섹션들이 있나 볼까요. 커널에서 섹션을 만드는 파일은 arch/x86_64/kernel/vmlinux.lds.S 에 있습니다. 파일을 열어보면 아래와같이 .data.percpu 섹션을 만든게 있습니다.

```
125   __per_cpu_start = .;
126   .data.percpu  : { *(.data.percpu) }
127   __per_cpu_end = .;
128   . = ALIGN(4096);
```
lds파일은 링커에게 어떤 섹션을 어떻게 만들지 등을 알려주는 파일입니다. 그리고 .data.percpu 섹션 이름 위아래에 __per_cpu_start와 __per_cpu_end가 있는데 각각 섹션의 시작 위치와 끝 위치를 저장한 변수입니다. 나중에 커널 코드에서 변수처럼 사용할 것입니다.

결국 링커를 이용해서 .data.percpu라는 섹션을 커널 이미지의 특정한 부분에 만들어놓고, 컴파일러는 DEFINE_PER_CPU 매크로로 정의한 변수들을 섹션에 저장한다는 것입니다.

그럼 ```__typeof__```가 뭔지는 알아보기위해 a.c 파일을 한번 빌드해서 실행해보겠습니다.
```
$ ./a.out
12 12
```
dd 변수는 int[3] 타입인데 k도 int[3] 타입이 된것 입니다. 타입을 복사하되 단순히 int 타입이라는것만 복사하는게 아니라 배열의 크기까지도 복사하기 때문에 per-cpu 변수로 배열을 쓰건 포인터를 쓰건 상관없이 똑같은 변수를 프로세서 갯수만큼 여러개 만들 수 있는 것입니다.

일단 DEFINE_PER_CPU는 .data.percpu 섹션에 변수를 하나 저장하는 일을 합니다. 그러면 프로세서 갯수만큼 변수를 여러개 만드는건 어디서 하는 걸까요?

그건 setup_per_cpu_areas() 함수를 보면 알 수 있습니다.

.data.percpu섹션의 크기는 위에 링커 스크립트에서 본대로 ```__per_cpu_end - __per_cpu_start``` 값이 됩니다. alloc_bootmem 함수로 섹션 크기만큼 메모리를 할당하고 .data.percpu 섹션을 복사합니다. cpu_pda[].data_offset에는 .data.percpu 섹션과 새로 할당된 메모리의 offset을 저장하는데, 나중에 cpu 번호를 알면 cpu_pda[cpu번호].data_offset 값과 변수의 포인터를 가지고 per-cpu 변수의 위치를 계산하게 됩니다.

예를 들어 .data.percpu 섹션의 시작 위치가 0x1000 이고, 크기가 0x1000이라고 가정해봅시다. 각 cpu별 per-cpu 변수가 저장될 메모리가 alloc_bootmeme으로 다음과 같이 할당되었다고하면,
```
cpu0 -> 0x2000
cpu1 -> 0x3000
cpu2 -> 0x4000
cpu3 -> 0x5000
```
각 프로세서의 data_offset은 다음고 같이 됩니다.
```
cpu[0].data_offset = 0x1000
cpu[1].data_offset = 0x2000
cpu[2].data_offset = 0x3000
cpu[3].data_offset = 0x4000
```

그러면 만약 foo라는 per-cpu변수의 원래 위치가 0x1010 이었다면, cpu0이 사용할 per_cpu_foo 변수의 위치는 0x1010 + 0x1000 = 0x2010 이 되고, cpu1은 0x3010 등이 될 것입니다.

이렇게 DEFINE_PER_CPU는 링커와 컴파일러의 기능을 사용해서 간단하게 프로세서별 메모리 영역을 만들어놓습니다.

##per-cpu 변수 사용하기

per-cpu 변수를 만들었으니 어떻게 쓰는지를 봐야겠지요.

가장 대표적인게 per_cpu매크로를 이용하는 방법입니다. per_cpu는 사실 DEFINE_PER_CPU 매크로에서 설명할 때 설명했던것 같이 원래 .data.percpu섹션에 있던 변수의 주소에 cpu_pda[cpu].data_offset을 더하는 작업을 매크로로 만든것 뿐입니다. 그런 주소값을 더하는 작업을 RELOC_HIDE라는 매크로로 처리합니다.

RELOC_HIDE 매크로의 정의를 보면 컴파일러마다 다르게 만들어놨는데요 주석을 읽어보면 gcc가 최초 .data.percpu 섹션에 정의된 변수의 주소에 이상한 값을 더하려한다는걸 알고 뭔가 최적화를 하거나 나름대로 뭔가를 더 하려고할 수 있는데, 그걸 방지하는게 목적이라고 써있습니다. gcc는 foo라는 변수가 0x1010 위치에 8바이트 크기로 저장된걸 알고있는데 갑자기 &foo + 0x1000 이라는 연산을 한다면 똑똑한 gcc는 에러라고 생각할 수 있다는 것입니다.

좀더 자세한 설명은 아래 참고자료를 읽어보세요.

참고: http://studyfoss.egloos.com/5374731

어쨌든 gcc가 아닌 intel 컴파일러용 코드 include/linux/compiler-intel.h를 보면 주소를 더할 뿐이라는걸 알 수 있습니다.

좀더 최신 버전에서 per-cpu 구현에 대한 설명도 참고 자료를 확인하시기 바랍니다.

참고: http://studyfoss.egloos.com/5375570

그럼 per_cpu 매크로를 사용하는 예제 코드를 보겠습니다.

v2.6.11 코드를 기준으로 recalc_bh_state라는 함수가 있습니다.

```
struct bh_accounting {
    int nr;			/* Number of live bh's */
	int ratelimit;		/* Limit cacheline bouncing */
};

static DEFINE_PER_CPU(struct bh_accounting, bh_accounting) = {0, 0};

static void recalc_bh_state(void)
{
	int i;
	int tot = 0;

	if (__get_cpu_var(bh_accounting).ratelimit++ < 4096)
		return;
	__get_cpu_var(bh_accounting).ratelimit = 0;
	for_each_cpu(i)
		tot += per_cpu(bh_accounting, i).nr;
	buffer_heads_over_limit = (tot > max_buffer_heads);
}
```

struct bh_accouting이라는 데이터구조는 버퍼헤드의 갯수 nr 정보를 가지고 있습니다. 만약 커널 전역 변수로 struct bh_accouting타입의 객체를 만들어놨으면 버퍼해드를 만들때마다 각 프로세서가 캐시를 날리고 메모리를 읽고 다른 프로세서의 캐시도 날리고 등의 작업을 할 것입니다.

그래서 커널에서는 DEFINE_PER_CPU로 bh_accouting 객체를 정적으로 만들어놨습니다. 그리고 recalc_bh_state 함수에서 각 cpu마다 bh_accounting 변수를 참조해서 총 갯수를 계산합니다. per_cpu 매크로의 결과가 포인터가 아니라는 것에 주의해야합니다.


##per-cpu 변수 동적으로 만들기

alloc_percpu을 이용해서 per-cpu변수를 동적으로 만들 수 있습니다. alloc_percpu함수 먼저 struct percpu_data 구조체의 객체를 만듭니다. struct percpu_data에는 void *ptrs[NR_CPUS] 배열이 있는데 프로세서 갯수만큼 요청받은 크기의 메모리를 할당합니다. percpu_data 객체는 시스템에 전역적인 객체이고, 이 객체가 각 프로세서별 데이터를 관리하는 것입니다. 그리고 percpu_data 객체를 반환합니다.
```
void *__alloc_percpu(size_t size, size_t align)
{
    int i;
	struct percpu_data *pdata = kmalloc(sizeof (*pdata), GFP_KERNEL);

	if (!pdata)
		return NULL;

	for (i = 0; i < NR_CPUS; i++) {
		if (!cpu_possible(i))
			continue;
		pdata->ptrs[i] = kmem_cache_alloc_node(
				kmem_find_general_cachep(size, GFP_KERNEL),
				cpu_to_node(i));

		if (!pdata->ptrs[i])
			goto unwind_oom;
		memset(pdata->ptrs[i], 0, size);
	}

	/* Catch derefs w/o wrappers */
	return (void *) (~(unsigned long) pdata);
```
그런데 마지막 반환값을 보면 pdata변수를 그대로 반환하는게 아니라 ~ 연산자를 써서 비트를 반전시켜서 반환합니다. 왜냐면 주석에도 설명했듯이 per-cpu 변수를 일반 변수 쓰듯이 접근하면 안되기때문에 비트를 반전시켜서 접근이 안되도록 한 것입니다.

per-cpu변수를 해지하는 함수는 free_percpu입니다. 해지할 객체의 ptrs 배열에 있는 각 프로세서별 메모리를 해지하고, 마지막으로 객체 자체를 해지합니다. free_percpu 코드의 시작 부분에는 전달된 per-cpu변수의 비트를 반전시킵니다. 그래야 원래의 per-cpu 변수에 접근할 수 있기 때문입니다.

alloc_percpu함수로 할당받은 per-cpu변수에서 프로세서별 변수에 접근하기 위해서per_cpu_ptr 매크로함수를 사용합니다. 
```
/* 
 * Use this to get to a cpu's version of the per-cpu object allocated using
 * alloc_percpu.  Non-atomic access to the current CPU's version should
 * probably be combined with get_cpu()/put_cpu().
 */ 
#define per_cpu_ptr(ptr, cpu)                   \
({                                              \
        struct percpu_data *__p = (struct percpu_data *)~(unsigned long)(ptr); \
        (__typeof__(ptr))__p->ptrs[(cpu)];    \
})
```
alloc_percpu함수로 받은 값의 비트를 반전시키면 percpu_data 객체를 얻을 수 있습니다. 그리고 percpu_data 객체의 ptrs[cpu] 값이 해당 프로세서용 변수의 포인터입니다.

이제 실제로 어떻게 사용하는지 예를 한번 보겠습니다. 다음은 include/linux/genhd.h 파일에 정의된 init_disk_stats 함수입니다.
```
static inline int init_disk_stats(struct gendisk *disk)
{
    disk->dkstats = alloc_percpu(struct disk_stats);
	if (!disk->dkstats)
		return 0;
	return 1;
}
static inline void free_disk_stats(struct gendisk *disk)
{
    free_percpu(disk->dkstats);
}
```

gendisk의 dkstats 필드에 disk_stats 타입의 per-cpu 변수를 생성합니다. 그리고 free_disk_stats함수에서 disk->dkstats 변수를 해지합니다. 다음은 per_cpu_ptr 함수의 예제입니다.

```
#define __disk_stat_add(gendiskp, field, addnd)     \
	(per_cpu_ptr(gendiskp->dkstats, smp_processor_id())->field += addnd)
```
참고로 smp_processor_id()는 이 코드가 실행된 프로세서의 번호를 반환해줍니다. 그리고 per_cpu_ptr함수의 결과값은 lvalue입니다. 따라서 함수의 결과값에 곧바로 ->같은 연산자를 적용할 수 있습니다.

