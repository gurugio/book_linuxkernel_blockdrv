# 시스템콜과 블럭 장치 flush

뜬금없이 왜 시스템콜을 이야기하는지 이상하게 생각하실것 같은데요 왜냐면 우리가 콜스택을 추적해볼때 시스템콜이 나왔기 때문입니다.

아래는 dd로 mybrd장치에 쓰기를 했을 때 출력된 콜스택입니다.
```
[  194.612304] Call Trace:
[  194.612474]  [<ffffffff8130dadf>] dump_stack+0x44/0x55
[  194.612833]  [<ffffffff814fac32>] mybrd_make_request_fn+0x42/0x260
[  194.613306]  [<ffffffff812efc8e>] generic_make_request+0xce/0x1a0
[  194.613761]  [<ffffffff812efdc2>] submit_bio+0x62/0x140
[  194.614158]  [<ffffffff8119fa78>] submit_bh_wbc.isra.38+0xf8/0x130
[  194.614571]  [<ffffffff811a188d>] __block_write_full_page.constprop.43+0x10d/0x3a0
[  194.615098]  [<ffffffff810ac8d5>] ? add_timer_on+0xd5/0x130
[  194.615523]  [<ffffffff811a1e50>] ? I_BDEV+0x10/0x10
[  194.615858]  [<ffffffff811a1e50>] ? I_BDEV+0x10/0x10
[  194.616238]  [<ffffffff811a1c60>] block_write_full_page+0x140/0x160
[  194.616654]  [<ffffffff811a2853>] blkdev_writepage+0x13/0x20
[  194.617107]  [<ffffffff8112344e>] __writepage+0xe/0x30
[  194.617518]  [<ffffffff81123e67>] write_cache_pages+0x1d7/0x4c0
[  194.617942]  [<ffffffff81194a30>] ? __mark_inode_dirty+0x2c0/0x310
[  194.618412]  [<ffffffff81123440>] ? domain_dirty_limits+0x120/0x120
[  194.618798]  [<ffffffff8111a4ee>] ? unlock_page+0x5e/0x60
[  194.619222]  [<ffffffff8112418c>] generic_writepages+0x3c/0x60
[  194.619616]  [<ffffffff81125ed9>] do_writepages+0x19/0x30
[  194.619972]  [<ffffffff8111b2ec>] __filemap_fdatawrite_range+0x6c/0x90
[  194.620456]  [<ffffffff8111b357>] filemap_write_and_wait+0x27/0x70
[  194.620923]  [<ffffffff811a29e9>] __blkdev_put+0x69/0x220
[  194.621325]  [<ffffffff811a3156>] ? blkdev_write_iter+0xd6/0x100
[  194.621723]  [<ffffffff811a2f97>] blkdev_put+0x47/0x100
[  194.622102]  [<ffffffff811a3070>] blkdev_close+0x20/0x30
[  194.622502]  [<ffffffff8116f7e7>] __fput+0xd7/0x1e0
[  194.622851]  [<ffffffff8116f929>] ____fput+0x9/0x10
[  194.623227]  [<ffffffff810702ae>] task_work_run+0x6e/0x90
[  194.623585]  [<ffffffff810021a2>] exit_to_usermode_loop+0x92/0xa0
[  194.624041]  [<ffffffff81002b2e>] syscall_return_slowpath+0x4e/0x60
[  194.624567]  [<ffffffff8188f30c>] int_ret_from_sys_call+0x25/0x8f
```
콜스택의 처음 시작이 int_ret_from_sys_call입니다. 딱 이름부터 뭔가 시스템콜과 연관이 있을것 같지 않나요? 우리는 이 콜스택이 페이지캐시에 있는 페이지들의 데이터가 장치로 전달되는 과정이라는걸 알았습니다. 그런데 이런 데이터 flush가 정확히 언제언제 발생되는건지는 아직 알지 못합니다.

그리고 분석하다보니 read나 write 등 시스템콜부터 드라이버까지 대강 따라가봤는데 대강 어떻게 시스템콜들이 만들어지는지는 알아두면 좋을것 같습니다.

시스템콜 호출에 대해서는 최근에 많이 바껴져서 우리도 확인했듯이 SYSCALL_DEFINE같은 매크로를 쓰기도하고 어셈블리 코드도 들어가는 등 분석이 쉽지 않습니다. 우리만 그렇게 생각하는게 아니기 때문에 이런 좋은 자료들이 만들어지는 것이겠지요.
* https://lwn.net/Articles/604287/
* https://lwn.net/Articles/604515/

커널을 분석하다보면 이건 좀 복잡한데, 뭔가 정리된게 없을까하고 생각하면 거의 대부분 좋은 문서들이 있습니다. 그리고 그 문서들의 대부분은 lwn 사이트에 있구요.

여튼 이렇게 좋은 문서가 있으니 이 강좌에서는 코드만 따라가면서 큰 흐름만 보겠습니다. 큰 흐름을 보고나서 이 문서들을 보면 더 잘 이해가 될 것입니다.

##시스템콜 초기화
이번 강좌는 인텔프로세서와 어셈블리에 대한 내용이 들어갑니다. 익숙하지않으시면 넘기셔도 좋습니다.

###syscall_init

참고문서 https://lwn.net/Articles/604287/ 에서 설명하듯이 시스템콜을 초기화하는 함수는 syscall_init입니다. 저는 참고문서를 보기전에 int_ret_from_sys_call을 호출하는 함수들을 추적하다가 syscall_init에서 시스템콜을 초기화한다는걸 알았습니다. 코드로 알던 문서로 알던 상관은 없겠지요.

wrmsrl이라는 함수가 나오는데 결국 wrmsr 이라는 어셈블리 명령을 실행하는 함수입니다.

```
static inline void native_write_msr(unsigned int msr,
    			    unsigned low, unsigned high)
{
	asm volatile("wrmsr" : : "c" (msr), "a"(low), "d" (high) : "memory");
}
```
wrmsr이 뭔지는 인텔프로세서 메뉴얼을 참고하셔도 좋고 구글로 검색해서 http://x86.renejeschke.de/html/file_module_x86_id_326.html 같은 사이트를 찾아봐도 좋습니다. 어쨌든 wrmsr 명령은 유저레벨 어플이 시스템콜을 호출했을 때 프로세서가 알아서 점프해야할 주소를 프로세서의 특수 레지스터에 저장하는 명령입니다. ecx레지스터에는 특수 레지스터의 주소를 저장하고, edx:eax 레지스터에 64비트의 주소값을 32비트씩 나눠서 저장하면 됩니다. 이런 어셈블리 명령 호출을 C 함수로 wrapper를 만든게 바로 wrmsrl입니다.

유저어플에서 어떤 어셈블리 명령을 쓰는지는 상관할 필요가 없고, 어쨌든 미리 정해진 규칙에 따라 시스템콜을 호출한다는 것만 생각하면 됩니다. 그럼 프로세서는 알아서 wrmsrl 함수로 전달된 entry_SYSCALL_64 주소로 점프합니다.

###entry_SYSCALL_64

그럼 entry_SYSCALL_64는 뭘까요? 함수인지 뭔지 grep으로 찾아보겠습니다. 
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

결론은 int_ret_from_sys_call은 시스템콜이 끝날때 호출되므로 시스템콜이 끝날때 블럭장치의 페이지캐시가 flush된다는 것입니다.

참고로 sys_call_table은 함수 포인터의 배열인데 C 코드에서 정의하고있지 않습니다. arch/x86/entry/syscall_64.c파일에 보면 sys_call_table 배열이 정의되어있는데, 그 값들이 하드코딩된게 아니라  #include <asm/syscalls_64.h> 만 써있습니다. syscalls_64.h 파일을 찾아보면 arch/x86/include/generated/asm/syscalls_64.h에 보일수도 있고 안보일 수도 있습니다. 결론적으로는 커널이 빌드될 때 arch/x86/entry/syscalls/Makefile 파일이 실행되고, syscalltbl.sh 스크립트를 이용해서 syscall_64.tbl 파일을 읽고, syscalls_64.h 파일을 생성합니다. 시스템콜이 언제든 추가될 수 있으니 이렇게 동적으로 테이블을 만들도록 구현한 것입니다.

## 그럼 어떤 시스템콜이 블럭 장치를 flush할까?

시스템콜이 끝날때 블럭 장치의 flush가 발생했다는건 알았으니 콜스택을 한번 따라가보겠습니다. 
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
