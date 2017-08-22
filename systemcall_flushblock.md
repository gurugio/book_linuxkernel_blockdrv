# system call and block device flushing

System call is usually starting point of device operations as we investigated in previous chapters.
Let's look into the system call in this chapter.

In the old kernel version, system calls were implemented simply.
But the latest version uses macro SYSCALL_DEFINE to define system calls and other optimization techniques.
So following documents are good references to understand system call implementation.
* https://lwn.net/Articles/604287/
* https://lwn.net/Articles/604515/

FYI, if you feel something difficult to investigate, there should be good reference for it, because somebody else feels the same.
And many of good documents are in lwn.org.

This chapter shows only code flow briefly.
After reading this chapter, you would be able to understand above document better.

## system call initialization

You need a little knowledge for x86 assembly.

### syscall_init

As you can see in https://lwn.net/Articles/604287/, system calls are initialized by syscall_init().

```
void syscall_init(void)
{
	wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
	wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
... skip ...
```

Entry point of all system calls is initialized with wrmsrl() function that is wrapper of wrmsr assembly instruction.

```
static inline void native_write_msr(unsigned int msr,
    			    unsigned low, unsigned high)
{
	asm volatile("wrmsr" : : "c" (msr), "a"(low), "d" (high) : "memory");
}
```

You can check http://x86.renejeschke.de/html/file_module_x86_id_326.html or Intel Processor Manuals to find what wrmsr does.
In short, it stores an address, entry_SYSCALL_64, in a special register MSR_LSTAR.
Now when application calls system call, processor automatically jumps to entry_SYSCALL_64 address.

### entry_SYSCALL_64

Let's find entry_SYSCALL_64 with grep tool.
```
$ grep entry_SYSCALL_64 * -IR
arch/x86/entry/entry_64.S:ENTRY(entry_SYSCALL_64)
arch/x86/entry/entry_64.S:GLOBAL(entry_SYSCALL_64_after_swapgs)
arch/x86/entry/entry_64.S:entry_SYSCALL_64_fastpath:
arch/x86/entry/entry_64.S:    jmp	entry_SYSCALL_64_fastpath	/* and return to the fast path */
arch/x86/entry/entry_64.S:END(entry_SYSCALL_64)
arch/x86/include/asm/proto.h:void entry_SYSCALL_64(void);
arch/x86/kernel/cpu/common.c:	wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
arch/x86/xen/xen-asm_64.S:	jmp entry_SYSCALL_64_after_swapgs
System.map:ffffffff8188b810 T entry_SYSCALL_64
System.map:ffffffff8188b813 T entry_SYSCALL_64_after_swapgs
System.map:ffffffff8188b85c t entry_SYSCALL_64_fastpath
```
arch/x86/include/asm/proto.h에 함수로 선언을 해놨네요. 그리고 arch/x86/entry/entry_64.S 파일에 코드가 있습니다. ENTRY라는 매크로와 END라는 매크로가 어셈블리 코드에서 함수의 시작과 끝을 표시하는 매크로라고 생각하면 됩니다. 자세히 볼 필요는 없으니 그냥 간단히만 알아보겠습니다.

시스템콜 자체를 보려는게 아니라 int_ret_from_sys_call이 어디서 언제 호출되는지를 알려는게 목적이니까 entry_SYSCALL_64 자체를 분석할 필요는 없습니다. 중요한건 시스템콜이 호출되면 entry_SYSCALL_64가 호출된다는 것입니다. 그리고 entry_SYSCALL_64에서 sys_call_table을 이용해서 시스템콜을 호출하고, 마지막으로 int_ret_from_sys_call을 호출한다는 것입니다.

We can find entry_SYSCALL_64 in arch/x86/entry/entry_64.S file.
ENTRY and END macros defines starting point and end point of macro function in assembly file.

We don't need to look into every instruction.
Core part is calling `*sys_call_table(, %rax, 8)`.
sys_call_table is an array of addresses of system calls, and rax has system call number and 8 is size of one entry of the table.
The entry point of all system calls are the same but `call	*sys_call_table(, %rax, 8)` instruction jumps to corresponding system call.

Where is sys_call_table?
It's a little bit difficult to find the definition of sys_call_table because it's not code in source file.
After building kernel, arch/x86/include/generated/asm/syscalls_64.h is generated automatically by arch/x86/entry_syscalls/Makefile that executes syscalltbl.sh script with syscall_64_tbl.

## System call to flush IO

Do you remember that callstack of writing data to disk starts with int_ret_from_sys_call?
```
/*
 * Syscall return path ending with IRET.
 * Has correct iret frame.
 */
GLOBAL(int_ret_from_sys_call)
```

It is the end point of all system calls.
So we can understand that flushing writing IO is done at the end of system call.

Let's check again the callstack of writing disk.

```
WRITE: int_ret_from_sys_call
--> syscall_return_slowpath
--> exit_to_usermode_loop
--> task_work_run
--> __fput
```

exit_to_usermode_loop에서 곧바로 task_work_run을 호출하는게 아니라 do_signal -> get_signal -> task_work_run 순서로 호출됩니다.

task_work_run은 무한 루프를 돌면서 struct task_struct 구조체의 task_works 리스트에 저장된 callback_head 들을 찾아내서 콜백함수를 실행합니다. 우리는 어떤 콜백함수가 호출되었는지 미리 알고있습니다. ```____fput```이지요. 그러니 어디에서 task_work_add 함수를 호출해서 ```____fput```을 task_works 리스트에 추가하는지만 찾으면 됩니다.

그럴때는 태깅툴에서 역참조 기능을 사용하면 편리합니다. ```____fput```을 호출하는 함수는 유일하게 fput 뿐이네요. fput 함수 바디에 init_task_work 함수로 ```____fput```을 호출하는 callback_head를 만드는걸 확인할 수 있습니다.

그럼 fput은 또 언제 호출된걸까요? fput의 역참조를 확인해봅니다. 굉장히 많은 코드에서 fput을 호출합니다만 우리는 dd 툴이 open/read/write/close하는 굉장히 단순한 툴인걸 알기때문에 시스템콜중에 하나라고 추측이 가능합니다.

그래서 결론적으로 제 생각에 dd에서 mybrd 장치를 닫을때 close 시스템콜을 호출했고, 그때 fput이 호출되었다고 생각됩니다. 그리고 장치파일을 닫을 때 file->f_op->release를 호출할 것이고 지금까지 몇번을 반복했던 것처럼 def_blk_fops와 def_blk_aops에 정의된 콜백 함수들이 호출되었을 것입니다.

그리고 여기에서 우리는 장치 파일이 닫혀질때 기본적으로 해당 장치 파일의 페이지 캐시들이 모두 flush된다는걸 알 수 있습니다. 당연하겠지요. 더이상 장치 파일을 사용하지않는데 장치 파일의 데이터를 메모리에 가지고있어봐야 낭비겠지요.

일반 파일이 닫힐때는 어떨까요? ext2같은 파일시스템이 어떤 file_operations 테이블과 address_space_operations 테이블을 정의했는지 확인해서 함수들을 따라가보면 페이지캐시를 flush하는지 안하는지 확인이될 것입니다.

파일시스템마다 정책이 다를 수도 있겠지요. 어떤 파일시스템은 파일이 닫혀도 나중에 다시 열릴걸 대비해서 페이지캐시를 가지고 있을 수도 있을 것입니다. 그렇게 파일시스템마다 페이지캐시의 정책을 결정할 수 있도록 address_space_operations을 파일시스템마다 별도로 정의할 수 있도록 만든 것입니다.

커널은 코드 한줄한줄 그냥 만든게 없습니다. 이미 이십년이 넘게 적어도 수천명의 개발자들이 매일 업무시간 취미시간에 리뷰하고 또 리뷰한 코드들입니다. 그리고 수백만 수천만 몇개가 될지 모르는 머신에서 24시간 동작하는 코드이구요. 그런걸 생각하면 커널 코드를 분석할 때 의미를 따져보는게 재미있고, 배우는 것도 많습니다.
