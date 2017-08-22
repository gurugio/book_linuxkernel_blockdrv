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

## static definition of per-cpu variable with DEFINE_PER_CPU macro

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

Linker reads vmlinux.lds file and makes .data.percpu section.
`__per_cpu_start` and `__per_cpu_end` defines the start and end address of the section.

Finally we understand that DEFINE_PER_CPU macro stores variable in .data.percpu section.

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

Let's see how we can use per-cpu variable.
The most common way is using per_cpu macro that uses RELOC_HIDE macro that calculate the address of the variable with cpu_pda[cpu].data_offset.

Let's take a look at real code.
Following is recalc_bh_state() function in v2.6.11.
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

struct bh_accounting has the number of buffer heads in nr field.
If a object of struct bh_accounting is global variable, all threads on all CPUs accesses that object.
It corrupts caches of all CPUs and generates so many cache coherent communication.

So kernel defines bh_accounting object with DEFINE_PER_CPU.
recalc_bh_state() function sums up bh_accounting values of all CPUs.

## dynamically define per-cpu variable

per-cpu variable can be defined dynamically with alloc_percpu().
It creates a object of struct percpu_data and allocates memory as much as CPUs for ptrs[NR_CPUs] array.
percpu_data is a global object of kernel and manages per-cpu data.

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

Please notice that it returns the negative value of the address of percpu_data object.
per-cpu variables should not be accessed directly, but via macro functions for per-cpu variable.
Accessing the return value directly will be generated error.

free_percpu() function frees per-cpu variable.
It frees per-CPU memory and then frees percpu_data object.

We should use per_cpu_ptr macro to access the per-cpu variable allocated by alloc_percpu().
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

As you can see, it just returns the address in the array of ptrs[].

Following is an example to create dynamic per-cpu variable.
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

struct gendisk has statistics information at dkstats field that is allocated via alloc_percpu().
And it is freed via free_percpu().

And following is an example to update the per-cpu value.
```
#define __disk_stat_add(gendiskp, field, addnd)     \
	(per_cpu_ptr(gendiskp->dkstats, smp_processor_id())->field += addnd)
```

smp_processor_id() returns the CPU number.
per_cpu_ptr() returns lvalue, so we can use -> operator to the per_cpu_ptr() macro.

