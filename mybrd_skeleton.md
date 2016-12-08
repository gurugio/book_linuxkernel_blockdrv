# mybrd 드라이버의 뼈대 빌드

이제 준비는 끝났으니 mybrd 드라이버를 만들기 시작합니다. 일단 아주 간단한 껍데기만 완성해서 우리가 만든 드라이버가 제대로 실행되는지 확인하고, 커널이 어떻게 드라이버를 빌드하고 실행시키는지를 알아보겠습니다. 커널 소스를 조금씩 보기 시작합니다. Linux device driver나 Understanding the Linux kernel 등의 참고 도서들도 같이 보시면 더 좋습니다. 설명하려는 커널의 버전이 다르므로 책과는 약간 다른 코드들이 보일 것입니다. 하지만 만약 제 설명에 이상한게 있으면 무조건 참고 도서들의 설명을 기준으로 이해하시면 됩니다.

## 커널에 mybrd.c 소스 추가

커널에 드라이버를 추가하는 방법을 소개하겠습니다. 아주 간단합니다. 리눅스 커널은 이미 전세계 개발자들이 이십년이 넘게 개발해왔으므로 개발하기 편리한 도구들을 모두 갖추고 있습니다.

드라이버들은 커널의 drivers 디렉토리에 있습니다. ls drivers 명령을 실행해보면 많은 디렉토리들이 있습니다. 그 중에서 우리는 블럭장치의 드라이버를 만들 것이므로 drivers/block 디렉토리에 드라이버 소스를 저장할 것입니다.
```
$ ls drivers/block/
amiflop.c     cpqarray.h    loop.o           paride       swim_asm.S
aoe           cryptoloop.c  Makefile         pktcdvd.c    swim.c
ataflop.c     DAC960.c      mg_disk.c        ps3disk.c    sx8.c
brd.c         DAC960.h      modules.builtin  ps3vram.c    umem.c
brd.o         drbd          modules.order    rbd.c        umem.h
built-in.o    floppy.c      mtip32xx         rbd_types.h  virtio_blk.c
cciss.c       hd.c          mybrd.c          rsxx         virtio_blk.o
cciss_cmd.h   ida_cmd.h     mybrd.o          skd_main.c   xen-blkback
cciss.h       ida_ioctl.h   nbd.c            skd_s1120.h  xen-blkfront.c
cciss_scsi.c  Kconfig       null_blk.c       smart1,2.h   xsysace.c
cciss_scsi.h  loop.c        null_blk.o       sunvdc.c     z2ram.c
cpqarray.c    loop.h        osdblk.c         swim3.c      zram
```
위에서 보는 것과 같이 mybrd.c 파일을 하나 만듭니다. 아직은 아무런 코드도 넣을 필요없이 그냥 빈 파일을 만들면 됩니다. 그리고 커널이 빌드될 때 mybrd.c도 빌드되도록 다음처럼 Kconfig와 Makefile을 수정합니다.
```
$ git diff drivers/block/Kconfig
diff --git a/drivers/block/Kconfig b/drivers/block/Kconfig
index 29819e7..6744d48 100644
--- a/drivers/block/Kconfig
+++ b/drivers/block/Kconfig
@@ -365,6 +365,9 @@ config BLK_DEV_RAM
          Most normal users won't need the RAM disk functionality, and can
          thus say N here.
 
+config BLK_DEV_MYRAM
+       tristate "myRAM block device support"
+
 config BLK_DEV_RAM_COUNT
        int "Default number of RAM disks"
        default "16"
gurugio@giohnote:~/linux-torvalds$ git diff drivers/block/Makefile
diff --git a/drivers/block/Makefile b/drivers/block/Makefile
index 6713290..ec939f2 100644
--- a/drivers/block/Makefile
+++ b/drivers/block/Makefile
@@ -14,6 +14,9 @@ obj-$(CONFIG_PS3_VRAM)                += ps3vram.o
 obj-$(CONFIG_ATARI_FLOPPY)     += ataflop.o
 obj-$(CONFIG_AMIGA_Z2RAM)      += z2ram.o
 obj-$(CONFIG_BLK_DEV_RAM)      += brd.o
+obj-$(CONFIG_BLK_DEV_MYRAM)    += mybrd.o
+
+
 obj-$(CONFIG_BLK_DEV_LOOP)     += loop.o
 obj-$(CONFIG_BLK_CPQ_DA)       += cpqarray.o
 obj-$(CONFIG_BLK_CPQ_CISS_DA)  += cciss.o
```
Kconfig 파일은 우리가 커널을 빌드하기 전에 커널의 옵션을 조정하기 위해 make menuconfig를 했을 때 나타날 메뉴를 만드는 파일입니다. 위와 같이 수정하고 커널 소스가 있는 디렉토리에서 make menuconfig를 실행한 다음 Device Drivers ---> Block devices 메뉴에 들어가보면 우리가 추가한 대로 myRAM block device support라는 옵션이 나타납니다. 아래 화살키를 눌러서 메뉴를 선택한 다음 스페이스바를 눌러서 메뉴를 선택합니다. M으로 표시되게 하면 안됩니다. 반드시 *가 표시되도록 선택해야합니다. M으로 빌드하면 커널 이미지에 포함되는게 아니라 별도의 .ko 파일로 빌드되므로 우리 환경에서 커널 부팅과 함께 즉시 실행해볼 수가 없습니다. 드라이버를 커널에 포함시키느냐 아니면 별도의 모듈로 빌드하느냐에 대한 것도 꽤 긴 내용이므로 다른 참고서적을 확인하시기 바랍니다.
```
 .config - Linux/x86 4.4.0 Kernel Configuration
 > Device Drivers > Block devices >  ─────────────────────────────────────────────
  ┌───────────────────────────── Block devices ─────────────────────────────┐
  │  Arrow keys navigate the menu.  <Enter> selects submenus ---> (or empty │  
  │  submenus ----).  Highlighted letters are hotkeys.  Pressing <Y>        │  
  │  includes, <N> excludes, <M> modularizes features.  Press <Esc><Esc> to │  
  │  exit, <?> for Help, </> for Search.  Legend: [*] built-in  [ ]         │  
  │ ┌─────────────────────────────────────────────────────────────────────┐ │  
  │ │    --- Block devices                                                │ │  
  │ │    <*>   Null test block driver                                     │ │  
  │ │    < >   Normal floppy disk support                                 │ │  
  │ │    <*>   Loopback device support                                    │ │  
  │ │    (8)     Number of loop devices to pre-create at init time        │ │  
  │ │    < >     Cryptoloop Support                                       │ │  
  │ │          *** DRBD disabled because PROC_FS or INET not selected *** │ │  
  │ │    <*>   RAM block device support                                   │ │  
  │ │    <*>   myRAM block device support                                 │ │  
  │ │    (2)   Default number of RAM disks                                │ │  
  │ │    (4096) Default RAM disk size (kbytes)                            │ │  
  │ │    < >   Packet writing on CD/DVD media                             │ │  
  │ │    [ ]   Very old hard disk (MFM/RLL/IDE) driver  
```

오른쪽 화살표를 누르면 Help 메뉴를 선택할 수 있는데 엔터를 누르면 아래와 같은 화면이 나타날 것입니다.
```
 .config - Linux/x86 4.4.0 Kernel Configuration
 > Device Drivers > Block devices ─────────────────────────────────────────────
  ┌────────────────────── myRAM block device support ───────────────────────┐
  │ There is no help available for this option.                             │  
  │ Symbol: BLK_DEV_MYRAM [=y]                                              │  
  │ Type  : tristate                                                        │  
  │ Prompt: myRAM block device support                                      │  
  │   Location:                                                             │  
  │     -> Device Drivers                                                   │  
  │       -> Block devices (BLK_DEV [=y])                                   │  
  │   Defined at drivers/block/Kconfig:368                                  │  
  │   Depends on: BLK_DEV [=y]       
```
이 옵션을 선택하면 CONFIG_BLK_DEV_MYRAM 이라는 빌드 상수가 y값이 되고, 결국 Makefile에서 CONFIG_BLK_DEV_MYRAM 값을 y로 받고 mybrd.o를 생성하게 됩니다. mybrd.o는 당연히 mybrd.c를 이용해서 빌드하겠지요.

물론 이런 메뉴 화면을 만들거나 메뉴를 선택한게 빌드 옵션으로 추가되는걸 만드려면 많은 일을 해야하지만, 새로운 드라이버를 추가하는 일은 이렇게 간단합니다. 메뉴를 추가하고 빌드할 파일을 추가하면 끝입니다.

우리가 선택한 커널 옵션들은 최종적으로 .config 파일에 저장됩니다. .config에 BLK_DEV_MYRAM이 추가됐는지 확인하려면 아래처럼 .config 파일을 열어보면 됩니다.
```
$ grep MYRAM .config
CONFIG_BLK_DEV_MYRAM=y
```
제대로 y가 됐으면 드라이버 파일의 추가는 끝난 것입니다.

옵션이 제대로 설정되었으면 이제 드라이버가 제대로 빌드되는지를 확인할 차례겠지요. 빈 파일이지만 빌드가 되는지 확인해볼 수 있습니다.
```
$ make bzImage -j4
  CHK     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
  CHK     include/generated/timeconst.h
  CHK     include/generated/bounds.h
  CHK     include/generated/asm-offsets.h
  CALL    scripts/checksyscalls.sh
  CHK     include/generated/compile.h
  CC      drivers/block/mybrd.o
  LD      drivers/block/built-in.o
  LD      drivers/built-in.o
  LINK    vmlinux
  LD      vmlinux.o
  MODPOST vmlinux.o
  GEN     .version
  CHK     include/generated/compile.h
  UPD     include/generated/compile.h
  CC      init/version.o
  LD      init/built-in.o
  KSYM    .tmp_kallsyms1.o
  KSYM    .tmp_kallsyms2.o
  LD      vmlinux
  SORTEX  vmlinux
  SYSMAP  System.map
  VOFFSET arch/x86/boot/voffset.h
  CC      arch/x86/boot/version.o
  OBJCOPY arch/x86/boot/compressed/vmlinux.bin
  GZIP    arch/x86/boot/compressed/vmlinux.bin.gz
  MKPIGGY arch/x86/boot/compressed/piggy.S
  AS      arch/x86/boot/compressed/piggy.o
  LD      arch/x86/boot/compressed/vmlinux
  ZOFFSET arch/x86/boot/zoffset.h
  OBJCOPY arch/x86/boot/vmlinux.bin
  AS      arch/x86/boot/header.o
  LD      arch/x86/boot/setup.elf
  OBJCOPY arch/x86/boot/setup.bin
  BUILD   arch/x86/boot/bzImage
Setup is 15708 bytes (padded to 15872 bytes).
System is 2555 kB
CRC cac8eadd
Kernel: arch/x86/boot/bzImage is ready  (#20)
```
drivers/block/mybrd.o 파일이 빌드돼었다고 나옵니다. 그리고 bzImage 커널 파일의 빌드도 완료됐습니다.

##드라이버의 시작 코드 구현

예제 코드는 몇번 더 수정될 수 있으므로 여기서 복사해놓지 않고 github에 ch01-skel 브랜치를 만들어서 관리하겠습니다. git에 익숙하지 않으시면 아래 링크로 확인하시면 됩니다.

https://github.com/gurugio/mybrd/blob/ch01-skel/mybrd.c

###헤더

각종 헤더들이 있습니다.

linux/init.h, linux/module.h, linux/moduleparam.h 등의 헤더는 모든 드라이버에 공통적으로 들어가야할 파일들입니다. init.h이 뭔지 맛보기만 해볼까요?

커널 소스에서 include/linux/init.h 파일을 열어보겠습니다. v4.4 기준으로 보면 가장 먼저 등장하는 내용은 __init 이라는 매크로가 뭔지에 대한 주석입니다. 예를 들어 함수 이름앞에 __init이라고 쓰면 __section(.init.text)로 선언된다는 것입니다. 그럼 __section은 뭐고 .init.text는 뭘까요?

이렇게 드라이버 소스에 있는 코드 하나하나를 커널 소스에서 찾아보고 알아내는 일이 리눅스 커널을 분석하는 일입니다. cscope나 global로 ```__section```의 정의를 따라가보면 ```__attribute__ ((__section__ (#S)))```라는걸 알 수 있습니다. ```__attribute__```는 따로 정의가 없습니다. 왜냐면 gcc 컴파일러가 사용하는 키워드이기 때문입니다. ```__section__```로 gcc의 키워드이고 #S는 ```__section```의 인자 S를 문자열로 바꿔주는 또다른 gcc의 pre-processor 기능입니다. 뭔가 .init.text를 문자열로 바꿔서 gcc에게 전달하는 것같습니다. gcc와 관련된걸 보니 커널의 동작보다는 빌드될때 뭔가를 하려는 코드네요.

결국 이렇게 하나하나 찾다보면 커널뿐 아니라 gcc의 컴파일러, 링커, 로더 등등 시스템 프로그래밍을 할 때 중요한 것들을 알아내게 됩니다. ```__section__```이 뭔지 ```__attribute__```가 뭔지 직접 찾아보세요.

major.h와 blkdev.h는 이름에서 보이듯이 장치의 장치 번호와 관련된 헤더와 블럭 장치와 관련된 헤더입니다. 그럼 다음 코드에는 장치 번호를 처리하고 블럭 장치를 처리하는 코드가 나오겠네요. 코드로 넘어가 보겠습니다.

###mybrd_init/mybrd_exit

mybrd_init() 함수는 드라이버가 시작될때 호출되는 함수입니다. 정확히 말하면 모든 드라이버는 시작 함수가 있어야합니다. 그리고 그 시작 함수는 module_init()로 등록합니다. mybrd.c의 끝에보면 module_init()함수에 mybrd_init 함수 포인터를 넘기는걸 알 수 있습니다. 커널이 커널 자체의 부팅이 어느정도 끝나면 드라이버들을 하나씩 실행시키겠지요. 그럴때 mybrd 드라이버가 실행될텐데, 커널에게 mybrd의 드라이버를 처음 시작될때 이 함수를 실행시켜달라고 저장해놓은 것입니다.

module_init()이 뭐냐면 간단히 말해서 커널 이미지의 특정 위치에 함수 포인터를 모아놓는 일을 합니다. 함수 포인터의 크기는 64비트니까 포인터의 배열이나 마찬가지지요. 그 특정 위치가 어디냐를 지정하는게 ```__attribute__``` 와 ```__section__``` 키워드입니다. 

어쨌든 드라이버를 구현하는 입장에서는 커널이 이미 드라이버가 실행되기 위해 필요한 프레임워크를 마련해놨으니, 거기에 따라서 module_init() 만 호출하면 끝입니다.

시작이 있으면 끝도 있겠지요. 드라이버가 끝날때의 함수는 module_exit()로 지정합니다. 우리는 mybrd_exit()함수를 드라이버의 끝으로 지정합니다.

###register_blkdev/unregister_blkdev

모든 장치는 결국 /dev 디렉토리에 있는 장치 파일로 존재합니다. 유닉스에서는 모든게 파일이니까요. 그럼 그 파일을 만들려면 커널에 우리 드라이버를 소개하고 파일을 만들어달라고 부탁해야합니다.

드라이버가 시작될 때 장치 파일을 만들기 위해 커널에 의뢰하는 함수가 register_blkdev 입니다. 장치 파일을 만드려면 major number와 minor number가 필요한데 이 major number를 커널로부터 받아오는 함수입니다.

커널 부팅과 드라이버 실행 확인

이제 커널을 빌드해서 부팅해보겠습니다. mybrd_init()함수에 메세지를 출력하도록 해놨으니 커널이 드라이버를 실행할 때 메세지들이 출력되야합니다.
```
/sys # dmesg | grep mybrd
[    0.323640] mybrd: mybrd major=253
[    0.323868] mybrd: 
[    0.323868] mybrd: module loaded
```
드라이버가 major number를 받아왔습니다. 장치가 제대로 등록됐는지 확인하려면 /proc/devices 파일을 확인하면 됩니다. 이 파일에는 커널에 등록된 장치들의 major number를 보여줍니다. register_blkdev() 함수에 장치의 이름을 my-ramdisk로 등록했으니 /proc/devices 파일에 my-ramdisk가 있는지 확인해보겠습니다.
```
# grep my-ramdisk /proc/devices 
253 my-ramdisk
```
그런데 아직 /dev에 새로운 파일이 없습니다. 장치 파일은 /dev에 있어야하는데 어디간걸까요. 지금까지는 장치 파일을 등록하기위한 번호를 받은 것이고 아직 장치 파일을 만든건 아닙니다. 다음 강좌에서 장치 파일을 만들어보겠습니다.

