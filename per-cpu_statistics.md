# per-cpu variable and statistics information(v2.6.11)

We saw `__inc_zone_page_state(page, NR_FILE_PAGES)` function in previous chapter that updates statistics information of a zone including the specified page.
You can see other statistics entries in include/linuc/mmzone.h file.

Some user applications implements statistics with atomic variable or shared variable based on spinlock.
They have poor scalability on multicore system.

For better scalability on many core, kernel supports per-cpu variable.
For example, `__inc_zone_page_state()` updates per-cpu variable corresponding to the working core.
And when user reads /proc/zoneinfo file, kernel handler sums values of all cores and shows the total value.

This chapter shortly describes implementation of per-cpu variable on v2.6.11.
(I tried to describe v4.4 but it's too complex and requires knowledge for complier and linker.
I think if you understand per-cpu implementation of v2.6, you could look into the implementation on 4.4 for yourself).

reference
* http://www.makelinux.net/ldd3/chp-8-sect-5

## static definition of per-cpu variable

### DEFINE_PER_CPU and per-cpu section

Let's start with where per-cpu is stored.
DEFINE_PER_CPU(type, name) macro creates a per-cpu variable with the specified type and name.
As you can see, it defines the per-cpu variable statically at compile time.

```
#define DEFINE_PER_CPU(type, name) \
    __attribute__((__section__(".data.percpu"))) __typeof__(type) per_cpu__##name
```

There are some GCC attributes.
* ```__attribute__```: inform GCC that there will be GCC attributes
* ```__section__```: specify section of the variable
* ```__typeof__```: return type of the variable

Following example is showing what is section and how we can use `__typeof__`.

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

Following is how to build the example and the result of "readelf -S" command.
"readelf -S" command shows information of sections in the file.

```
$ gcc a.c
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

A section is a kind of area in the file.
So "readelf -S" shows the start and end location of each section.
You might be already used to .data, .bss and .text sections.
`__attribute__ ((__section__ ("section-name")))` stores a variable in the specified section.

So what sections kernel has?
A linker file arch/x86_64/kernel/vmlinux.lds.S declares sections of kernel.
And we can find a section for per-cpu variables, so called as .data.percpu.

```
125   __per_cpu_start = .;
126   .data.percpu  : { *(.data.percpu) }
127   __per_cpu_end = .;
128   . = ALIGN(4096);
```
lds파일은 링커에게 어떤 섹션을 어떻게 만들지 등을 알려주는 파일입니다. 그리고 .data.percpu 섹션 이름 위아래에 __per_cpu_start와 __per_cpu_end가 있는데 각각 섹션의 시작 위치와 끝 위치를 저장한 변수입니다. 나중에 커널 코드에서 변수처럼 사용할 것입니다.

Linker reads vmlinux.lds file and makes .data.percpu section.
`__per_cpu_start` and `__per_cpu_end` defines the start and end address of the section.

Finally we understand that DEFINE_PER_CPU macro stores variable in .data.percpu section.


그건 setup_per_cpu_areas() 함수를 보면 알 수 있습니다.

.data.percpu섹션의 크기는 위에 링커 스크립트에서 본대로 ```__per_cpu_end - __per_cpu_start``` 값이 됩니다. alloc_bootmem 함수로 섹션 크기만큼 메모리를 할당하고 .data.percpu 섹션을 복사합니다. cpu_pda[].data_offset에는 .data.percpu 섹션과 새로 할당된 메모리의 offset을 저장하는데, 나중에 cpu 번호를 알면 cpu_pda[cpu번호].data_offset 값과 변수의 포인터를 가지고 per-cpu 변수의 위치를 계산하게 됩니다.

Then let's check setup_per_cpu_ares() function that initializes per-cpu variables in .data.percpu section.
It checks the size of .data.percpu section with `__per_cpu_end - __per_cpu_start`.
Next it allocates memory for each core with alloc_bootmem() and copies the section into the allocated memory.
Finally cpu_pds[core].data_offset has address of per-cpu area of the core.

For example, each core allocates per-cpu memory like following with alloc_bootmem(),
```
cpu0 -> 0x2000
cpu1 -> 0x3000
cpu2 -> 0x4000
cpu3 -> 0x5000
```

each core has data_offset as following.
```
cpu[0].data_offset = 0x1000
cpu[1].data_offset = 0x2000
cpu[2].data_offset = 0x3000
cpu[3].data_offset = 0x4000
```

If foo variables is stored at offset 0x10 in .data.percpu, foo of cpu0 is 0x1010, cpu1 0x2010 and so on.

## how to use per-cpu variable

per-cpu 변수를 만들었으니 어떻게 쓰는지를 봐야겠지요.

가장 대표적인게 per_cpu매크로를 이용하는 방법입니다. per_cpu는 사실 DEFINE_PER_CPU 매크로에서 설명할 때 설명했던것 같이 원래 .data.percpu섹션에 있던 변수의 주소에 cpu_pda[cpu].data_offset을 더하는 작업을 매크로로 만든것 뿐입니다. 그런 주소값을 더하는 작업을 RELOC_HIDE라는 매크로로 처리합니다.

RELOC_HIDE 매크로의 정의를 보면 컴파일러마다 다르게 만들어놨는데요 주석을 읽어보면 gcc가 최초 .data.percpu 섹션에 정의된 변수의 주소에 이상한 값을 더하려한다는걸 알고 뭔가 최적화를 하거나 나름대로 뭔가를 더 하려고할 수 있는데, 그걸 방지하는게 목적이라고 써있습니다. gcc는 foo라는 변수가 0x1010 위치에 8바이트 크기로 저장된걸 알고있는데 갑자기 &foo + 0x1000 이라는 연산을 한다면 똑똑한 gcc는 에러라고 생각할 수 있다는 것입니다.


어쨌든 gcc가 아닌 intel 컴파일러용 코드 include/linux/compiler-intel.h를 보면 주소를 더할 뿐이라는걸 알 수 있습니다.


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

