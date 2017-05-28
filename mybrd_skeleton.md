# skeleton code of mybrd

Boring environment setup is done.
Let us start coding mybrd driver.
First we start with skeleton code to only help us to check whether the driver is loaded correctly.
And this chapter will describe how to add our driver code into kernel so that the driver is built and run with kernel.
We will start reading kernel source now.
Please keep "Linux device driver" and "Understanding the Linux kernel" books close to you.

## add mybrd.c to kernel

We want to make our block driver be part of Linux kernel. So we will add our driver source, mybrd.c file, into Linux kernel directory. Driver sources have to be added in drivers directory. You can find many sources and sub-directory with 'ls drivers' command. Our driver is for block device. We will place mybrd.c into drivers/block/ directory.


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

First let's make an just empty file, mybrd.c. And let's change Kconfig and Makefile like following to build mybrd.c with kernel image.

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
$ git diff drivers/block/Makefile
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

Kconfig file makes menu when you run 'make menuconfig' to set up kernel options. After fixing like above, run 'make menuconfig'. You can find the new option, "myRAM block device support" in "Device drivers ---> Block devices menu". You can activate the option with pressing spacebar key.
There is one thing to notice. You must activate the option with '*', not 'M'. 'M' means that the driver is not included in kernel image and saved in separated .ko file. '*' is for merging driver into kernel image. We will boot kernel image on busybox filesystem. So if you want to seperate driver, you must copy driver file into the busybox filesystem and build filesystem image. You don't want to build filesystem image whenever you change the driver source. So we merge the driver into kernel image to simplify driver test.

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

If you select 'Helo' menu, you can see help message of the driver like following.
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
If you select the option for mybrd driver, ``CONFIG_BLK_DEV_MYRAM`` value become y. Then make tool checks if obj-$(CONFIG_BLK_DEV_MYRAM) is obj-y and build mybrd.o file.

Yes, adding a new driver file is so easy. Adding menu and source file is all we should do.
Every option we select is stored int .config file. You can confirm that CONFIG_BLK_DEV_MYBRD is stored in .config.

```
$ grep MYRAM .config
CONFIG_BLK_DEV_MYRAM=y
```

Now adding driver source is done. Now let's try building.

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
Building drivers/block/mybrd.o is completed. And building bzImage is also completed..

## skeleton code

I saved source file in github. You can see full source with following link.

https://github.com/gurugio/mybrd/blob/ch01-skel/mybrd.c

I created branches for each chapter. A branch for this chapter is ch01-skel. If you checkout ch01-skel branch, you can check that mybrd.c has minimal code as desribed in this chapter.

### headers

mybrd.c includes 5 header files. "linux/init.h", "linux/module.h" and "linux/moduleparam.h" are common for all drivers. You need to know about compiler, linker and loader to understand those files. So I don't describe what they do in detail.

major.h and blkdev.h are, as shown by names, related to device number and block device. If you want to make a device driver and the device is block device, you must include those headers.

### mybrd_init/mybrd_exit

mybrd_init() fundtion is the entry pointe of the driver. It's called when the drier is loaded and starts to execute.
Every driver should have starting point that is registered by module_init() function.
In mybrd.c file, you can see module_init() is called with mybrd_init function pointer at the end of the file.

There are two ways to load a driver.
First is loading a driver dynamically after kernel booting is finished.
The loader finds a function that is registered by module_init() and calls it.
Second is loading a driver when kernel boots.
After kernel finishes booting itself, kernel calls all functions that is registered by module_init().
You can select how the driver is loaded when you selects kernel option.

The module_init() function selects entry points of each driver into a specific area.
That area is just like an array of function pointers.

The module_exit() is for registering the exit point of the driver.

### register_blkdev/unregister_blkdev

Every devices in Unix/Linux system exists as files in /dev directory.
Everything is file in Unix/Linux system.
So we should introduce our driver to kernel and request to create a file in /dev directory.
That is what register_blkdev() function does.

# kernel booting and driver loading

Now let's boot kernel. Our driver is loaded automatically because it is merged into kernel.
mybrd_init() function prints some message, so we should be able to find following message in kernel messages as following.

```
/sys # dmesg | grep mybrd
[    0.323640] mybrd: mybrd major=253
[    0.323868] mybrd: 
[    0.323868] mybrd: module loaded
```

We can confirm the driver get a major number from kernel.
And we can check /proc/devices if device is registered correctly.
The /proc/devices file shows the major number of each devices that are registered to kernel.
Our driver registers the name of device as my-ramdisk via register_blkdev() function.
So we can find my-ramdisk in the /proc/device file as following.

```
# grep my-ramdisk /proc/devices 
253 my-ramdisk
```

But you should note that there is no device file in /dev directory.
So far, we registered device and get the number from kernel but did not create device file.
We will do that in the next chapter.
