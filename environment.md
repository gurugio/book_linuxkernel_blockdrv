#개발환경
v4.4버전의 커널을 빌드해서 qemu로 부팅해보겠습니다. qemu, busybox 툴을 이용합니다.

##커널 소스 준비

커널에서 드라이버를 개발한다는건 사실 운영체제를 계속 재부팅할 일이 생길 수밖에 없다는 것입니다. 아무리 드라이버를 동적으로 로딩되게해도 잘못 만들면 운영체제 자체가 죽어버리게 되니까요. 그래서 우리는 qemu를 써서 가상 환경을 만들어서 커널을 실행하겠습니다. 불필요한 일을 줄이기위해 동적 로딩도 쓰지 않고, 그냥 커널 자체를 부팅하면서 mybrd 드라이버를 실행하겠습니다. 

가장 먼저 준비할건 당연히 커널입니다. 최신 버전은 이 강좌를 언제 읽으시냐에 따라 달라질것이니 제 맘대로 v4.4 를 기준으로 하겠습니다. 

리눅스 커널을 받으려면 당연히 리눅스 커널의 홈페이지에 접속해야겠지요.

https://www.kernel.org/

리눅스 커널을 다운받는 방법은 홈페이지에 나온대로 git으로 받는 방법과 압축 파일을 받는 방법이 있습니다. 저는 git을 선호합니다만 편한대로 사용하면 됩니다. 참고로 압축 파일의 링크를 남깁니다.

https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.4.tar.gz

커널 분석하는데 태깅을 안하면 안되겠지요. emacs + global을 쓰는 방법은 다른 강좌에 남겨놨으니 참고하세요. vi를 쓰시는 분들은 어서 emacs로 전환해서 광명찾으시구요 ;-)


##qemu 설치

qemu는 그냥 설치만 하면 됩니다. 배포판에 따라 다르니 저는 참고용으로 우분투용 설치법만 남겨놓겠습니다. 최종적으로 qemu-system-x86_64 실행파일만 설치되면 됩니다.

```
# apt-get install qemu-system
# which qemu-system-x86_64 
/usr/bin/qemu-system-x86_64
```


qemu를 더 다양하게 활용해보고 싶으신 분들은 다음 링크를 참고하세요.

https://gurugio.kldp.net/wiki/wiki.php/qemu_custom_kernel

busybox로 initramfs 준비

커널만 달랑 있으면 부팅은 되겠지만, 그 외에는 아무것도 할 수 없습니다. 쉘이 있어야 드라이버를 테스트해볼 수 있을거고, 테스트를 하려면 쉘뿐 아니라 dd같은 툴들도 필요합니다. 그런 최소한의 툴들을 모아놓은게 바로 busybox입니다. 이 busybox로 커널이 실행될 파일시스템을 만들겠습니다.

일단 busybox 홈페이지에서 가장 최신 소스를 받아서 압축을 풉니다.

http://busybox.net

빌드는 다음 순서로 진행됩니다.

```
# make defconf
# make menuconfig
# make
```

make menuconfig 명령을 실행하면 busybox의 옵션을 선택할 수 있습니다. 커널 빌드하고 매우 유사하지요. 반드시 선택해야하는 옵션이 하나 있습니다.

Busybox Settings --> Build Options --> Build Busybox as a static binary

라는 옵션을 찾아서 꼭 선택해야합니다.

```
 [*] Build BusyBox as a static binary (no shared libs) 
```

이렇게 [ ] 안에 [*] 표시가 되면 선택된 것입니다. ESC를 계속 누르면 저장할 거냐는 질문이 나오는데 Yes를 선택하면 됩니다. 우리가 만든 커널과 파일시스템에는 일반적인 프로그램이 실행되는데 필요한 라이브러리들이 없습니다. 그런 라이브러리들을 하나하나 설치하려면 커널 공부할 시간은 없어집니다. 그러니 파일시스템에 들어갈 모든 프로그램들을 라이브러리없이도 실행될 수 있도록 정적 빌드를 하는 것입니다.

그 다음은 make 를 실행합니다.

빌드가 끝나면 _install 이라는 디렉토리가 생깁니다.

```
# ls _install/
bin  linuxrc  sbin  usr
```

_install 안에는 이렇게 몇가지 디렉토리가 들어있습니다.
_install/bin에는 뭐가 있을까요.

```
# ls _install/bin -l
total 2504
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 ash -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 base64 -> busybox
-rwxr-xr-x 1 gohkim gohkim 2560432 Nov 12  2015 busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 cat -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 catv -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 chattr -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 chgrp -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 chmod -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 chown -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 conspy -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 cp -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 cpio -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 cttyhack -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 date -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 dd -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 df -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 dmesg -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 dnsdomainname -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 dumpkmap -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 echo -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 ed -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 egrep -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 false -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 fatattr -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 fdflush -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 fgrep -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 fsync -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 getopt -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 grep -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 gunzip -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 gzip -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 hostname -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 hush -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 ionice -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 iostat -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 ipcalc -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 kbd_mode -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 kill -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 linux32 -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 linux64 -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 ln -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 login -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 ls -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 lsattr -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 lzop -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 makemime -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 mkdir -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 mknod -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 mktemp -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 more -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 mount -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 mountpoint -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 mpstat -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 mt -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 mv -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 netstat -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 nice -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 pidof -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 ping -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 ping6 -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 pipe_progress -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 printenv -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 ps -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 pwd -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 reformime -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 rev -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 rm -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 rmdir -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 rpm -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 run-parts -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 scriptreplay -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 sed -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 setarch -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 setserial -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 sh -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 sleep -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 stat -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 stty -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 su -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 sync -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 tar -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 touch -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 true -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 umount -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 uname -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 usleep -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 vi -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 watch -> busybox
lrwxrwxrwx 1 gohkim gohkim       7 Nov 12  2015 zcat -> busybox
```

_install/bin 디렉토리에 우리가 사용할 다양한 툴들이 설치된걸 알 수 있습니다.

이제 busybox는 빌드됐으니 이걸로 커널 부팅에 사용할 파일 시스템을 만들 차례입니다. initramfs/x86-busybox 라는 디렉토리에 파일 시스템을 만들어 보겠습니다. 다른 디렉토리를 원하시면 원하시는대로 만드시면 됩니다.

```
$ mkdir -p ./initramfs/x86-busybox
$ cd ./initramfs/x86-busybox
$ mkdir -pv {bin,sbin,etc,proc,sys,usr/{bin,sbin}}
$ cp -av ./busybox/_install/* .
```

다시 설명하면 x86-busybox라는 디렉토리에 bin, sbin, etc, proc, sys, usr/bin, usr/sbin 등의 디렉토리를 만들고, 빌드된 busybox의 _install 디렉토리에 있는 bin, sbin 등의 디렉토리를 덮어씌우는 것입니다. 그럼 결국 x86-busybox라는 디렉토리에 우리가 만든 커널이 부팅하고 마운트할 파일시스템의 디렉토리가 됩니다.

마지막으로 해야할 일이 하나 있습니다. init 파일을 만드는 것입니다. 커널은 커널 자신의 부팅이 끝나면 마지막으로 파일시스템의 /init 파일을 읽어서 실행합니다. 만약 이 파일이 없다면 커널은 init을 찾을 수 없다는 에러 메세지를 내고 정지될 것입니다. 다음처럼 init 파일을 bash 스크립트로 만들어 줍니다.

```
$ vim init
#!/bin/sh
 
mount -t proc none /proc
mount -t sysfs none /sys

#Create device nodes
mknod /dev/null c 1 3
mknod /dev/tty c 5 0
mdev -s
 
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
 
exec /bin/sh

$ chmod +x init
```

마지막은 initramfs를 만드는 명령입니다.

```
$ find . -print0 \
    | cpio --null -ov --format=newc \
    | gzip -9 > ./initramfs-busybox-x86.cpio.gz
```

각 명령들의 자세한 설명은 man page를 참고하세요.


##커널 빌드해서 부팅하기

커널은 어떻게 빌드할까요. 간단합니다. 다만 사용하는 컴퓨터에 따라 시간이 좀 걸릴 수 있습니다. 다음 명령대로만 하면 됩니다.

```
$ make x86_64_defconfig
$ make kvmconfig
$ make -j8 bzImage
```

간단하게만 설명하겠습니다.

* make x86_64_defconfig: arch/x86/configs/x86_64_defconfig 파일을 커널 옵션으로 사용합니다.
* make kvmconfig: qemu를 사용할때 필요한 커널 옵션들을 추가합니다.
 * -j8에서 8은 사용할 컴퓨터의 코어 갯수로 바꾸면 됩니다. 그러면 멀티코어를 전부 사용해서 빌드를 합니다.
* make bzImage: make 만 실행하면 커널과 드라이버를 모두 빌드합니다. 커널 빌드보다 드라이버 빌드 시간이 훨씬 깁니다. 그러니 커널만 빌드하도록 make bzImage를 실행합니다.

잠시 기다리면 빌드가 완료됐다는 메세지가 나옵니다. arch/x86/boot/bzImage 파일이 생성되었는지 확인해봅시다. 이 파일이 바로 커널입니다.

다음 명령으로 커널과 파일시스템을 부팅할 수 있습니다. 스크립트 파일로 만들어놓으면 편하겠지요.
```
$ qemu-system-x86_64 -smp 4 -kernel arch/x86/boot/bzImage \
-initrd ../initramfs-busybox-x86.cpio.gz \
-nographic -append "console=ttyS0 init=/init" -enable-kvm
```
간단한 옵션 설명입니다.

* -kernel: 커널의 위치
* -initrd: busybox로 빌드한 파일시스템 파일의 위치
* -nographic -append "console=ttyS0 init=/init": qemu를 실행한 터미널에 커널 부팅 메세지를 곧바로 출력합니다. init옵션은 부팅될 커널에게 init 파일의 위치를 가르쳐줍니다.
* -enable-kvm: kvm이라는 가상 머신에 필요한 커널 드라이버를 활용합니다. 없어도 상관없지만 부팅이 약간 더 빨라지므로 편리합니다.

qemu로 가상의 컴퓨터를 만들고 그 컴퓨터에 우리가 빌드한 커널을 부팅시키는 것입니다. 파일시스템까지 같이 가상 컴퓨터에 실행시켜주니 너무나 편리합니다. 십수년전에 qemu가 성숙하지 못했을 때는 커널 개발이 얼마나 번거로웠을지 생각해보세요. qemu는 저같은 커널, 드라이버 개발자에게 정말 구글보다 더 소중한 존재입니다.

명령을 실행하자마자 아래와 같이 커널이 부팅하면서 출력하는 메세지들이 주르륵 나와야합니다.
```
[    0.000000] Linux version 4.4.0+ (gurugio@giohnote) (gcc version 5.2.1 20151010 (Ubuntu 5.2.1-22ubuntu2) ) #19 SMP Mon Oct 31 23:05:54 CET 2016
[    0.000000] Command line: console=ttyS0 init=/init
[    0.000000] x86/fpu: Legacy x87 FPU detected.
[    0.000000] x86/fpu: Using 'lazy' FPU context switches.
[    0.000000] e820: BIOS-provided physical RAM map:
...
...
[    1.609875] Write protecting the kernel read-only data: 6144k
[    1.613420] Freeing unused kernel memory: 568K (ffff880001372000 - ffff880001400000)
[    1.635361] Freeing unused kernel memory: 932K (ffff880001517000 - ffff880001600000)

Boot took 2.02 seconds

/bin/sh: can't access tty; job control turned off
/ # 
```
마지막에는 쉘이 실행됩니다. ls 도 실행해보고 각 디렉토리들을 확인해보시면 busybox로 만든 디렉토리인 것을 알 수 있습니다. 제가 ramfs이니 뭐니 설명해도 감이 안잡힐 수 있습니다. 하지만 이렇게 실행해보니 실제 하드디스크에 설치된 파일시스템이 아니라 ramfs (Ram filesystem)이 뭔지 감이오시지 않나요? 리눅스를 계속 쓰시다보면 언젠가는 또 다른 기회로 ramfs를 만나게 될 것입니다.

이 강좌에서는 리눅스 운영체제를 설명하려는게 아니므로 이런 툴등의 환경은 쓰기만 하겠습니다.

마지막으로 종료하기 위해서는 먼저 ctrl-a c 를 실행해서 qemu의 모니터를 실행합니다. 그리고 quit을 입력하면 qemu가 종료됩니다.
```
QEMU 2.3.0 monitor - type 'help' for more information
(qemu) quit
```

별다른 설명없이 그냥 따라하기 식으로 환경을 만들어봤습니다. 제가 빼먹은 것이 있을지도 모릅니다. 천천히 해보시고 안되는게 있으면 답글로 알려주세요. 수정하겠습니다.

참고로 다음 링크에 접속하시면 설명을 빼고 실행할 명령어들만 써있습니다. 

https://gurugio.kldp.net/wiki/wiki.php/qemu_kernel

다음 링크에는 qemu를 이용해서 우분투를 실행하고 qemu의 더 많은 옵션들을 활용해서 네트워크 접속을 하는 등 더 본격적인 개발 환경을 꾸미는 방법이 있습니다. qemu의 옵션뿐 아니라 네트워크 설정등을 하려면 꽤 많은 옵션들과 설정 툴들을 알아야하지만 한번만 환경을 만들어놓으면 계속 쓸 수 있으므로 쓸만합니다

https://gurugio.kldp.net/wiki/wiki.php/qemu_custom_kernel

