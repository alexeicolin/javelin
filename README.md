This guide shows how to install Debian on a USB drive and boot Javelin directly
from it, using the stock kernel and module binaries from the original firmware.

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

Re-using the stock kernel and module binaries has the disadvantage that they
are not the bleeding edge versions compiled by you (sad), but have the
advantage that the device hardware is practically guaranteed to function as
completely and correctly as with stock firmware (particularly thermal
monitoring, fan speed, power-saving features, etc.). Also, the `t3sas` driver
for the Promise SATA controller is closed source. As BadIntensions
[implied](http://www.patriotmemory.com/forums/showthread.php?6980-Heavy-duty-Javelin-hacking&p=45967#post45967),
there is an open source alternative (not sure how the feature sets compare).

The following procedure does not introduce anything particularly new to the
findings by
[BadIntensions](http://www.patriotmemory.com/forums/showthread.php?6980-Heavy-duty-Javelin-hacking),
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
connector](http://www.patriotmemory.com/forums/showthread.php?6980-Heavy-duty-Javelin-hacking&p=42064#post42064)
[discovered by
BadIntensions](http://www.patriotmemory.com/forums/showthread.php?6980-Heavy-duty-Javelin-hacking&p=42064#post42064).

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

Connect the USB->UART cable into host and check that it is detected, i.e. that
a `/dev/ttyUSB0` device exists. If it does not, then check `dmesg` and `lsusb`.
Open a terminal client, like `screen` (or `minicom`):

    # screen /dev/ttyUSB0 115200

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

*Troubleshooting*: this warning is printed during init (probably related to
`udev`), but whatever is broken did not bite so far:

    [warn] CONFIG_SYSFS_DEPRECATED must not be selected ... (warning).
    [warn] Booting will continue in 30 seconds but many things will be broken ... (warning).

    udevd[1313]: failed to execute '/sbin/modprobe' '/sbin/modprobe -b platform:ppc4xx_gpio': No such file or directory

Reboot and catch U-Boot prompt with Ctrl-C:

    # reboot

This time we want to boot with MTD partitions being accessible:

    # setenv addtty ${addtty} ${mtdparts}
    # run nfsboot

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
by the kernel to mount a root file system from the USB drive (ignore the
warnings from `update-initramfs` about nonexistant `modules.{order,builtin}`):

    # apt-get install initramfs-tools
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

Userspace
---------

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
in `drivers/char/ppc4xx_gpio.c` ([in source tree released by
GPL](http://www.patriotmemory.com/software/Javelin/JV4800p_GPL_Source_v1.0.rar))
and through resources on GPIO on Linux.

Finish
------

Enjoy *your* hardware! One day they will sell it ready for use out-of-the-box...
