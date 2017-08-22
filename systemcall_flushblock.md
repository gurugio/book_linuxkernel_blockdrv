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
exit_to_usermode_loop calls do_signal() -> get_signal() -> task_work_run() functions.
task_Work_run() function executes callback functions in task_works list of struct task_struct.
We already know what callback function we should find. 
It is `____fput`.
So let's find what adds `____fput` into the task_works list..

You can use grep tool or reverse-reference feature of tagging tool.
Yes, only fput() uses `____fput` to create a work via init_task_work().
And we know close system call calls fput().

Finally we understand what flushes writing IOs
* write system call stores data on the page cache (delayed IOs)
* close system call calls fput()
* fput() registers `____fput` work
* close system call is finished with int_ret_from_sys_call
* task_work_run() executes `____fput`
* `____fput` flushes delayed IOs

Can you draw a big picture from system call to block driver?

When mybrd device file is closed, every delayed IO are processed.
But what happens if normal file of disk-based filesystem is closed?
Now you know what you should find in filesystem sources.
Yes, address_space_operations and file_operations.
Please investigate ext2 or ext4 filesystem for yourself.
