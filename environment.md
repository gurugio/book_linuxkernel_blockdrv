# prepare the environment

Let us start with build v4.4 kernel and booting it with qemu emulator and busybox.
We cannot avoid rebooting when we develope kernel driver because a tiny mis-behavior can kill the entire OS.
So we should use qemu emulator to make a virtual machine.
Even-if our driver do something wrong and kill the OS, we don't need to reboot our desktop.
We will just kill a qemu process and run it again.

Please refer to other documents if you don't know what qemu and busybox are.

## get kernel source

The first thing is, of course, kernel itself.
We don't need the latest version, so we will download v4.4.

The homepage of Linux kernel is here:
https://www.kernel.org/

As the guide in the homepage, we can download the kernel source with git or download zipped file.
For your convenience, you can download here:
https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.4.tar.gz

Source editor and tagging tool and other tools are up to you.

We will build the kernel soon.

## install qemu

Just install qemu tool. It depends on the distribution you use. For example, you can install qemu on Ubuntu with following commands.

```
# apt-get install qemu-system
# which qemu-system-x86_64 
/usr/bin/qemu-system-x86_64
```

I will describe only a few options that are used in this document.
Qemu has so many powerful features like other virtualization tools like Vmware and VirtualBox.
Please refer to qemu manual for the detail.

# make boot filesystem, initramfs, with busybox

Kernel can boot but that's all. We cannot do anything unless we have tools such like shell, dd and ls. It will take huge amount of time if we download source of each tool and build and install. So our dear hackers already made a toolset including essential tools for Linux OS. It is busybox. With the busybox, we can make a simple filesystem and the kernel can execute shell in the filesystem.

Please visit the homepage and download the latest source.

http://busybox.net

Build can be done with following commands.

```
# make defconfig
# make menuconfig
# make
```

The busybox is frequently used with Linux kernel. So it has the same build interface "make menuconfig".
You MUST set one option to make a bootable filesystem with the busybox.
You can find the option with following sequence.

Busybox Settings --> Build Options --> Build Busybox as a static binary

The option is look like following.

```
 [*] Build BusyBox as a static binary (no shared libs) 
```

If you press space or enter key, ``[]`` will be ``[*]``. The the options is selected. Press ESC and select YES.
We don't have time to install the shared libraries that busybox program will seek when it runs. So we build the busybox statically. After builing the bootable filesystem, you can see that there is no library.

Now we are ready. Run make now.

After all, ``_install`` directory will be created.

```
# make CONFIG_PREFIX=./_install install
# ls _install/
bin  linuxrc  sbin  usr
```
You can see several directories in there.
Let us check what are there in ``_install/bin`` directory.

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
All programs we want are there.

Now we have tools. Next we make a filesystem. Following commands make a filesystem in initramfs/x86-busybox directory.

```
$ mkdir -p ./initramfs/x86-busybox
$ cd ./initramfs/x86-busybox
$ mkdir -pv {bin,sbin,etc,proc,sys,usr/{bin,sbin}}
$ cp -av ./busybox/_install/* .
```
Above command create directories, bin, sbin, etc, proc, sys, usr/bin, usr/sbin, that are essential for Linux OS.
And it copies busybox tools into proper directories.
Finally x86-busybox directory is a root filesystem for kernel we will build for ourselves.

One more job to be done for bootable filesystem. It is creating init file. The Linux kernel run /init file after it finishes booting process. It there is no init program, the kernel would terminate itself. The init can be script or executable file. Let's make init like following.

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

Filesystem is ready. Next command creates a single image file with filesystem directory.
Run the following command at the root directory of our filesystem.
It generates a single file "initramfs-busybox-x86.cpio.gz".

```
$ find . -print0 \
    | cpio --null -ov --format=newc \
    | gzip -9 > ./initramfs-busybox-x86.cpio.gz
```
Please check man page of each tool for the detail.

## build kernel

Building Linux kernel is time-consuming job and sometime we should try many times.
But we can simply build kernel at the moment because we run the kernel on qemu, not real machine.
And also we don't need any kernel drivers. So we can reduce amount of building time.

Following commands are for kernel build.

```
$ make x86_64_defconfig
$ make kvmconfig
$ make -j8 bzImage
```

Let me describe briefly.
* make x86_64_defconfig: enable options in arch/x86/configs/x86_64_defconfig file
  * x86_64_defconfig file has common options for INTEL/AMD processor
* make kvmconfig: add several options for qemu
  * Recent kernel has changed the name into kernel/configs/kvm_guest.config
  * Or you could use kernel/configs/qemu-busybox-min.config
* make bzImage: build only kernel
  * "make" will build kernel and drivers. But building drivers takes much more time than kernel. And we don't need drivers.
  * "make bzImage" will build only kernel image
  * -j8: use 8 cores. You can set the number of cores in your system. More cores takes less time.

After build, please check arch/x86/boot/bzImage file exists. It is the kernel image file.

Following command is booting the kernel image and filesystem image.

```
$ qemu-system-x86_64 -smp 4 -kernel arch/x86/boot/bzImage \
-initrd ../initramfs-busybox-x86.cpio.gz \
-nographic -append "console=ttyS0 init=/init" -enable-kvm
```

Let me introduce the options.
* -kernel: location of kernel image
* -initrd: location of filesystem image
* -nographic -append "console=ttyS0 init=/init": print the kernel booting message on your terminal
  * Without this option, you cannot see anything.
  * If you used kernel/configs/qemu-busybox-min.config, you don't need to add it because the config file already has it.
* -enable-kvm: use kvm driver. It is not mandatory but it makes booting fast.

You can see the booting message of Linux kernel.
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

A shell is executed by init script. Test directories in the filesystem and check they are the filesystem we built with busybox.

We just use busybox and qemu to test Linux kernel, so I don't describe them in detail.
Above options are all we need to know at the moment.

The last command we need to know is terminating qemu.
"ctrl-a c" will run qemu monitor mode. "quit" command terminates qemu.
```
QEMU 2.3.0 monitor - type 'help' for more information
(qemu) quit
```

That's it.
Now we can build our kernel and driver and run it on virtual machine.
We can start making driver.
