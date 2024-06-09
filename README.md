This guide shows how to install Debian on a USB drive and boot Javelin directly
from it, using either the stock kernel and module binaries from the original
firmware or a customized kernel built from the source released by Patriot.

Intro
-----

Debian is a good choice because it is one of the few distributions that
supports PowerPC at all and because there are no space constraints in this use
case (4Gb USB sticks are commonplace). There is little point to struggle with
tiny distros like OpenWRT or Optware when you can just have the convenience of
Debian repositories. The Embedian project has been terminated for the same
reason. The "small" distributions decrease the disk usage of the root file
system but have practically no bearing on runtime resource usage. A fresh
Debian system ran literally nothing except kernel threads, `init`, `getty` and
`bash`.

Re-using the stock kernel binary or compiling the exact version released by
Patriot has the disadvantage that it is not the bleeding edge version, but have
the advantage that the device hardware is practically guaranteed to function as
completely and correctly as with stock firmware (particularly thermal
monitoring, fan speed, power-saving features, etc.). The closed-source `t3sas`
driver for the Promise SATA controller would also work, however [it is not
recommended](#sata).

The following procedure does not introduce anything particularly new to the
findings by
[BadIntensions](http://web.archive.org/web/20130202084740/http://www.patriotmemory.com/forums/showthread.php?6980-Heavy-duty-Javelin-hacking),
[senomoto](http://www.patriotmemory.com/forums/showthread.php?8421-Bricked-Javelin),
and everyone else, except maybe the LED handling ;). The Debian installation is
standard procedure.

Root file system
----------------

All commands are on a Debian-based host system *as root user*, which can be
done using `sudo` before each command or by switching user to root by

    $ su -

Initialize the root file system (first phase of bootstrap):

    # apt-get install debootstrap
    # mkdir jav-rootfs
    # debootstrap --arch=powerpc --foreign --variant=minbase wheezy jav-rootfs/ http://ftp.us.debian.org/debian/

Create device nodes for serial console:
    
    # mknod jav-rootfs/dev/ttyS0 c 4 64
    # mknod jav-rootfs/dev/console c 5 1

Note: `/dev/console` is needed for first boot to `/bin/sh` and `/dev/ttyS0` is
needed for normal boot to `init`.

Create 'inittab' from a template:

    # cp jav-rootfs/usr/share/sysvinit/inittab jav-rootfs/etc/inittab

Edit `jav-rootfs/etc/inittab` to *comment* all `getty` lines that end with `tty#`
(since we have no screen), such as:

    1:2345:respawn:/sbin/getty 38400 tty1

and *uncomment* the line with `ttyS0` and change its baudrate from `9600` to
`115200` to get:

    T0:23:respawn:/sbin/getty -L ttyS0 115200 vt100

Set hostname (otherwise it defaults to host''s hostname):

    # echo javelin > jav-rootfs/etc/hostname

Expose the root file system via NFS:

    # apt-get install nfs-kernel-server
    # ln -s $PWD/jav-rootfs /export/jav-rootfs
    # echo '/export/jav-rootfs *(rw,nohide,insecure,no_subtree_check,no_root_squash,async)' >> /etc/exports
    # service nfs-kernel-server restart

To limit connections to LAN (if you are not safely behind a NAT), replace `*`
with an ip range like `192.168.0.1/24`.

UART connection
----------------

Theoretically, it is possible to develop a procedure that does not require a
UART connection by modifying the U-Boot environment settings stored in the NAND
memory manually and overwriting the old settings in NAND. However, even if that
succeeds, any unrelated issues that might come up during kernel boot or in
early userspace init would be near impossible to diagnose without a console.
Packaging an uber-user-friendly solution that would work out-of-the box without
requiring a UART purchase and setup might be feasible, but is not for me to do.

Get the [4-pin JST
connector](http://web.archive.org/web/20130202084740/http://www.patriotmemory.com/forums/showthread.php?6980-Heavy-duty-Javelin-hacking&p=42064#post42064)
[discovered by
BadIntensions](http://web.archive.org/web/20130202084740/http://www.patriotmemory.com/forums/showthread.php?6980-Heavy-duty-Javelin-hacking&p=42064#post42064).

Get a [USB->UART 3.3v adapter](https://www.sparkfun.com/products/12977) (most
commonly based on FTDI chip). To hook the JST connector to the UART-to-USB
cable I would get a [0.1'' header](https://www.sparkfun.com/products/116) and
solder the wires from the JST connector to the header. Only the TX, RX, and GND
(three wires) need to be connected. VCC should NOT be connected, it is 5v on
many USB->UART cables, but the VCC on the Javelin board is 3.3v. The pinout
from BadIntension''s
[picture](http://www.patriotmemory.com/forums/attachment.php?attachmentid=527&d=1309663838)
with pins numbered left-to-right:

    Javelin [pin] -> UART-to-USB cable
    --------------|-------------------
    TX       [1]  -> RX
    GND      [2]  -> GND
    RX       [3]  -> TX
    VCC      [4]  -> no connection
    
![javelin-s4-uart-pinout](https://user-images.githubusercontent.com/1269002/114777267-81243180-9d41-11eb-965e-0a9b346481d7.jpg)


Connect the USB->UART cable into host and check that it is detected, i.e. that
a `/dev/ttyUSB0` device exists. If it does not, then check `dmesg` and `lsusb`.
Open a terminal client, like `screen` (or `minicom`):

    # screen /dev/ttyUSB0 115200

Booting
-------

There are two options for serving the root file system (created in the above steps):

* NFS from any Linux host on a local LAN
* USB drive

The NFS option is described step-by-step below, because this is the approach I
took.

For the USB option (not tested), partition the drive (see [Bootable USB
drive](#bootable-usb-drive)), copy `jav-rootfs` to the system partition (same
section). Then, insert USB stick into Javelin.  In U-boot prompt, set
`bootargs` to contain `root=/dev/sda3` and `init=/bin/sh`. Then,`run `nandboot`
to boot the kernel stored in NAND (there is no kernel on the USB stick yet!),
but with the root file system from the USB stick.  Then, continue with
finishing bootstrap (see end of section [NFS boot: first
stage](#nfs-boot-first-stage)).

NFS boot: first stage
---------------------

Plug in the Javelin into a power socket. You should immediately see output from
UART.  Press Ctrl-C immediately to abort autoboot. Then, tell U-Boot to boot
with your rootfs via NFS as follows. Edit the following ip addresses according
to your local network (on mine, host system is at `192.168.0.3` and Javelin is
requested to be at `192.168.0.5`). The stock kernel was compiled without
`CONFIG_IPCONFIG*`, so have to assign IP statically, autoconfiguration using
DHCP won''t work.

    # setenv ipaddr 192.168.0.6
    # setenv gatewayip 192.168.0.1
    # setenv serverip 192.168.0.3
    # setenv rootpath /export/jav-rootfs

Optionally commit the above (harmless) environment changes to the NAND
to not have to re-enter them on subsequent couple of boots:

    # saveenv

Modify kernel cmd line arguments and boot:

    # setenv addtty ${addtty} init=/bin/sh
    # run nfsboot

Note: `nfsboot` overrwrite `bootargs`, so need to either modify `nfsboot` or
append to `addtty`, which is shorter.

*Troubleshooting*: `Warning: unable to open an initial console.` from `/bin/sh`
appears when there is no `/dev/console` device node. See above.

Remount root file system as read-write:

    # mount -o remount,rw /

Finish the bootstrap in the shell that the device booted into:

    # debootstrap/debootstrap --second-stage

Set root password and exit shell to reboot, and as it begins to boot catch
U-Boot with Ctrl-C to get to U-Boot prompt:
    
    # passwd
    # exit

NFS boot: second stage
----------------------

Tell U-Boot to boot into normal mode (`init`) but still with NFS root fs:

    # run nfsboot

*Troubleshooting*: if after kernel log you see output from init like `INIT:
version 2.88 booting`, but no login prompt appears, then something went wrong
with `/etc/inittab` or `mknod` for `/dev/ttyS0` in the steps above.

Login as root and add package repo for `apt`:

    # echo 'deb http://http.debian.net/debian wheezy main' >> /etc/apt/sources.list
    # apt-get update

In this boot the Internet connection should exist by virtue of the `ip` arg on
the kernel cmd line. *If* you have trouble resolving domain names, then try
    
    # echo 'nameserver 8.8.8.8' >> /etc/resolv.conf

Next boot will not have the `ip` kernel arg, so we need to configure
networking. Replace with your network addresses (NOTE: dhcp here did not work):

    # apt-get install net-tools ifupdown
    # cat >>/etc/network/interfaces <<EOF
    > auto eth0
    > iface eth0 inet static
    > address 192.168.0.6
    > netmask 255.255.255.0
    > gateway 192.168.0.1
    > EOF

Install `udev`, which monitors hardware drivers and creates device nodes
automatically, and is necessary to enable specifying the root fs by LABEL as we
will do shortly:

    # apt-get install udev

*Troubleshooting*: this warning is printed during init, but whatever is broken
did not bite so far (when [building custom kernel](#custom-kernel), we get rid
of this warning):

    [warn] CONFIG_SYSFS_DEPRECATED must not be selected ... (warning).
    [warn] Booting will continue in 30 seconds but many things will be broken ... (warning).

    udevd[1313]: failed to execute '/sbin/modprobe' '/sbin/modprobe -b platform:ppc4xx_gpio': No such file or directory

Reboot and catch U-Boot prompt with Ctrl-C:

    # reboot

This time we want to boot with MTD partitions being accessible:

    # setenv addtty ${addtty} ${mtdparts}
    # run nfsboot

First, we will get a working bootable setup using the stock kernel shipped with
the Javelin extracted from the NAND. A [later section](#custom-kernel) will
cover an option to customize the kernel configuration and build it from source.

Get the modules from the stock ramdisk (message from gunzip about trailing
garbage is because we copied the whole partition instead of the image, it is
ok):
    
    # dd if=/dev/mtd5ro of=stock-rootfs.img.gz bs=64 skip=1
    # gunzip stock-rootfs.img.gz
    # mkdir /mnt/stock-rootfs
    # mount -t ext2 -o loop,ro stock-rootfs.img /mnt/stock-rootfs
    # mkdir -p /lib/modules/$(uname -r)
    # cp /mnt/stock-rootfs/lib/modules/*.ko /lib/modules/$(uname -r)/

Create a new ramdisk for Debian with the `usb_storage` module, which is needed
by the kernel to mount a root file system from the USB drive, and with `udev`,
which is needed for refering to the root filesystem on the kernel command line
(ignore the warnings from `update-initramfs` about nonexistant
`modules.{order,builtin}`):

    # apt-get install initramfs-tools
    # dpkg-reconfigure udev
    # cat > /etc/initramfs-tools/hooks/usb-storage <<EOF
    > #!/bin/sh
    > . /usr/share/initramfs-tools/hook-functions
    > manual_add_modules usb_storage
    > EOF
    # chmod +x /etc/initramfs-tools/hooks/usb-storage
    # update-initramfs -c -k $(uname -r) -b $PWD

Package the ramdisk into a U-Boot image:

    # apt-get install u-boot-tools
    # mkimage -T ramdisk -A powerpc -O linux -n "initramfs-$(uname -r)" -C none -d initrd.img-$(uname -r) ramdisk.img

Extract the stock kernel and DTB images from NAND:

The bare minimum needed are the module binaries (which we already packaged into
the ramdisk above), since the kernel and the device tree blob (DTB) binaries
could be loaded as before from the NAND. However, to boot exclusively from the
USB stick without relying on the NAND at all (for example, to be safe against NAND
[suddenly
corrupting](http://www.patriotmemory.com/forums/showthread.php?8421-Bricked-Javelin)
on you) we will get the kernel and DTB images:

    # dd if=/dev/mtd1ro of=dtb.mtd.img
    # dd if=/dev/mtd4ro of=kernel.mtd.img

These are raw partition images, from which the packed uboot images would be
extracted (not strictly required, but done in next section on the host system).

Shutdown to release the NFS mounted file system:

    # sync
    # shutdown -h now

Bootable USB drive
------------------

*On the host*, partition a USB drive (wipes the data that is there!) using
`fdisk` with `X` replaced with the device letter, which can be found out by
running `fdisk -l` and looking at `dmesg`. DO NOT SPECIFY YOUR HARDDRIVE''s
DEVICE LETTER!

    # apt-get install gnu-fdisk
    # fdisk /dev/sdX

Example layout:

           Device    Boot      Start         End      Blocks   Id  System
    [boot 128M] /dev/sdX1               1          16      128488   83  Linux     
    [swap 1G]   /dev/sdX2              17         138      971932   82  Linux swap
    [sys  3G]   /dev/sdX3             139         487     2795310   83  Linux     

Create file systems with labels (we will later specify the root file system by
label):
    
    # mkfs.vfat -n boot /dev/sdX1
    # mkswap -L swap /dev/sdX2
    # mkfs.ext3 -L sys /dev/sdX3

Note: U-Boot can read from ext2, but much slower than from FAT (30sec vs 5sec).

Copy the root file system to the USB drive (Javelin should be off!):

    # mkdir /mnt/boot /mnt/rootfs
    # mount /dev/sdX1 /mnt/boot
    # mount /dev/sdX3 /mnt/rootfs
    # rsync -a jav-rootfs/ /mnt/rootfs/

Node: the base system turned out to be 252 MB. It will of course grow
significantly as we make it more usable in a later section.

Copy the images from rootfs into current directory for convenience:

    # cp jav-rootfs/root/*.img .

Extract the U-Boot-packed kernel image from partition image and check that it
looks good with `mkimage` (this extracting is not strictly required, but allows
to validate the image and reduces the number of bytes uboot has to load ;):

    # imgsize() { echo $((0x$(xxd -p -g 1 -s 12 -l 4 $1) + 64)); }
    # dd if=kernel.mtd.img of=kernel.img bs=1 count=$(imgsize kernel.mtd.img)
    # mkimage -l kernel.img

The DTB image is not a U-Boot image, so its length is not stored in the header,
but again it''s harmless to copy the whole partition, just rename it for
consistency:
    
    # cp dtb.mtd.img dtb.img

Copy all images for U-Boot to boot partition of the USB drive (mounted above):
    
    # cp kernel.img dtb.img ramdisk.img /mnt/boot/

Unmount the USB drive:

    # umount /mnt/{boot,rootfs}

USB Boot
--------

Plugin the USB drive into Javelin (only the right-hand side port works on my
unit for some reason), open the terminal with the UART connection (see UART
Connection above), plugin Javelin into power socket, and once U-Boot output
shows up, press Ctrl-C to stop autobooting.

Tell U-Boot to boot from the USB drive (just this time) and commit the newly
defined commands (harmless):

    # run enable_ext
    # setenv usbargs 'setenv bootargs root=LABEL=sys'
    # setenv usbboot 'run enable_ext;run usbargs mtdargs addtty;fatload usb 0:1 0x1200000 kernel.img;fatload usb 0:1 0x1b00000 ramdisk.img;fatload usb 0:1 0x1a00000 dtb.img;bootm 1200000 1b00000 1a00000'
    # saveenv

Try out booting from the USB drive:

    # run usbboot

Once you have verified that the boot succeeds all the way to the login prompt
and the shell, reboot, catch U-Boot with Ctrl-C again, and finally make
the default automatic boot be the USB drive boot:

    # setenv real_bootcmd 'run usbboot'
    # saveenv

If you ever want to boot the stock firmware, get to U-Boot prompt and execute
`run nandboot`. Execute `setenv real_bootcmd run nandboot` to make the choice
permanent.

Userspace: basic system setup
-----------------------------

Apt needs a dialog frontend, so before anything else install it:

    # apt-get install apt-utils dialog

Before installing anything else, it''s good to have a locale generated, install
this and when it asks, select (for example) `en_US.UTF-8` from the list and as
the default:

    # apt-get install locales
    # dpkg-reconfigure locales
    # echo LANG=en_US.UTF-8 > /etc/locale.conf

Get yourself an editor (for example `nano` or `vim`) and remember to add [full
list of repos](https://wiki.debian.org/SourcesList) to `/etc/apt/sources.list`
and `apt-get update`.

Install ssh and certificates (should not need the UART dongle starting now):
    
    # apt-get install ssh ca-certificates

Create a user and add it to sudo group and install sudo:

    # apt-get install sudo
    # adduser johndoe
    # usermod -a -G sudo johndoe

Add the usb drive partitions to `/etc/fstab`, the [following
example](https://raw.githubusercontent.com/alexeicolin/javelin/master/fstab) is
for the disk layout given in the preceding section:

    #<file system>          <dir>   <type>  <options>       <dump>  <pass>
    LABEL=sys               /       ext3    defaults        1       1
    LABEL=swap              swap    swap    defaults        0       0
    LABEL=boot              /boot   vfat    defaults        1       1

For some reason swap created earlier on the host system is not detected by the
Javelin, so re-create it on the Javelin (replace X with the letter of the USB
drive that the Javelin assigned to it -- THIS IS **NOT** ON THE HOST SYSTEM):

    # mkswap -L swap /dev/sdX2

Custom kernel
-------------

To customize the kernel configuration, you can build the kernel yourself.  The
version released by Patriot as part of GPL compliance (2.6.32.14) builds and
boots smoothly with the Device Tree Blob (DTB) extracted from NAND
[earlier](#nfs-boot-second-stage). Getting a more recent mainline kernel to
work is likely to be very very difficult. Also, it is easiest to compile
natively as opposed to getting a cross-compilation toolchain working. Build
time is on the order of hours.

SSH into your newly-setup Javelin system, and get the kernel source
[JV4800p_GPL_Source_v1.0.rar](http://www.patriotmemory.com/software/Javelin/JV4800p_GPL_Source_v1.0.rar) (also mirrored at link given at end of this guide), and install build
dependencies:

    # apt-get install unrar-free
    # unrar -x JV4800p_GPL_Source_v1.0.rar
    # cd GPL_Source
    # tar xf linux-2.6.32
    # cd linux-2.6.32
    #
    # apt-get build-dep linux
    # apt-get install libncurses5-dev

Configure the kernel for native build:

    # export CROSS_COMPILE=""
    # make menuconfig

In the menu, using the search function (`/`):

* set `CONFIG_LOCALVERSION` to a suffix appended to the kernel version, to
  distinguish your customized kernel
* enable `CONFIG_PPC_DISABLE_WERROR`: to make it build at all
* enable `CONFIG_USB_STORAGE`: to boot from root file system on USB drive
* enable `CONFIG_SATA_AHCI`: to access hard drives as block devices (see
  [section on SATA](#sata))
* disable `CONFIG_SYSFS_DEPRECATED_V2`: to get rid of warning and delay in
  boot issued by the Debian userspace
* whatever else you want to customize

Build the kernel and install the modules (only one by default:
`sci_wait_scan.ko`, which can't be built-in and recommented by comments in
`Kconfig` to be enabled):

	# make
	# sudo make modules_install
	# cd ..

Backup the currently booted kernel image:

	# cp /boot/kernel.img{,.bak}
	# cp /boot/ramdisk.img{,.bak}

Package the kernel into a uBoot image:

	# objcopy -O binary linux-2.6.32/arch/powerpc/boot/vmlinux.strip vmlinux.bin
	# gzip -v9 vmlinux.bin
	# mkimage -A powerpc -O linux -T kernel -C gzip -n "Linux-2.6.32.14-suffix" -d vmlinux.bin.gz /boot/kernel.img

Create the ramdisk, and package it into a uBoot image:

	# cd ..
	# sudo update-initramfs -c -k 2.6.32.14-suffix -b $PWD
	# mkimage -T ramdisk -A powerpc -O linux -n "initramfs-2.6.32.14-suffix" -C none -d initrd.img-2.6.32.14-suffix /boot/ramdisk.img

The Device Tree Blob (DTB) necessary for boot was extracted from NAND in an
[earlier step of this guide](#nfs-boot-second-stage), and should be in
`/boot/dtb.img`.

Mounting the stock NAND partitions
----------------------------------

For some reason `udev` creates the `/dev/mtd*` device nodes, but not the
`/dev/mtdblock*` nodes, which are needed for mounting the NAND partitions.
Fixed by writing an init script that creates the nodes via `mknod`:

    # apt-get install wget
    # wget -O /etc/init.d/mtdblock https://raw.githubusercontent.com/alexeicolin/mtdblock-init/master/mtdblock
    # chmod +x /etc/init.d/mtdblock
    # update-rc.d mtdblock defaults

To mount the partitions from NAND, add the following to `/etc/fstab`:

    # noauto is important since fstab is "executed" before the mtdblock
    # init script that creates the /dev/mtdblock* device nodes
    /dev/mtdblock6  /mnt/mtd/usr/img jffs2  noauto,user,ro  0       0
    /dev/mtdblock9  /mnt/mtd/app/img jffs2  noauto,user,ro  0       0

    # These can only be mounted after the above have been mounted
    /mnt/mtd/usr/img/usr_sqfs /mnt/mtd/usr/fs squashfs noauto,user,ro 0     0
    /mnt/mtd/app/img/app_sqfs /mnt/mtd/app/fs squashfs noauto,user,ro 0     0

Create the mount directories:

    # mkdir -p /mnt/mtd/{usr,app}/{img,fs}

Create a couple extra loop devices to be able to mount both `app` and `usr` at
the same time (and maybe something else too), `loop0` exists by default:

    # mknod /dev/loop1 b 7 1
    # mknod /dev/loop2 b 7 2

Then, to mount the file system, use two steps, for example for `usr` fs:
    
    # mount /mnt/mtd/usr/img
    # mount /mnt/mtd/usr/fs

Note: mounting as regular user works, but unmounting strangely complains that
"mount disagress with the fstab" unless you are root (incl, via sudo).

Sensors
-------

Temperature monitoring and fan control is done by the `w83l786ng` chip. The
standard `sensors` utility reads it nicely:

    # apt-get install lm-sensors
    # sensors

Fan control based on temperature is setup entirely in the kernel driver for
the monitoring chip, so it works without doing anything in userspace.

LEDs
----

In auto-boot mode U-Boot sets the system status LED to red. The stock firmware
userspace turned it to blue once booted. To replicate this, we can make use of
`ledctl` utility from the stock firmware and an init script.

    # mount /mnt/mtd/usr/img
    # mount /mnt/mtd/usr/fs
    # cp /mnt/mtd/usr/fs/bin/ledctl /usr/local/bin/
    # wget -O /etc/init.d/system-status-led https://raw.githubusercontent.com/alexeicolin/javelin/master/init.d/system-status-led
    # chmod +x /etc/init.d/system-status-led
    # update-rc.d system-status-led defaults

Edit the `system-status-led` if you want the LED completely off instead of blue.

Presumably, the `ledctl` binary creates a device node for `ppc4xx_gpio` char
device and does I/O on it. To avoid using a binary, it should be possible to
write your own equivalent script. It might help to look through the driver code
in `drivers/char/ppc4xx_gpio.c` in source tree released by GPL (see
`JV4800p_GPL_Source_v1.0.rar` linked above), and through resources on GPIO on
Linux.

SATA
----

Javelin contains the following SATA hardware:

  * [Promise
    PDC42819](http://firstweb.promise.com/product/product_detail_eng.asp?product_id=191)
    SATA/SAS RAID Controller hooked up to the PCI Express bus

  * Synopsys DesignWare Cores SATA Controller (DWC) built into the AMCC 460EX SoC
    (presumably this serves the eSATA port on the back)

Output from `lspci`:

    81:00.0 RAID bus controller: Promise Technology, Inc. PDC42819 [FastTrak TX2650/TX4650]

The driver for DWC is `sata_dwc` and is present among the module binaries
copied earlier, so it should work out of the box once you attach a drive to the
port on the back, as suggested by `dmesg | grep -i dwc` output:

    sata-dwc sata-dwc.0: id 0, controller version 1.82
    sata-dwc sata-dwc.0: DMA initialized
    sata-dwc sata-dwc.0: **** No neg speed (nothing attached?)

There are two drivers for Promise PDC42819:

  1. The Promise proprietary ("partially open source") [FX2650/4650
     driver](http://firstweb.promise.com/support/download/download2_eng.asp?productID=191&category=all&os=100):
     in `t3sas` module

     The [FX2650/4650 user
     manual](http://firstweb.promise.com/upload/Support/Manual/FastTrak_TX4650-2650_User_v1.0b.pdf)
     gives a comprehensive description of what it supports.

     This a ["fake RAID"](https://raid.wiki.kernel.org/index.php/Linux_Raid)
     controller. IIUC, the biggest problem is that once you create a RAID
     array using this driver, only this driver is capable to interpret the
     data from the drives (BAD!). Also, to use the drives at all you need
     the vendor's user space tools (Promise WebTRACK).

     With `t3sas` module loaded, the `i2arytool` in stock `usr` fs sees the
     drives (e.g. with `i2arytool extlist 0`). So, it probably is feasible
     to make use of this Promise driver in this Debian install. *However that
     is an unrecommended dead-end pursuit*. Maybe some kind of performance
     gain could be extracted, but one would need to hit a real performance
     bottleneck in one's use-case before even considering it.

  2. [AHCI generic SATA driver](https://raid.wiki.kernel.org/index.php/Linux_Raid)
     in `ahci` module

     This is a generic driver that supports multiple chipsets. It exposes
     the drives as raw block devices, with devices nodes created at
     `/dev/sd*` by `udev` as usual. This is the way to go. If RAID is
     desired, portable Linux software RAID should work fine.

     The `ahci` module binary is not included in the original firmware, but
     takes a few quick minutes to build as a kernel module. If you built
     your own kernel according to [instructions above](#custom-kernel), then
     `ahci` is already built into the kernel, and no steps are necessary here.

SSH into your Javelin, follow the first several steps of [the section on
compiling your own kernel](#custom-kernel) to download the kernel source code
and open the kernel configuration menu.

Set `CONFIG_SATA_AHCI=m` by finding it in Device Drivers -> Serial ATA -> AHCI.
Build the module (`crtsavres.o` prerequisite is not built automatically for
some reason, so build it manually), and install (the warning about symbol
versions info seems to be harmless):

    # make oldconfig prepare modules_prepare
    # make arch/powerpc/lib/crtsavres.o
    # make M=drivers/ata modules
    # make M=drivers/ata modules_install
    # depmod

Blacklist the Promise module to prevent it from claiming the drives:

    # echo 'blacklist t3sas' > /etc/modprobe.d/promise-sata-blacklist.conf

The `ahci` module will be loaded automatically at boot by virtue of
`depmod`+`udev` and `/dev/sd*` nodes should become available.

Finish
------

Enjoy *your* hardware! One day they will sell it ready for use out-of-the-box...

Files
-----

Just in case, the archive with the source code released under GPL, images
for the stock and my compilation of the kernel along with DTB and ramdisk,
and user manual for Promise FastTrak TX4650/2650 hardware are
available <a
href="https://drive.proton.me/urls/9YXA9KSGWG#Sz6k4jHFwQBO">here</a>.
